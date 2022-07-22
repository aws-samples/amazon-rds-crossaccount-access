
> ## Access Amazon RDS across VPCs using AWS PrivateLink and Network Load Balancer

In this post I will provides a solution to access [Amazon Relational Database Service](https://aws.amazon.com/rds/ "https://aws.amazon.com/rds/") (Amazon RDS) across AWS accounts and VPCs, without using [VPC peering](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-peering.html) with [Amazon Virtual Private Cloud](https://aws.amazon.com/vpc/) (Amazon VPC) or [AWS Transit Gateway](https://aws.amazon.com/transit-gateway/).

We use [AWS PrivateLink](https://aws.amazon.com/privatelink/) and [Network Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html "https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html") to redirect database traffic to Amazon RDS, [Amazon Aurora](https://aws.amazon.com/rds/aurora/ "https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_AuroraOverview.html"), or [Amazon RDS Proxy](https://aws.amazon.com/rds/proxy/ "https://aws.amazon.com/rds/proxy/"). We also use [AWS Lambda](https://aws.amazon.com/lambda/) to automate Network Load Balancer target group IP address synchronization in the event of RDS failover.

AWS PrivateLink provides private connectivity between VPCs, AWS services, and your on-premises networks, without exposing your traffic to the public internet. AWS PrivateLink makes it simple to connect services across different accounts and VPCs to significantly simplify your network architecture.

Network Load Balancer functions at the fourth layer of the Open Systems Interconnection (OSI) model. It can handle millions of requests per second. After the load balancer receives a connection request, it selects a target from the target group for the default rule. It attempts to relay a TCP connection to the selected target on the port specified in the listener configuration.

_Failover_ is a high availability feature that replaces a database instance with another one when the original instance becomes unavailable. A failover might happen because of a problem with a database instance. It might also be part of normal maintenance procedures, such as during a database upgrade. Failover applies to RDS DB instances in a Multi-AZ configuration, and Aurora DB clusters with one or more reader instances in addition to the writer instance. [Target groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-target-groups.html) for Network Load Balancer use IPv4 and IPv6 target IP addresses. Amazon RDS and Aurora change their IP address in the event of database failover. Therefore, database clients connecting to Amazon RDS using a Network Load Balancer endpoint receive an error if the RDS database has failed over and the new database instance IP address hasn’t been updated to the Network Load Balancer target group.
<![endif]-->

## Why Network Load Balancer for Amazon RDS?

Let’s say an organization's cloud infrastructure consists of multiple AWS accounts and VPCs. In this scenario, Amazon RDS or Aurora run under private subnets in one of the VPCs, and applications run in a different VPC. To access Amazon RDS across VPCs, you can either use VPC peering or  AWS Transit Gateway, or you can create [VPC endpoint services](https://docs.aws.amazon.com/vpc/latest/privatelink/endpoint-service.html) using AWS PrivateLink. Unlike VPC peering which logically merges two VPCs and exposes all the services across the VPCs, the VPC endpoint service option creates an AWS PrivateLink route to access only the RDS service to other Amazon RDS-hosted VPCs. However, the VPC endpoint service talks to only the Network Load Balancer service in the RDS VPC.

## Challenges

Amazon provides [high availability](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html) and automatic failover support for Aurora clusters with Regions and [Multi-AZ deployments](https://aws.amazon.com/rds/features/multi-az/) for RDS instances. Amazon RDS uses several technologies to provide failover support. The failover mechanism automatically changes the [Domain Name System (DNS)](https://aws.amazon.com/route53/what-is-dns/) record in [Amazon Route 53](https://aws.amazon.com/route53/) for the new appropriate (writer or reader) instance. Therefore, client connections are redirected to a new primary instance immediately (or depending on the DNS TTL and [TCP keepalive parameters settings](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.BestPractices.html)) after the failover. When you use RDS Proxy for connection pooling, RDS Proxy uses its internal mechanism to detect the failover and redirect the client request to the new writer instance. However, RDS Proxy works within the same AWS account and Region where the RDS service is hosted.

If you’re using Network Load Balancer to redirect client connections to the Amazon RDS or Aurora database, you must update the IP address of the new primary instance to the Network Load Balancer target group manually when database failover occurs.

## Solution overview

The Amazon RDS connection route workflow is as follows:

A.	Database users or applications connect to Amazon RDS using VPC endpoints.
B. The endpoints establish the user connection to VPC endpoint services (AWS PrivateLink) in other VPCs.
C. The VPC endpoint services establish the connection request to the Network Load Balancer.
D. The Network Load Balancer forwards the connection to the RDS primary instance.

The workflow to update the new IP to the Network Load Balancer target group is as follows:

1. The Amazon RDS failover process updates the new primary instance IP to Route 53.
2. The failover process generates a failover event, which triggers an [Amazon Simple Notification Service](https://aws.amazon.com/sns/) (Amazon SNS) topic.
3. The SNS topic has a subscription that triggers an [AWS Lambda](https://aws.amazon.com/lambda/) function.
4. The Lambda function gets the current IP from Route 53, and gets the current registered IP from Network Load Balancer.
5. The function checks if both of the IPs are the same. If not, it registers the new IP received from Route 53 to Network Load   Balancer, and deregisters the old IP from Network Load Balancer.
6. Now all the new user connections are redirected to the new primary instance.

You use the following AWS services and features to implement this solution:

[AWS CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html) – AWS CloudFormation helps you model and set up your AWS resources, provision them quickly and consistently, and manage them throughout their lifecycle. You use a template to describe your resources and their dependencies, and launch and configure them together as a stack, instead of managing resources individually.
[Amazon CloudWatch Events](http://aws.amazon.com/cloudwatch) – CloudWatch Events delivers a near-real-time stream of system events that describe changes in AWS resources. For more information, refer to [Creating a CloudWatch Events Rule That Triggers on a Schedule](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/Create-CloudWatch-Events-Scheduled-Rule.html).
[AWS Lambda](https://aws.amazon.com/lambda/) – Lambda is a compute service that supports running code without provisioning or managing servers.
[Amazon RDS](https://aws.amazon.com/rds/) – Amazon RDS is a managed relational database service for MySQL, PostgreSQL, MariaDB, Oracle BYOL, or SQL Server.
[Amazon RDS Proxy](https://aws.amazon.com/rds/proxy/) –RDS Proxy is a fully managed, highly available database proxy for Amazon RDS that makes applications more scalable, more resilient to database failures, and more secure.
[Amazon Route 53](https://aws.amazon.com/route53/what-is-dns/) – Route 53 is a highly available and scalable cloud DNS web service. It’s designed to give developers and businesses an extremely reliable and cost-effective way to route end-users to internet applications by translating names like www.example.com into the numeric IP addresses like 192.0.2.1 that computers use to connect to each other.
[AWS SDK for Python (Boto3)](https://aws.amazon.com/sdk-for-python/) – The SDK for Python provides a Python API for AWS infrastructure services. With the SDK for Python, you can build applications on top of [Amazon Simple Storage Service](http://aws.amazon.com/s3) (Amazon S3), [Amazon Elastic Compute Cloud](https://aws.amazon.com/ec2/) (Amazon EC2), [Amazon DynamoDB](https://aws.amazon.com/dynamodb/), and more.
[Amazon SNS](https://aws.amazon.com/sns) – Amazon SNS coordinates and manages the delivery or sending of messages between publishers and clients, including web servers and email addresses. Subscribers receive all the messages published to the topics to which they subscribe, and all the subscribers to a topic receive the same messages.

## Prerequisites

This post assumes that you have an AWS account and cloud infrastructure such as a VPC, subnet groups, and a security group.
You can can use this solution with Amazon RDS, Aurora, and RDS Proxy, assuming that these AWS service components are in place.

## Pricing

Pricing of AWS services depends on many factors, for example AWS region of service, hourly/ monthly resource price, service utilization, and data transfer by service etc.
You can check the pricing in detail and calculate the pricing estimate on AWS service page or the links given below.

[AWS PrivateLink pricing](https://aws.amazon.com/privatelink/pricing/)
[Elastic Load Balancing pricing](https://aws.amazon.com/elasticloadbalancing/pricing/)
[Amazon RDS pricing](https://aws.amazon.com/rds/pricing/)
[AWS Lambda pricing](https://aws.amazon.com/lambda/pricing/)

## Create an IAM role in VPC-A

We will create an IAM role in client account, later we will be using Amazon Resource Name (ARN) of IAM role to establish trust relationship with Amazon RDS VPC. 
To create an IAM role in VPC-A, you [create an IAM role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html "https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html") with AssumeRole trust relationships. Then you attach the following IAM policy with least required privileges to the role:


{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "ec2:Describe*",
                "ec2:CreateTags",
                "ec2:CreateVpcEndpoint",
                "route53:AssociateVPCWithHostedZone"
            ],
            "Resource": "*"
        }
    ]
}


## Launch a CloudFormation stack in VPC-B

You can deploy this solution using a CloudFormation template. For more information, refer to [Creating a stack on the AWS CloudFormation console](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html).

1. Download the template from the Github repo.
2. Open the AWS CloudFormation console in the same Region where Amazon RDS is running.
3. Deploy the template NLB_Auto_Update_v1.0.yml.
4. For **Stack name**, enter a name (for this post, RDSAccess).
5. **Lambda Function Name**, enter a name for your Lambda function.
6. For **Select RDS Type**, specify if you’re using an Aurora cluster or RDS instance.
7. For **Enter (Amazon Aurora/Amazon RDS/Amazon RDS Proxy) endpoint**, enter an RDS writer endpoint.
8. For **Enter RDS Port**, enter an RDS port number. (This port number is used in Network Load Balancer to redirect connections to Amazon RDS.)
9. For **Select VPC from the list**, choose the VPC name or ID where you want to deploy the stack.
10. For **Select Subnets**, choose the private subnets for your resources. Make sure that the subnets belong to the VPC that you specified.
11. For **Select Security Group for Lambda**, choose the appropriate security group for Lambda.
12. For **Provide Role ARN of target Account/VPC**, enter the ARN of the role that you created in the VPC-A account. (The role ARN is used to establish a trust relationship in the endpoint.)
13. Choose **Next**.
14. Select the acknowledgement check box and choose **Create stack**.
15. After the stack is created, open the Lambda console and perform a test run to update the RDS primary instance IP address to the Network Load Balancer.
16. On the Amazon VPC console, choose **Endpoint services** and copy the service name to use in the next steps.

## Create the VPC endpoint in VPC-A

To create your VPC endpoint in VPC-A, complete the following steps:

1. Sign in to your account for VPC-A.
2. Switch to the role you created earlier.
3. On the Amazon VPC console, choose **Endpoints** in the navigation pane.
4. Choose **Create endpoint**.
5. Enter a name.
6. Select **Other endpoint services** and enter the endpoint service name you copied in the last section.
7. Choose **Verify service**. (If you have followed all the preceding steps correctly, you should get a message that the service name is verified.)
8. Choose the appropriate VPC.
9. Choose at least two private subnets in different Availability Zones for high availability.
10. Choose the appropriate security groups for the endpoint.
11. Add optional tags.
12. Choose **Create endpoint**.

When the endpoint is ready, you can retrieve the DNS names under the endpoint details. You use common DNS names (first on the list) to connect to Amazon RDS from VPC-A.

## Limitations

This solution works across AWS accounts and VPCs within the same Region. To make this solution works across Regions, use [AWS PrivateLink access over VPC peering](https://aws.amazon.com/about-aws/whats-new/2019/03/aws-privatelink-now-supports-access-over-vpc-peering/).

Depending on DNS TTL or [TCP keepalive parameter settings](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.BestPractices.html) on the client side, client traffic may still route to previous primary IP address for a while. Therefore, if the client session is connected to the reader instance (the previous primary instance, which is now the reader instance after the failover) and tries to run, the DML operation receives an error: ERROR: cannot execute INSERT in a read-only transaction.

## Conclusion

Amazon RDS Multi-AZ database deployment makes Amazon RDS highly available, and RDS failover handles Route 53 updates. RDS Proxy is also a highly available database proxy service that makes your applications more scalable and resilient to database failover, and doesn’t requires any additional mechanisms for handling RDS failover.

In this post, we learned how to implement a custom solution to connect to Amazon RDS hosted under different AWS accounts or VPCs, without VPC peering.

Leave your feedback in the comments section to further improve this post.
