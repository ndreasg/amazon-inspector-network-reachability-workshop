Module 4 - Integration and Remediation
======================================

In the previous module you identified issues with the network configuration and began remediating them. In this module you will leverage additional AWS services to deploy automatic remediations to known bad configurations. 

Integrating Inspector with other AWS Security Services
======================================================

In the previous module you configured your assessment to send findings to a Simple Notification Service topic. Now you are going to use that to trigger a Lambda function to remediate common findings. The CloudFormation template has deployed a Lambda function to block SSH access to misconfigured instances and included it in the CloudFormation template. Let's build the connection from SNS to Lambda and review the code.

1.	Go to the Amazon Simple Notification Service console

2.	Click on Topics on the left hand side

	![](./images/mod4-1-sns-topics.png)

3.	Click on the Topic named "InspectorAutomation"

	In order to have the SNS topic send data to Lambda, you need to create a subscription. You should see the subscriptions window on the bottom and it should be empty.

4.	Click Create Subscription

	![](./images/mod4-2-subscription.png)

5.	The Topic ARN should be filled out, but if it's not click on it and select the topic.

6.	Click on the Protocol drop down and select AWS Lambda

7.	You should now see another dropdown. Click on it and select the one Lambda function that's there.

	![](./images/mod4-3-create-subscription.png)

8.	Click Create Subscription

	We've now configured SNS to send any alerts it receives to our Lambda function. Our Lambda function is configured to only respond to specific findings in specific ways. If you're interested in reviewing the Lambda function you can go to the Lambda console. The relevant piece of code for this activity are shown below:

	![](./images/mod4-13-lambda-code.png)

	You can see here that the Lambda code adds a Network ACL line that blocks SSH from the internet to any instance that has SSH open to the internet.

	To trigger this you need to have Inspector submit a finding to SNS. Rather than wait 15 minutes for Inspector to finish an assessment though, you can simulate this action.

9.	While still in SNS click on Topics on the left hand side

10.	Click on the Topic named "InspectorAutomation"

11.	In the top right click on the button that says "Publish Message"

	![](./images/mod4-4-publish.png)

	Using the ARN you copied down in Step 3 of Module 3 you are going to publish a fake SNS message using the appropriate ARN to kick off the Lambda function.

	!!! info "What ARN?"
		If you don't have the ARN, you can go back to Inspector and copy the ARN from the Medium finding.

12.	Paste the ARN into the appropriate place in the following text: {replace the "**INSERT ARN HERE**" with your arn)

	``` 
	{"template":"arn:aws:inspector:us-east-1:123456789012:target/0-a12b3c4d/template/0-5e6f7g8h","run":"arn:aws:inspector:us-east-1:123456789012:target/0-a12b3c4d/template/0-5e6f7g8h/run/0-9i0j1k2l","time":"2019-04-09T00:00:01.401Z","finding":"INSERT ARN HERE","event":"FINDING_REPORTED","target":"arn:aws:inspector:us-east-1:123456789012:target/0-a12b3c4d"}
	```

13.	Paste the SNS message from above in the "Message body to send to the endpoint" text box

14.	Leave all the other fields empty

	![](./images/mod4-5-message.png)

15.	Click "Publish Message"

	!!! info "If you're bored"
		<p style="font-size:16px;">
		Alternatively if you have the time, you can re-run the Inspector report and watch once it's complete to see if the change was made.
		</p>

	To confirm that it worked you will check the Network ACL's.

16.	Click on Services on the top right and click on VPC

17.	On the left hand navigation click on Network ACLs

	![](./images/mod4-6-vpc-nacls.png)

18.	Since the Proof of Concept VPC is the one with the misconfiguration, click on the ACL associated with that VPC

	![](./images/mod4-7-nacls.png)

19.	On the bottom navigation, click on "Inbound Rules"

	![](./images/mod4-8-nacl-rules.png)

	Do you see a rule blocking SSH?

	But if SSH is completely blocked to the instance, how can legitimate administrators configure the machine? Well, they can modify the Security Group and then the NACL through their Change Process. But if they want to make sure the instance wasn't compromised there's another option.

20.	Click on Services on the top right and click on Systems Manager

21.	On the left hand navigation, click on Session Manager

	![](./images/mod4-9-systems-manager.png)

22.	On the right hand side click on Start Session

	![](./images/mod4-10-session-manager.png)

	Do you remember the instance ID with the misconfigured Security Group? If not, don't worry, it was the PoC Web Server for AZ2

23.	Click on the radio button next to the instance

24.	Click Start Session

	![](./images/mod4-11-session-instances.png)

25.	Type "ping 8.8.8.8" - Are you able to ping out to the world? Hit Cntl-C when you're ready to move on.

26. Type "whoami" - What user are you logged into the box as?

	![](./images/mod4-12-active-session.png)

	This looks just like an SSH session! Instead though, this is a proxy created by the AWS System Manager Agent installed on the AMI. With the AWS Systems Manager Session Manager feature you can create an SSH-like access to devices that don't have port 22 open at all. All that's necessary is to allow traffic to the Systems Manager Endpoint over port 443 and return traffic.

27.	When you're done, hit "Terminate" in the top right corner

28.	Confirm you want to terminate the session.

So now we've learned how you can use Inspector to kick off a Lambda function and automatically remediate potentially risks configurations. Additionally, you've seen how when you isolate instances from the world, you can still use AWS services to securely access them and perform troubleshooting or incidence response.

!!! Attention
	<p style="font-size:16px;">
	Now since there are some instances still open to the internet and potentially vulnerable, let's clean up what's been built.
	</p>

[Cleanup](05-cleanup.md)
