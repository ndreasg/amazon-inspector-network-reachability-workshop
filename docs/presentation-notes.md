
Optional: Presentation Notes
============================

While the environment is building let’s talk about what this represents. Remember, this scenario is comprised of a new company that has been in AWS for a few months and are moving their first few workloads in the cloud. The CloudFormation template is modeling their environment. The first workload to move was an external website with a database back end. Security insisted that access to those servers only be through a set of bastion hosts in a separate VPC. Recently, a developer has created a public proof of concept for a new highly available web service and connected it to the environment without going through all the proper change management. The developer did the right thing by putting two different servers in two different availability zones. But since the two AZs have different Security Groups, each with different port filtering setup, the workload may not be able to successfully failover correctly. The various IT stakeholders have been working off some assumptions but have not had the chance to test them yet.

Assumption 1.\> Instances in private subnets are not accessible from the internet

Assumption 2.\> Putting servers in different Availability zones provide failover and better reliability

Assumption 3.\> Nothing can route through the bastion VPC.

Assumption 4.\> Access to the servers is limited by least privilege

Assumption 5.\> The bastion hosts can access all environments

The IT team found the developer’s new environment not because of controls, but because the bill came back above budget. Now the security and operations team want to validate the right security is applied to the new environment as well as the existing one. They will use the Network Reachability report in Amazon Inspector to provide them validation of their network assumptions. This is a recent addition to Inspector, going public in November of 2018. 

Let’s look at their architecture on Slide 2.

Example Corp. has isolated workloads by VPC and has used VPC peering to meet some connectivity assumptions. You can see they tried to build public and private subnets, but we have to dig deeper to see what’s actually going on there. Their production web app on the left has a load balancer in front of it, which is great. But you can see the PoC environment is not as mature.

Before we dive too deep though, let’s go back and see what the built environment looks like.

[Running the Inspector Report](02-running-inspector.md)

<a name="mod2"></a>Module 2 - Architecture Review
==============================

Let’s go back to the architecture we looked at earlier. We know we need more information to better understand the data flows and connectivity. So let’s dig deeper to see if they designed everything right.

Let’s look at the route tables they’ve built. Slide 3 contains all of the information regarding routes in their environment.

Right away we see that something named “Public Route” applies to the Database subnets. What else can we glean from these? For example, what IP address range is their corporate office?

Okay, let’s look at the Security Groups on Slide 4.

So here we see some good things and some troubling things as well. The Load Balancer and Web Server look like they’ve been configured well. They both use Security Group references instead of CIDR blocks to allow access. But the database server Security group is in trouble. Rather than using the Web Server Security Group it looks like it’s set to everyone. The route table misconfiguration we saw earlier can mean this is a real problem.

What about the Bastion Server security group and CIDR blocks. Is there a better way to handle those? For example, is there a better way than IP addresses to reference groups of servers in AWS?

There are more Security Groups on Slide 5. The PoC environment specifically. What do you see wrong here? 

Let’s go back to the assumptions we talked about earlier. Slide 6 has a refresher. We can walk through each of these individually in the following slides.

1.	*Let’s talk about Assumption 1 the Security Team put in place. Assuming a properly architected three-tier web application, do the Web or DB Subnets need to be open to the internet for the website to work?*

	- No. The only group that needs to be public is the Security Group for the Load Balancers.

2.	*What does a properly secured solution look like from a data flow perspective? What subnets or instances need to be public?*

	- The Load Balancer in the public subnet needs to be public. But the webserver can have port 80 locked down to only be accessible from the Load Balancers. The Web server security group can enforce this. The Database subnet and servers can also be locked down to just the Web servers by using Security Group associations. To meet the security requirement, SSH and RDP can also be locked down to the Bastion servers.

3.	*Can the Bastion servers be referenced by Security Group, or just IP address range?*

	- Security groups can be shared across VPC Peers when the proper approvals are put in place.

4.	*What about the Bastion hosts? How do we use Ingress and Egress Security Group rules in Security Groups to control their access?*

	- Only requests coming from on-premises should be able to SSH or RDP into the Bastion servers. This is limited by Ingress rules. The Bastion hosts can then only talk to the servers it has been approved to talk to. This is controlled by Egress Rules.

5.	*What about Assumption 2? What should we look at to validate if this is true and does this diagram look right?*

	- We would want to look at Routing Tables and NACL’s for subnets, Security Groups for Instances, and NAT and Internet Gateways in VPC’s. In this case, the WebApp VPC looks both good and bad. The Security Groups are shared and the routing tables are the same between the subnets in different AZ’s. The single NAT Gateway is not ideal but does minimize the number of route tables necessary reducing the risk of a misconfiguration. The Bastion VPC looks setup correctly. The route tables and security groups look right. For the Proof of Concept VPC though there’s a lot wrong. There’s no need to split the security group and in doing so many of the rules are incorrect. This means that the Availability Zone failover won’t work right.

6.	*But if the WebApp VPC and Proof of Concept VPC are both connected to the Bastion VPC, can’t they talk to each other too? (Assumption 3)*

	- No. AWS VPC’s are not transitive in nature. Routing is basically a single hop only for VPC Peering. Now that can be changed with Transit VPC’s, 3rd party tools, or instances acting like routers, but in general VPC Peering does not have transitive properties. The Network Reachability report will show us this.

7.	*So we’ve talked about the routing to each subnet and server, but what about the Security Groups. For Assumption 4 what are the best ways to accomplish and validate least privilege? How can we use automated checks to validate this instead of doing it manually?* 

	- To accomplish least privilege each Security Group should map to other Security Groups. For example: The Web Servers should have an ingress rule only allowing traffic from the Load Balancers on Port 80, and the Database Servers should only have an ingress rule from the Web Servers from port 3306. Even across VPC Peers it’s possible to use Security Groups to only allow traffic from the Bastion Hosts to instances in the WebApp VPC if peering allows for it.

		But to validate this manually is time consuming and error prone, especially in large environments. So instead, using things like the Inspector Reachability Report we can validate certain assumptions. For example, making sure that only Web Servers can talk to Database servers. So even in non-ideal scenarios, where IP blocks are used to allow access, we can check to make sure instances aren’t accessible from the internet. Or if they are, that Security Groups block their access. This is Defense in Depth.

8.	*Given that sometimes changes happen without people’s knowledge, how can we confirm what’s accessible? For example, when the developer built the Proof of Concept space, they built a peer relationship and routed to the Bastion hosts. Can they connect?*

	- No. The developer did a good job of building connectivity to the Bastion VPC, but the Bastion VPC does not have the proper routing or Security Group rules to allow access to the PoCVPC. Both would need to be added to provide connectivity. The Network Reachability report shows this too.

So then let’s go to [Evaluating Findings](03-evaluate-findings.md) to look at how the report helps us validate these assumptions or findings.

<a name="faq"></a>Frequently Asked Questions
==========================

1.  **Does it replace a vulnerability scan?**

    1.  No. This will tell you what servers can talk to what, but it doesn’t give you details of everything installed on those servers. When used in conjunction with other Inspector reports you do get closer to a full vulnerability scan, but you may want to compare to existing tools in your environment.

2.  **Are you sending real traffic?**

    1.  No. The Network Reachability report uses the Zelkova framework inside of AWS to create mathematical proofs of communication channels. No actual packets are sent in the network. You can find out more at: <https://aws.amazon.com/security/provable-security/>

3.  **Is the agent open source?**

    1.  Not at this time.

4.  **What operating systems support the Amazon Inspector Agent?**

    1.  Certain Amazon provided AMI’s come with the Inspector agent already installed, and those will be notes in the title with “**with Amazon Inspector Agent**”. You can find the complete list of supported operating systems in the AWS documentation at: <https://docs.aws.amazon.com/inspector/latest/userguide/inspector_supported_os_regions.html>

5.  **Is the agent running as root?**

    1.  When installed following the AWS documentation, yes. More information can be found here: <https://docs.aws.amazon.com/inspector/latest/userguide/inspector_agents.html>.

6.  **Does the agent need a TCP/UDP port to be open?**

    1.  Yes. Outbound 443 to the Inspector and S3 (for logging) service endpoints is required. Please see the documentation at <https://docs.aws.amazon.com/inspector/latest/userguide/inspector_agents.html> for more details.
