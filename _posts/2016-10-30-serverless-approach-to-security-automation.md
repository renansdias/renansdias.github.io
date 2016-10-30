---
layout: post
title: Serverless Approach to Security Automation
description: Learn how to use a serverless approach to protect your systems against an SSH-based attack
categories: [devops, security]
tags: [devops, security]
---

<p>I originally published this article on <a href="https://bsdmag.org/download/serverless-approach-security-automation-new-bsd-mag/" target="_blank">BSD Magazine</a>.</p>

# Introduction

<p style="text-align: justify">Everyone is talking about DevOps (Development and Operations). Some people say DevOps is a job of a single man, some other people say DevOps is rather a culture and is about the collaboration between the development team and the operations team. Truth is that, regardless of the conflicting ideas, everyone agrees that one of DevOps’ main goals is automation. Software release automation, right? Well, not quite. There are a lot of things that can be automated: software release, infrastructure provisioning, testing, benchmarking and security, to name a few. Now, did you notice the last thing on this list? Security.
It seems that people on the DevOps wave are only focusing on software release and not treating security as a first-class citizen. For that reason, a new term has been shed light on recently: DevSecOps (or DevOpsSec). [DevSecOps](www.devsecops.org) aims to turn Security into a first-class citizen in the DevOps wave by treating security as code and automating security procedures. For instance, suppose a company that deals with sensitive information at the infrastructure level. They probably have procedures in place to test their infrastructure to make sure there is as little security glitches as possible. However, depending on the size of the infrastructure, it might turn out quite expensive to run all of these tests manually, which means that these procedures need to be automated. They could automate running vulnerability scans and penetration tests when significant changes have been pushed to the infrastructure, for example. If you had never thought about automating security procedures, that is what this article is all about.</p>

# Problem

<p style="text-align: justify">A <strong>tainted server</strong> is a system where there has been any sort of unauthorised activity. This means that, if an unauthorised SSH session is open on a server, for instance, this server becomes tainted.
Consider now the following scenario: you have a cluster of servers in Amazon Web Services (AWS) running a web application, and these servers are part of the same Auto Scaling Group (an Auto Scaling Group is a service that scales your cluster up and down depending on the demand). Now, if you had a cluster of 1000 servers, how would you know, for instance, when an intruder manages to log in to one of the servers? You might have several monitoring tools in place that send a notification to your phone when someone manages to log in to the servers. That is awesome. But the fact is, that depending on when this happens, you may or may not be able to take action. What if it’s in middle of the night after a long day at work? Or what if you are in a highway driving during your holiday? That’s one of the reasons why we should bring automation to the InfoSec field.</p>

# Solution

<p style="text-align: justify">In a nutshell, when an unauthorised SSH session is open on a server, a script will kick in and will tag the server as tainted. As soon as the server is tagged, another procedure will decide whether the session seems to be legit or not. If the session is legit, then the server will be untagged. However, if the session is not legit, the server will be stopped.</p>

# Technology stack

The solution in this article will use the following technology components:

* Amazon Web Services
	* EC2
	* Lambda
	* Cloud Watch
	* AWS Command Line Interface
* Ubuntu 14.04 (instructions for CentOS 7 will also be given)
* Linux-PAM (Pluggable Authentication Modules)
* Python

<p style="text-align: justify">This is how all of the above will be used to build the solution: you will first launch an EC2 instance and configure PAM to execute a shell script when an SSH session is open. This shell script will then use the AWS Command Line Interface to add a tag with key <strong>tainted</strong> and value <strong>true</strong> to the instance. In one of the tests, this EC2 instance will also have another tag with key <strong>manageable</strong> and value <strong>true</strong>. The presence of the tag manageable indicates that if an ssh session is established, it will not be seen as an unauthorised access, but rather as an access that was made in order to carry some sort of maintenance (which the administrator is aware of). After configuring PAM, you will create an AWS Lambda function written in Python, which will be triggered by Cloud Watch. Cloud Watch will be responsible for checking when a new tag is added to an instance and then will trigger the Lambda function. The function, in turn, will get the instance ID and will check whether the tainted and manageable tags are present. If both are, the script will only remove the tainted tag and not shut down the instance. Else, if tainted is present and manageable is not, the instance will be stopped.</p>

# Step 1: Launch an EC2 instance and configure PAM

<p style="text-align: justify">The first step will be to launch an EC2 instance. If you’re already an EC2 instance launching expert, just launch an instance with the operating system of your choosing, attach an Identity and Access Management (IAM) role which allows the <strong>“ec2:CreateTags”</strong> action, add a tag to your instance with the key <strong>manageable</strong> and value <strong>true</strong>, and then skip to the part where PAM will be configured. If you have not ever set up an EC2 instance, keep reading.</p>

Go to the AWS console and click on **EC2**:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/aws-console.png" width="100%" height="100%" />

<br />

Then, click on **Launch Instance**:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/launch-instance.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">To launch an EC2 instance with Ubuntu 14.04, click on <strong>Ubuntu Server 14.04 LTS (HVM), SSD Volume Type</strong>:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/ubuntu-launch-instance.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">To launch an EC2 instance with CentOS 7, click on <strong>AWS Marketplace</strong> (left-hand side) and type in <strong>CentOS 7</strong>. The console will show at the top the CentOS 7 AMI - <strong>CentOS 7 (x86_64) - with Updates HVM</strong>:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/centos7-aws-marketplace.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">From now on, the images will only show the instructions for Ubuntu 14.04, but the equivalent instructions for CentOS 7 will be given when necessary.</p>

<p style="text-align: justify">After selecting an operating system, choose the size of your instance and click on <strong>Next: Configure Instance Details</strong> (since PAM does not require a powerful CPU or memory, the <strong>t2.nano</strong> instance size will do):</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/t2nano-selection.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">Feel free to change any configuration you’d like on this page. The most important thing, though, is the <strong>IAM role</strong>. The instance will need a role which allows the action <strong>CreateTags</strong>. If you happen to already have a role with such permission, select it using the drop-down list. Else, create a new role by clicking on <strong>Create new IAM role</strong> (the link will be open in a new tab or window):</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/create-new-role.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">This page will list all IAM roles you already created. But before creating a role, you will need to create a policy yourself because there is no AWS Managed Policy with the required permission (you could select the AmazonEC2ReadOnlyAccess policy, but we only need the CreateTags action - always bear in mind the <a href="https://en.wikipedia.org/wiki/Principle_of_least_privilege" target="_blank">Principle of Least Privilege</a>. To create a new policy, click on <strong>Policies</strong> on the left panel:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/select-policy-left-panel.png" width="100%" height="100%" />

<br />

Then, click on **Create Policy**:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/create-new-policy.png" width="100%" height="100%" />

<br />

To create your own policy, select the bottommost option:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/create-own-policy.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">Name the policy <strong>TaintedServerEC2Access</strong> and add a brief description (you could leave the description empty if you’d like):</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/name-and-description-policy.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">Now, in regards to the policy document, copy and paste the following JSON object:</p>

```
{
    "Version": "2012-10-17",
    "Statement": [{
        "Sid": "Stmt1475523748249",
        "Action": [
            "ec2:CreateTags"
        ],
        "Effect": "Allow",
        "Resource": "*"
    }]
}
```

<p style="text-align: justify">A Policy Document is a JSON object which states the methods of the AWS API that can or cannot be called by identities (users, groups, and roles). The policy document above, for instance, allows these identities to call the method CreateTags of the AWS API for any resource. If you wish to restrict this permission to a specific resource (e.g., an S3 bucket), you will need to get the resource’s Amazon Resource Name (ARN). Learn more about <a href="docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements.html" target="_blank">Policy Documents</a> and <a href="docs.aws.amazon.com/general/latest/gr/aws-arns-and-namespaces.html" target="_blank">Amazon Resource Names</a>.</p>

Click on **Validate Policy** to make sure there is no syntax error:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/policy-validation.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">After successfully validating your policy, click on <strong>Create Policy</strong>.</p>

<p style="text-align: justify">Now that you have your own policy, create a new role by clicking on <strong>Roles</strong> on the left panel and then on the <strong>Create New Role</strong> button at the top:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/role-create.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">Call it <strong>role-tainted-server</strong> and click on <strong>Next Step</strong> to move forward:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/role-name.png" width="100%" height="100%" />

<br />

Now select the **Amazon EC2** role type:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/amazon-ec2-role-type.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">And type TaintedServerEC2Access to filter the managed policy you’ve just created. Select it and click <strong>Next Step</strong>:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/select-custom-managed-policy.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">After creating the role, go back to the EC2 launching page and click on the <strong>refresh</strong> arrow so the console refreshes the list of roles and shows the role you’ve just created. Select your role and hit <strong>Next: Add Storage</strong>:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/select-new-role.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">Change the storage configuration if necessary, then hit <strong>Next: Tag Instances</strong>:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/config-volume.png" width="100%" height="100%" />

<br />

In the **Tag Instance** page, add two tags: 

```
Key				Value

Name				Tainted Server Test
manageable			true
```

<p style="text-align: justify">The <strong>Name</strong> key is just a suggestion because you will be able to identify this instance more easily in case you have loads of EC2 instances. But the <strong>manageable</strong> tag is mandatory (later in this article, you will understand why we need it). Add any further tags you find necessary and click on <strong>Next: Configure Security Group</strong>:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/tag-page.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">On the security group page, select any existing security group which allows inbound traffic on port 22 (SSH) from your IP address (it’s a really bad practice to open SSH to <strong>0.0.0.0/0</strong>). If you don’t have a security group with this rule, create a new security group. Call it <strong>seg-tainted-server</strong>, select the <strong>My IP</strong> option in the drop-down list under <strong>Source</strong> and hit <strong>Review and Launch</strong>:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/sg-page.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">Review all the instance launch details and click on <strong>Launch</strong>. If you already have SSH keys registered in your account, select one of them. Otherwise, create a new key pair, download it and click on <strong>Launch Instances</strong>:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/create-key.png" width="100%" height="100%" />

<br />

Grab either the public DNS name or the public IP address of the instance:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/server-address.png" width="100%" height="100%" />

<br />

And ssh into it:

```
$ ssh -i ~/.ssh/tainted-server.pem ubuntu@52.36.141.206

The authenticity of host '52.36.141.206 (52.36.141.206)' can't be established.
ECDSA key fingerprint is SHA256:Zh18mK+6LdMeipbvRJf+q5KhXZq2VaE3Fx+frOOKBbk.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '52.36.141.206' (ECDSA) to the list of known hosts.
Welcome to Ubuntu 14.04.4 LTS (GNU/Linux 3.13.0-92-generic x86_64)

 * Documentation:  https://help.ubuntu.com/

  System information as of Tue Oct  4 17:53:19 UTC 2016

  System load: 0.16              Memory usage: 10%   Processes:       82
  Usage of /:  10.0% of 7.74GB   Swap usage:   0%    Users logged in: 0

  Graph this data and manage this system at:
    https://landscape.canonical.com/

  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.

The programs included with the Ubuntu system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Ubuntu comes with ABSOLUTELY NO WARRANTY, to the extent permitted by
applicable law.

ubuntu@ip-172-31-35-204:~$
```

Note: if you received the message below:

```
Warning: Permanently added '52.36.141.206' (ECDSA) to the list of known hosts.
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@         WARNING: UNPROTECTED PRIVATE KEY FILE!          @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
Permissions 0644 for '/Users/renandias/.ssh/tainted-server.pem' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
Load key "/Users/renandias/.ssh/tainted-server.pem": bad permissions
Permission denied (publickey).
```

Just change the permission of your SSH key to **600**:

```
$ chmod 600 <path-to-your-ssh-key>
```

Once you log into the server, open the PAM configuration for the SSH daemon located at /etc/pam.d/sshd:

```
ubuntu@ip-172-31-35-204:~$ sudo vim /etc/pam.d/sshd
```

<p style="text-align: justify">PAM already comes installed in many Linux distributions. But if your system does not have PAM, do a quick research on how to install it on your distribution.
To execute a script upon successful login, you will use PAM’s pam_exec module. Add the following rule to the bottom of the file:</p>

```
# Executes a script upon successful login.
session optional pam_exec.so /usr/local/bin/tainted
```

Now, create the **/usr/local/bin/tainted** file with the following code: 

```
#!/bin/bash

if [[ "$PAM_TYPE" == "open_session" ]]; then

    instance_id=$(curl http://169.254.169.254/latest/meta-data/instance-id)

    aws ec2 create-tags \
        --resources $instance_id \
        --tags Key=tainted,Value=true \
        --region us-west-2
fi
```

<p style="text-align: justify">First, the script checks the PAM_TYPE. PAM passes data to the script via environment variables. With the PAM_TYPE, it’s possible to identify whether the user has just opened the session, or closed it, for instance. And that’s what the if statement below the shebang line is doing. We are not interested in tagging the instance upon session closure, but when the session has been opened. The curl command makes a request to the IPv4 Link-Local address 169.254.169.254 to get the instance’s ID. Now, let’s make a pause here. I can only imagine that you might be asking yourself what random IP address is this. According to the <a href="https://tools.ietf.org/html/rfc3927#section-2.1" target="_blank">RFC 3927</a>, an IPv4 Link-Local address is an IP address within the 169.254/16 range to communicate with other devices on the same physical (or logical) <a href="https://en.wikipedia.org/wiki/Link_layer" target="_blank">link</a>. Amazon hasn’t published any implementation details about how this is done, but my assumption is that there is some sort of device on the instance’s physical or logical link responsible for dealing with metadata requests. Back to the script, it then uses the AWS CLI to tag the instance with the tainted key (you will need to provide the region code to the AWS CLI, unless you run <i><strong>aws configure</strong></i> and manually set the region to be used as default). Note, though, that the AWS CLI does not come installed on the Ubuntu AMI (or the CentOS AMI). Save the script above and install the AWS CLI with the following commands:</p>

```
$ curl "https://s3.amazonaws.com/aws-cli/awscli-bundle.zip" -o "awscli-bundle.zip"

$ unzip awscli-bundle.zip # install unzip if necessary

$ sudo ./awscli-bundle/install -i /usr/local/aws -b /usr/local/bin/aws
```

Note: you will not need any AWS credential since the instance already have an IAM role attached to it.

Last but not least, change the script’s permission so the system can execute it:

```
$ sudo chmod +x /usr/local/bin/tainted
```

<p style="text-align: justify">Without any further ado, test if the instance will tag itself after we ssh into it. The test is pretty easy: just terminate your SSH session and log in again. This should trigger the tainted script and the instance should have a new tag with key <strong>tainted</strong> and value <strong>true</strong>:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/tainted-tag-created.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">Perfect, everything seems to be working! In case your instance does not tag itself, run the <strong>tainted</strong> script manually and check for any errors. After you fix the error(s), remove the tainted tag and confirm that the instance will tag itself when a user logs in with ssh.</p>

# Step 2: Create a Lambda function

<p style="text-align: justify">Now that the server is ready, you need to create a Lambda function which will execute as soon as an unauthorised ssh session is established. In this article, the function will stop the instance. However, if you wish to use this approach to protect your servers in production, you need to assess your environment first.</p>

<p style="text-align: justify">Suppose you have a service running on multiple machines in an Auto Scaling Group, and you decide that no one should ssh into these server to carry out any sort of maintenance. In this case, since you have multiple servers and the Auto Scaling Group is configured to spin new instances as instances go down, it is perfectly fine to stop/terminate upon unauthorised ssh session establishment (unless an ssh session is established in all servers of the Auto Scaling Group at the same time, which indicates a serious security glitch). What kind of action the Lambda function will carry out is totally up to you. The goal of this article is to show you how to use a serverless approach to secure your systems.</p>

<p style="text-align: justify">Even if the servers in a certain Auto Scaling Group are not to be managed, you still might want to carry out some maintenance given the circumstances. In this case, you could indicate to your Lambda function that everything is fine with the “tainted” instance and it should not be stopped/terminated. This could be accomplished by using a different tag. Remember the <strong><i>manageable</i></strong> tag you created for the instance? That’s right, you got it! If the <strong><i>manageable</i></strong> tag is present, this means that an authorised person is logged in to the instance (if an unauthorised person manage to tag the instance with the <strong><i>manageable</i></strong> tag and ssh into it, then that’s another security glitch you might need to look into).</p>

<p style="text-align: justify">To summarise the idea of what the Lambda function will do:
the function will get the instance ID of the tainted server.
If it finds the <strong><i>manageable</i></strong> tag, it will not stop the instance and will remove the <strong><i>tainted</i></strong> tag instead.
Else, if it does not find the <strong><i>manageable</i></strong> tag, the instance will be stopped.</p>

<p style="text-align: justify">The function in this article will be written in Python. But it could be easily ported to either Node.js or Java (which are currently the programming languages supported by AWS Lambda). Here is the function:</p>

```
import boto3
import json

ec2 = boto3.client('ec2')

def lambda_handler(event, context):
    
    # 1
    # Retrieve EC2 instance ID
    instanceId = event['detail']['requestParameters']['resourcesSet']['items'][0]['resourceId']
    
    # 2
    # Get all the tags of the instance
    response = ec2.describe_tags(
        Filters = [
            {
                'Name': 'resource-id',
                'Values': [
                    instanceId    
                ]    
            }
        ]
    )
    
    # 3
    isManageable = False
    
    # 4
    # Finding out if instance is manageable
    for tag in response['Tags']:
        if tag['Key'] == 'manageable' and tag['Value'] == 'true':
            isManageable = True

    # If the instance is not manageable, the instance will be shut down
    if isManageable is False:
        # 5
        try: 
            response = ec2.stop_instances(
                InstanceIds = [
                    instanceId   
                ]
            )
            
            print(response)
        except Exception as e:
            print(e)
            print("Exception thrown when shutting down instance: " + instanceId)
            raise e
    else:
        # 6
        # Removing tainted tag since the instance is manageable
        try:
            response = ec2.delete_tags(
                Resources = [
                    instanceId
                ],
                Tags = [
                    {
                        'Key': 'tainted',
                        'Value': 'true'
                    }
                ]
            )

            print(response)
        except Exception as e:
            print(e)
            print("Exception thrown when removing Tag tainted")
            raise(e)
```

Here’s the breakdown of what the function is doing:

* **#1** - Retrieves the instance ID of the instance which triggered the Lambda function (Note: the event dictionary is quite complex, so to get all the correct keys, print the dictionary first to understand how it is structured)
* **#2** - Describe the tags of the instance. The response is a dictionary with the key Tags
* **#3** - Declares a variable called **<i>isManageable</i>**. This variable will hold False in case the instance does not have the manageable tag and True otherwise.
* **#4** - Loops through the list of tags and set the variable **<i>isManageable</i>** to True in case the instance has the manageable tag
* **#5** - If there’s no manageable tag set, the instance is stopped.
* **#6** - If there is a manageable tag set, the instance is **not** stopped, and the tainted tag is removed.

Time to create this Lambda function! 

Go to the AWS Lambda dashboard. If you haven’t created a function yet, click on **Get Started Now**:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/get-started-lambda.png" width="100%" height="100%" />

<br />

On the Select Blueprint page, scroll down to the bottom and hit **Skip**:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/select-blueprint.png" width="100%" height="100%" />

<br />

Then, do not configure any trigger yet and hit **Next**:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/configure-trigger.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">Now the important bit. Call this function <strong>RecycleInstance</strong>, give it a brief description of what it does, and select <strong>Python 2.7</strong> (or any other version of Python available):</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/name-description-runtime.png" width="100%" height="100%" />

<br />

Next, copy and paste the function into the Lambda function code frame:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/lambda-function-code.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">Scrolling down to the <i>Lambda function handler and role</i>, you do not need to change the Handler field because it’s already set up with the correct function to be called (<i>lambda_handler</i> is the default name of the function called by the AWS Lambda service):</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/lambda_handler.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">If you defined a function called <strong>my_awesome_function</strong> instead, then the <i>Handler</i> would be called <strong>lambda_function.my_awesome_function</strong>. As to the role, select the <strong>Create a new role</strong> option using the drop-down list. A new tab will be open in the browser:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/role-for-lambda.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">You can leave the <i>Role Name</i> as is. The important thing is the <i>Policy Document</i>. The Policy Document will state all actions and <i>resources</i> your lambda function will have access to. Basically, your lambda function needs access to:</p>

* Create log group
* Create log stream
* Put log events
* Describe EC2 tags
* Stop instances
* Delete tags

The resulting Policy Document will be the following:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": "arn:aws:logs:*:*:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeTags",
                "ec2:DeleteTags",
                "ec2:StopInstances"
            ],
            "Resource": "*"
        }
    ]
}
```

<p style="text-align: justify">Note: this Policy Document states that the lambda function will be able to stop all instances. If you wish the function to only stop certain instances, get their ARN (Amazon Resource Name) and put them into an array for the <i>Resource</i> key.</p>

Double-check all the information and hit **Allow**:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/allow-lambda-function.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">After clicking on <strong>Allow</strong>, the tab will be closed. Go back to the Lambda page, and you will see your new role:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/select-existing-role.png" width="100%" height="100%" />

<br />

Note: I have experienced an issue when creating a new role. After creating the role, the lambda page would say that there was an error when creating the role. However, reloading the page was enough to make the role appear when selecting the <strong>Choose an existing role</strong> option. So, if you experience this issue, just reload the page.

Moving on. The Advanced Settings is useful if you want to dedicate more memory and time to your function. In this case, we don’t need more than 128 MB of memory and 3 seconds of timeout since the function is extremely simple. Hit <strong>Next</strong>:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/advanced-settings.png" width="100%" height="100%" />

<br />

Review all the information and click on **Create function**:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/create-lambda-function.png" width="100%" height="100%" />

<br />

Congratulations! Your (first?) lambda function is ready!

# Step 3: Create the Trigger

<p style="text-align: justify">The third and last step is to create the trigger. To do that, go to the CloudWatch dashboard. On the left-hand side, click on <strong>Rules</strong>:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/cloudwatch-dashboard.png" width="100%" height="100%" />

<br />

Now click on **Create rule**:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/create-rule.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">The event source will be the **AWS API call**. To use the <strong>AWS API call</strong> as an event, you will need to enable Cloud Trail, which is a service that stores all calls made to the AWS API. If you haven’t enabled Cloud Trail yet, do so before proceeding. Now, select <strong>EC2</strong> for the Service name. Finally, click on <i>Specific operation(s)</i> and select <strong>CreateTags</strong>:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/event-selector.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">On the right-hand side, you will select the target, which will be the lambda function aforementioned. Hit <strong>Configure details</strong> to proceed:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/configure-details.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">Choose a name to the rule (I named my rule <strong>RecycleInstanceRule</strong>), give it a brief description and hit <strong>Create rule</strong>:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/create-cloudwatch-rule.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">Phew, looks like everything is in place! Time to test the whole thing now. You will do 2 tests:</p>

* With the <i>manageable</i> tag
* Without the <i>manageable</i> tag

<p style="text-align: justify">Let’s see what happens in each scenario. First, with the manageable tag. Go to the EC2 dashboard, find the server you created a while ago and add the <strong>manageable</strong> tag if you haven’t done so:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/add-tag.png" width="100%" height="100%" />

<br />

Hit save and you’re good to go! Log into your instance.

```
$ ssh -i <your-ssh-key> ubuntu@<ip-address>
```

<p style="text-align: justify">As soon as you log in, go back to the AWS Console, refresh the list of instances, and look for the <strong>tainted</strong> tag:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/tainted-tag-added.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">Works like a charm. However, If you have just enabled CloudTrail, wait a few minutes before testing so CloudTrail has enough time to start tracking down API calls. Now, remember about the logic of the lambda function? Since there’s a tag with key <i>manageable</i>, the <i>tainted</i> key will be removed. If you refresh the console after a few more seconds, you will notice that the <i>tainted</i> tag is gone:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/tainted-tag-gone.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">To make sure there is no mistake, and to confirm your lambda function actually ran, go to the CloudWatch dashboard again. Click on <strong>Logs</strong>:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/cloudwatch-logs.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">You will notice that there is a log group called <i>/aws/lambda/RecycleInstance</i>. This is the log group that our Lambda function created. Click on it and you will see a log stream:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/log-stream.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">Whatever you told your function to print, it will be shown there. Now, the second test!</p>

Remove the **manageable** tag from your instance:

<br />

<img src="/assets/media/serverless-approach-to-security-automation/remove-manageable.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">And ssh into it again. After a few seconds, you will notice 2 things. The first one is that the instance was tagged with the **tainted** key as expected. The second is that the instance is being stopped:</p>

<br />

<img src="/assets/media/serverless-approach-to-security-automation/stopping-instance.png" width="100%" height="100%" />

<br />

<p style="text-align: justify">There you go, both tests have passed! You are now officially credentialed to go serverless in order to protect your infrastructure when servers are tainted! But don’t think that’s the end of it. There is so much more you can do. You could, for example, set up a software called <a href="https://www.snort.org/" target="_blank">Snort</a> to detect intrusion on your system and notify the personnel responsible for the system via Slack. Or use <a href="http://ossec.github.io/" target="_blank">OSSEC</a> to monitor when a file is altered and log to <a href="https://papertrailapp.com/" target="_blank">papertrail</a>. Or use <a href="https://cisofy.com/lynis/" target="_blank">Lynis</a> to identify potential vulnerabilities in your system. Play around with all the mentioned tools in this article a little bit more, and you will realise that automation can not only be easily used to deploy software, but to also protect your infrastructure and systems.</p>