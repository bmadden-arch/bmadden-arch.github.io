---
layout: post
title: Amazon Connect SMS - Enabling Two Way SMS
---

## Introduction ##

Welcome to another year of posts covering various AWS products and services. The content in these posts is driven by what I'm working or investigating for use with my clients. In November 2023, AWS announced that Amazon Connect now supports two-way Short Messaging Service (SMS).
Amazon Connect SMS uses the same configuration and agent experience as calls &  chats, making it easy to deliver an enhanced omnichannel customer experiences. Customers can request assistance by simply sending a text message or by responding to a proactive message initiated by an outbound campaign.

Amazon Connect SMS is available in US East (N. Virginia), US West (Oregon), Asia Pacific (Tokyo), Asia Pacific (Singapore), Canada (Central), Europe (Frankfurt), and Europe (London).

In this blog post I will cover how to setup SMS messaging within Amazon Connect leveraging Amazon Pinpoint.


## Amazon Connect ##

Amazon Connect is a self-service, cloud-based contact-centre service that makes it easy for any business to deliver better customer service at lower cost. The self-service graphical interface in Amazon Connect makes it easy for non-technical users to design contact flows, manage agents, and track performance metrics.

**Features**
- Easy-to-use omnichannel cloud contact center
- Network of telephony providers from around the world
- High-quality 16kHz audio resistant to packet loss
- Intelligent automation, including interactive voice response and customer voice authentication
- Single, easy to use communication interface for agents
- Dynamic and personal contact flows
- Skills-based routing
- Real-time and historic metrics
- Natural language chatbots using Amazon Lex
- AI Powered Speech Analytics

**Benefits**
- Set up and make changes in just a few clicks
- Easily scale up or down to meet demand
- Save up to 80 percent compared to traditional solutions
- Ensure contacts are sent to the right agent
- Empower automated interactions and help agents improve customer service
- Save agents time and increase productivity
- Agents don't have to learn and work across multiple tools
- Make data-driven decisions to increase productivity and reduce wait times
- Make changes in minutes, not months

## Amazon Pinpoint ##

Amazon Pinpoint SMS is a cloud-based application-to-person (A2P) text messaging offering that provides a cost-effective, flexible, and scalable way for businesses to connect with their customers through SMS.

With Amazon Pinpoint SMS, senders can:

- Manage global scale and high throughput: Send SMS messages to customers in over 240 countries with a variety of supported sender types.
- Create resiliency: Use phone pools to seamlessly manage all your sending resources in one place and send messages with failover capabilities.
- Self-service registration: Register and request various sender types directly in the Pinpoint SMS console including US 10DLC and toll-free numbers.
- Integrate SMS messaging into your apps and workflows: Stream SMS events to CloudWatch for monitoring, Kinesis for analytics, and SNS to fan out SMS event data to any system you choose.
- Send and receive messages with two-way SMS: With Amazon Pinpoint SMS, you can enable two-way messaging allowing your customers to send messages with an automatically triggered response.  
- Test SMS sending: Use the SMS simulator in the Amazon Pinpoint SMS console to get setup quickly and start testing sending in a matter of minutes.

## Enabling SMS in Amazon Connect ##

You will use Amazon Pinpoint SMS to obtain an SMS-enabled phone number and enable two-way SMS on the number. Finally we will import it into Amazon Connect.

* Open the Amazon Pinpoint console at [https://console.aws.amazon.com/sms-voice/] in the Frankfurt (EU-Central-1) region. You must claim Pinpoint number in the same region as your Amazon Connect instance.

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage1.png)

* In the navigation pane, under Configurations, choose Phone numbers and then Request originator.

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage2.png)

* On the Select country page you must choose the Message destination country from the drop down that messages will be sent to. Choose Next. We going to select Ireland for this exercise

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage3.png)

* On the Messaging use case section, enter the following and then click Next.

- Under Number capabilities choose SMS.

- Under Estimated monthly SMS message volume per month – Select < 5,000

- For Company headquarters, select Local

- For Two-way messaging select Yes

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage4.png)

* Under Select originator type, choose Long Code and click Next. To request a long code you need to open a case with AWS Support. I'm not going to cover the opening of the Support Case. It took 10 days for AWS to receive an Irish mobile number to assign to me in Pinpoint.

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage5.png)

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage6.png)

* Picking up from when AWS provided me with a Long Code.

## Setting Amazon Connect as Two-way SMS Destination ##

Once AWS have procured a number for you then it will appear in the Amazon Pinpoint SMS console, for you to enable two-way SMS.

* You can either open the SMS console at [https://eu-central-1.console.aws.amazon.com/sms-voice/home?region=eu-central-1#/overview] or go to Amazon Pinpoint.Select Pinpoint SMS and then press Manage SMS.

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage7.png)

* In the navigation pane, under Configurations, choose Phone numbers.

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage8.png)

* On the Phone numbers page, click on the telephone number you wish to enable Two-way SMS for.

* Now on the Two-way SMS table choose the Edit settings button.

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage9.png)

* For the Destination type choose Amazon Connect.

* For Amazon Connect in the Two-way channel role choose existing IAM roles

* Choose Save changes

* The Import Phone Number to Amazon Connect window opens. You need to select which of your Amazon Connect instances to send the incoming messages to.

*  Choose Import Phone number.

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImag10.png)

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage11.png)

* After you imported the number in the steps above, the number will be visible in Amazon Connect under Channels and Phone numbers.

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage12.png)

## Amazon Connect SMS Call Flow ##

* In my Amazon Connect I have created a new simple Call flow called SMS.

* You need to add a Check contact attributes block to the flow.

- Namespace value is Segment attributes
- Key has a value of subscriptionType
- Condition is Equals
- Value must be connect:SMS

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage13.png)

* Also included in my flow is Set Working Queue and Transfer to flow. Please see my earlier blog posts if you need assistance on creating a call flow.

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage14.png)

* Final step is to associate the phone number with the call flow. In Amazon Connect go to Channels, Phone numbers and associate the number to the call flow created in Step 1.

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage15.png)

## Testing Two-Way SMS ##

We are now ready to test two-way SMS. For this we will need the Amazon Connect Agent Workspace or Contact Control Panel (CCP) and a mobile phone.

* In the Agent Workspace set yourself to available

* Using a mobile device, send a SMS to the phone number created in Pinpoint.

* The Agent Workspace will ring with a SMS chat.

![_config.yml]({{ site.baseurl }}/images/blog/Amazon-Connect-SMS/BlogImage16.png)

* The agent and customer communicate as normal over SMS.

**N.B**
Please note to release the phone number claimed if you no longer need it as there is a monthly charge of **$10**.

## Conclusion ##

With Amazon Connect SMS you can now use all the same automation, routing, configuration, analytics and agent experience that you have in place for voice and web-chat ensuring a seamless omnichannel customer experience. Customers can now seek assistance by sending a SMS to one of your public numbers which can be routed to a chatbot or live agent. They can also respond to a SMS sent via an outbound campaign. Stay tuned for future blog posts.
