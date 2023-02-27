---
layout: post
title: AWS Landing Zone - Root Account Usage Alerts
---

## Introduction ##

In the past couple of posts we have discussed AWS Landing Zones. Over the next few posts I'm going to discuss how can leverage various services across our AWS Organization to perform certain functions. One of the first things on my checklist is security posture and securing the root user credentials. In this post I will step you through how to leverage CloudTrail, CloudWatch Logs and SNS to alert if somebody logs into any of your AWS accounts leveraging Root User credentials. You only have to enable this alarm in one place and it is automatically enabled for all existing and future AWS accounts assuming they are contained within AWS Organization and  AWS Control Tower. For those of you that are new to AWS a brief description of the various services that are using to secure the Root account credentials to ensure that they are not been used incorrectly.

## Brief description of AWS Services that are Using ##


#### AWS CloudTrail ####

AWS CloudTrail enables auditing, security monitoring, and operational troubleshooting. CloudTrail records user activity and API calls across AWS services as events. CloudTrail records two types CloudTrail of events:

- Management events that capture control plane actions on resources, such as creating or deleting Amazon Simple Storage Service (S3) buckets.

- Data events that capture data plane actions within a resource, such as reading or writing an Amazon S3 object.

#### Amazon CloudWatch Logs ####

CloudWatch Logs enables you to centralise the logs from all of your systems, applications, and AWS services that you use, in a single, highly scalable service. You can then easily view them, search them for specific error codes or patterns, filter them based on specific fields, or archive them securely for future analysis. CloudWatch Logs enables you to see all of your logs, regardless of their source, as a single and consistent flow of events ordered by time, and you can query them and sort them based on other dimensions, group them by specific fields, create custom computations with a powerful query language, and visualize log data in dashboards.

#### Amazon SNS ####

Amazon Simple Notification Service (Amazon SNS) is a managed service that provides message delivery from publishers to subscribers (also known as producers and consumers). Publishers communicate asynchronously with subscribers by sending messages to a topic, which is a logical access point and communication channel. Clients can subscribe to the SNS topic and receive published messages using a supported endpoint type, such as Amazon Kinesis Data Firehose, Amazon SQS, Amazon Lambda, HTTP, email, mobile push notifications, and mobile text messages (SMS).

## Check to see if CloudTrail and CloudWatch are setup Correctly ##

AWS IAM best practices recommend using IAM users or roles instead of root credentials. Unauthorised usage of your root account can be very dangerous as it has full access to everything. In this section, I will show you how to use CloudTrail, CloudWatch Logs and Amazon Simple Notification Services (SNS) to detect when root access is been used, so you can take the necessary remediations if necessary.

If you followed the steps in the previous posts or have enabled Control Tower then CloudTrail will be making all your account's CloudTrail events available to CloudWatch Logs in the home region of your AWS Management Account.

If you go to CloudTrail Dashboard you should see a trail called *aws-controltower-BaselineCloudTrail* and it should have a status of *Logging*.

![_config.yml]({{ site.baseurl }}/images/blog/Secure-LZ-Root-Account-Alarm/BlogImage1.png)

Now if you go to CloudWatch Logs in your Control Tower Home Region you should see a log group *aws-controltower/CloudTrailLogs*.

![_config.yml]({{ site.baseurl }}/images/blog/Secure-LZ-Root-Account-Alarm/BlogImage2.png)

## Setup a Metric Filter in CloudWatch ##

- Open the CloudWatch console by typing CloudWatch in the top navigation search bar.  

- Click Log groups on the left navigation pane.  

- Select the checkbox next to the *aws-controltower/CloudTrailLogs* group. Click on Actions and select Create metric filter.  

![_config.yml]({{ site.baseurl }}/images/blog/Secure-LZ-Root-Account-Alarm/BlogImage3.png)

- Copy and paste into the Filter pattern box this text.  

**_{ $.userIdentity.type = "Root" && $.userIdentity.invokedBy NOT EXISTS && $.eventType != "AwsServiceEvent" }_**

![_config.yml]({{ site.baseurl }}/images/blog/Secure-LZ-Root-Account-Alarm/BlogImage4.png)

- You can ignore the test pattern and click on Next.  

- Add a name for the Filter in the Name Box.  

![_config.yml]({{ site.baseurl }}/images/blog/Secure-LZ-Root-Account-Alarm/BlogImage5.png)

- Add a name under *Metric Namespace*.  

- Add a name in the *Metric Name field*.  

- For the *Metric value field*, enter *1*.  

![_config.yml]({{ site.baseurl }}/images/blog/Secure-LZ-Root-Account-Alarm/BlogImage6.png)

- You will only have the ability to click Next, when you have filled in the above required fields.  

- Click on Create metric filter.

![_config.yml]({{ site.baseurl }}/images/blog/Secure-LZ-Root-Account-Alarm/BlogImage7.png)

## Create the Alarm ##

- Find the metric you just created under Metric Filters, put a check in the box beside the filter name and Click on Create alarm.

![_config.yml]({{ site.baseurl }}/images/blog/Secure-LZ-Root-Account-Alarm/BlogImage8.png)

- On the Metric screen accept the defaults.

![_config.yml]({{ site.baseurl }}/images/blog/Secure-LZ-Root-Account-Alarm/BlogImage9.png)

- For the conditions set it to *Greater/Equal* and the threshold value as *1*.  

![_config.yml]({{ site.baseurl }}/images/blog/Secure-LZ-Root-Account-Alarm/BlogImage10.png)

- Click next, under the Select an SNS topic, select Create a new topic.  

![_config.yml]({{ site.baseurl }}/images/blog/Secure-LZ-Root-Account-Alarm/BlogImage11.png)

- Give the topic a name, add the emails of the relevant people who need to be informed.  

- Click Create Topic and then click next.  

- Add your alarm name and click next.  

![_config.yml]({{ site.baseurl }}/images/blog/Secure-LZ-Root-Account-Alarm/BlogImage12.png)

- Finally click Create alarm.  

- Don't forget to go into your email to confirm subscription to the SNS topic.  

## Test the Alarm ##

- Log out of the console and log in as a root user from any of your AWS accounts.

- You should receive an email alert for root user usage. This alert will also be reflected in your CloudWatch alarms.

![_config.yml]({{ site.baseurl }}/images/blog/Secure-LZ-Root-Account-Alarm/BlogImage13.png)

## Conclusion ##

This is just a quick guide on one of the first steps you should take to secure your AWS Organization. As additional AWS accounts are added via Control Tower they are automatically enrolled in CloudTrail. This means the CloudTrail events for new accounts get sent to the CloudWatch Logs group created by Control Tower so this alarm will automatically cover all new accounts added via Control Tower account factory process. Don't just create the alarm, talk to your Security Team so that the recipients of the alert know what is about and what are the agreed actions when the alarm is triggered.
