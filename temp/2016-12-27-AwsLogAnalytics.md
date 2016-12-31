---
layout: post
title:  "Build log analytics solution using AWS"
categories: AWS, streaming
---

This is a AWS solution designed based on AWS services including:

* Amazon EC2
* Amazon Kinesis Firehose
* Amazon S3
* Amazon Kinesis Analytics
* Amazon Elasticsearch Service

Over all solution is depicted in the figure below.

![Solution overview]({{ base.url }}assets/img/aws_log_blog/AwsLogSolution.png)

This blog is inspired by [AWS blog source]. Please be warned that following this tutorial might cost you up top USD1.00.

* place holder for TOC
{:toc}

# Creating EC2 Instance and IAM role

We need to create an EC2 instance. 

* Go to EC2 Console and click on the "Launch Instance button"
* Select Amazon AMI 64-bit
* Select t2.micro instance
* Click on "Next: Configure Instance Details"
* In the IAM role click on create a new AIM role
* On the IAM management console select "Amazon EC2" 
* Select a name e.g. Firehose_Access and select the following permissions
** AmazonKinesisFirehoseFullAccess
* Click next
* Go back to EC2 console and assign newly created role to our instance (you might need to refresh to get the new role in the combobox)
* Click review and Launch
* Select an existing PEM key from the dropdown or create a new one.
* Select "Launch Instance"
* ssh to your instance
** ssh -i [your pem] ec2-user@[Ec2 public ip address]

## Creating a log file source

* Get the zip file from the https://github.com/kiritbasu/Fake-Apache-Log-Generator
* upload the zip file using scp to the EC2 instance
** $ scp -i [your pem file] location/to/Fake-Apache-Log-Generator-master.zip ec2-user@[ec2 ip address]:/home/ec2-user/
* ssh to ec2 insantance and unzip the file
** $ unzip Fake-Apache-Log-Generator-master.zip
** $ cd Fake-Apache-Log-Generator-master.zip
** $ sudo yum -y install gcc-c++ python27-devel atlas-sse3-devel lapack-devel
wget https://pypi.python.org/packages/source/v/virtualenv/virtualenv-1.11.2.tar.gz
tar xzf virtualenv-1.11.2.tar.gz 
python27 virtualenv-1.11.2/virtualenv.py sk-learn
. sk-learn/bin/activate
** $ sudo pip install -r requirements.txt
* And start creating an infinite log file
** $ python apache-fake-log-gen.py -n 0 -o LOG
* take a note of the log file that is generated e.g. access_log_20161227-084525

## Create Amazon Kinesis agent

You need to download and install amazon agent that can read log files and send contents to kinesis firehose.

* $ sudo yum install –y aws-kinesis-agent
* configure aws-kinesis-agent as follows
** sudo vi /etc/aws-kinesis/agent.json
** Content should look like the following

{
 "cloudwatch.endpoint": "monitoring.us-east-1.amazonaws.com",
 "cloudwatch.emitMetrics": true,
 "firehose.endpoint": "firehose.us-east-1.amazonaws.com",
 "flows": [
 	{
      "filePattern": "/home/ec2-user/Fake-Apache-Log-Generator-master/access_log*",
      "deliveryStream": "web-log-ingestion-stream"
      "dataProcessingOptions": [
 			{
 				"initialPostion": "START_OF_FILE",
 				"maxBufferAgeMillis":"2000",
 				"optionName": "LOGTOJSON",
 				"logFormat": "COMBINEDAPACHELOG"
 			}]
 	}
 ]
}

* Make sure you have given full permission to the /home/ec2-user/Fake-Apache-Log-Generator-master e.g. chmod 777 /home/ec2-user/Fake-Apache-Log-Generator-master for agent to be able to read the log file.
** Start the agent service 
** $ sudo service aws-kinesis-agent start

# Create Amazon Kinesis

We will create amazon Kinesis that will retreave the log file from the EC2 instance and pushes it to S3.

* Open the Amazon Kinesis console at
* Click Go to Firehose
* Click Create Delivery Stream
* On the Destination screen:
** Select destinaiton as S3
** name as "web-log-ingestion-stream"
** S3 bucket choose create new bucket, (needs to be globally unique name e.g. web-log-ingestion-bucket-12311)
** click next
** In the Configuration screen For IAM role, choose Create/Update Existing IAM Role.
** click next and "allow" "firehose_delivery_role" in the new screen
** click next and click "create delivery stream" after you reviewed the fire hose and S3 configuration


# References 

[AWS blog source]:https://aws.amazon.com/getting-started/projects/build-log-analytics-solution/
