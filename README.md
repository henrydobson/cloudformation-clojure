# Guosto Clojure Tech Test - Henry Dobson

<!-- MDTOC maxdepth:6 firsth1:2 numbering:1 flatten:0 bullets:0 updateOnSave:1 -->

1. [Inventory](#inventory)   
2. [Prerequisities](#prerequisities)   
3. [CloudFormation template](#cloudformation-template)   
&emsp;3.1. [Parameters](#parameters)   
&emsp;3.2. [Mappings](#mappings)   
&emsp;3.3. [Resources](#resources)   
&emsp;3.4. [Outputs](#outputs)   
4. [Ansible Clojure Role](#ansible-clojure-role)   
&emsp;4.1. [Plays](#plays)   

<!-- /MDTOC -->

## Inventory

Test Component  | URL
--------------- | ------------------------------------------------------------
CF Template     | [URL](https://github.com/henrydobson/cloudformation-clojure)
ansible-clojure | [URL](https://github.com/henrydobson/ansible-clojure)

## Prerequisities

- You must have an SSH Key in your AWS account.
- You must have a valid email.
- You must have reviewed the source materials and must be satisfied to use these in your own environment

## CloudFormation template

### Parameters

Parameter            | Value Constraints  | Example                | Notes
-------------------- | ------------------ | ---------------------- | --------------------------------------------------------------------------------------------------------------------
AWSRegion            | Multiple choice    | eu-west-1              |
EnvironmentType      | Multiple choice    | testing                | This parameter will define the ami & instance type
KeyPair              | Multiple choice    | aws-henrydobson        |
OperatorEmail        | Email format       | henrydobson@me.com     | This email will be used for ASG notification via SNS
PermittedSSHSourceIP | CidrIp             | 91.XXX.76.XXX/32       | Security groups use this IP to restrict traffic for web and SSH
S3ProvisioningBucket | CommaDelimitedList | henrydobson-XXX,Object | This is not actively used in this example but demonstrates how S3 could be used in UserData. See LaunchConfiguration
WorkingVPC           | Multiple choice    | vpc-XXXXXX             |

### Mappings

I have included basic mappings as a proof of concept but in this example they do little.

`RegionAndInstanceTypeToAMIID`: based on your region choice a different ami will be used. `EnviromentTypeToInstanceType`: based on your environment choice a different instance type will be used.

### Resources

Resource                     | Type                 | References                                                                     | Notes
---------------------------- | -------------------- | ------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------
NotificationTopic            | SNS Topic            | OperatorEmail                                                                  | Notifies topic with autoscaling errors and warnings
ClojureRole                  | IAM Role             | S3ProvisioningBucket                                                           | This is not actively used here however would allow GetObject requests from the specified S3 bucket in parameter S3ProvisioningBucket
ClojureInstanceProfile       | IAM InstanceProfile  | ClojureRole                                                                    | Associated to EC2 nodes in ASG from LaunchConfiguration so all autoscaling nodes inherit the role permissions
InstanceSecurityGroup        | Security Group       | PermittedSSHSourceIP, ELBSecurityGroup, WorkingVPC                             | Allows traffic from the ELBSecurityGroup and SSH from the PermittedSSHSourceIP
ELBSecurityGroup             | Security Group       | PermittedSSHSourceIP, WorkingVPC                                               | Allows traffic on 8080 from the PermittedSSHSourceIP
ClojureCollectorGroup        | ASG                  | AZs, ClojureCollectorLaunchConfig, NotificationTopic, ElasticLoadBalancer      | MultiAZ, ASG size 1-10, notications sent to SNS topic, ELB type HealthCheck
ClojureCollectorLaunchConfig | Launch Configuration | InstanceSecurityGroup, ClojureInstanceProfile, Mappings, Keypair OperatorEmail | UserData bootstraps nodes ready for ansible-pull. Commented actions show how one would provision with S3.
ScaleUpPolicy                | Auto Scaling Policy  | ClojureCollectorGroup                                                          | +1 simple scaling
CPUAlarmHigh                 | CloudWatch Alarm     | ScaleUpPolicy, ClojureCollectorGroup                                           | As requested, if CPUUtilization exceeds 70% for 2 periods of 5 mins, trigger alarm
ElasticLoadBalancer          | ELB                  | ELBSecurityGroup, AZs                                                          | MultiAZ, 8080-8080, HealthCheck points to test result URL

### Outputs

The output URL is constructed from resources in order to present the test result URL in it's full form.

## Ansible Clojure Role

I chose to use Ansible instead of my day to day management tool, Puppet as I am currently reviewing our internal toolchain and Ansible is one option going forward and you specifically wanted to see a pull model in action.

This was the first time I'd looked at Ansible in any depth and I believe I have made an extremely simple role with 3 major plays. I have not written these all from scratch and there are some secret variables (not real secrets) that you may wish to change before deploying. See `group_vars/tomcat-servers`.

The role has been written for use via ansible-pull and, in this context, is pulling from a public github repository. For production variants this could be modified to pull from a private repo or private s3 bucket.

### Plays

Play                     | Function                            | Result
------------------------ | ----------------------------------- | ----------------------------------
tomcat-server            | Provision tomcat7                   | Basic tomcat7 web server started
deploy-clojure-collector | use get_url module to pull WAR file | Adds WAR file to webapps directory
ansible-pull-setup       | Provision node for ansible pull     | Ansible pull runs on cronjob
