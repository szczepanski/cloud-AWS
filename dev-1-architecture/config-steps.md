Following notes are based on udemy course by Sai Kiran Rathan. 

https://www.udemy.com/setup-aws-infrastructure-for-production-learn-terraform/learn/v4/overview

#  Architecture 

![alt text](https://github.com/szczepanski/cloud-aws/blob/master/dev-1-architecture/architecture%20design.png "Logo Title Text 1")


- Region - geographic
- VPC - private IP space 
- two availability zones - resilence
- 6 subnets - three per each availability zone

# VPC and Network set up
## Three Tier Architecture:
- Private subnet
- Public subnet
- DMZ subnet

## VPC components: 

- 2 Elastic load balancers - DMZ Subnet 
- 2 Application servers - public subnet 
- 2 database servers - private subnets

Client can only access ELB DMZ subnet. 

APP servers communicates only with ELB. 
Database servers communicate with App servers only

Security groups act as firewall - restricting access. 

## Steps

VLSM / CIDR Subnet calculator 
1. www.vlsm-calc.net
2. Vpc->create Vpc [specify name and CIDR range with subnet mask, no dedicated Tenancy]
192.160.0.0/19
3. Set Up 6 Subnets to override the 3 default subnets - size 256 hosts each 
![alt text](https://github.com/szczepanski/cloud-aws/blob/master/dev-1-architecture/Screen%20Shot%202018-03-11%20at%2013.17.37.png "Logo Title Text 1")

## Route table, Internet Gateway IGW and NAT Gateway 
Basic route table has been configured automatically once the initial VPC setup has been done. 
### Internet Gateway config and additional Route Tables 
4. Configure new Gateway and attach to VPC
5. Set Up Main Route Table - RT:
 Go to routes/ routes tab / edit/ and add another route:
0.0.0.0/0 - all web;  select taget IGW: Dev-1-IGW; save
6. Set up new RT - Subnet specific - DMZ:
Add internet address and target Dev-1-IGW. 
save and switch to subnet associations tab and add two dmz subnets
7. Set up new RT Subnet specific  - Public:
Add relevant subnets
8. Set up new RT Subnet specific  - Private:
Add relevant subnets
### NAT - Set up private route tables - NAT via NAT Gateway
Purpose for NAT - all private instances/ subnets can not be accessed via Internet BUT can access internet without Public IP - NAT. 
9. NAT Gateways > Create NAT GW - two (HIGH avialibility HA)
One NAT GW per availibility zone 
select public subnet 1 and allocate new EIP - elastic IP -  NAT GW 1
select public subnet 2 and allocate new EIP - elastic IP - NAT GW 2
10. Set up two new NAT private route tables (HA)
- 0.0.0.0/0 - all web;  select taget NAT GW 1; save
- switch to subnet associations tab and Private subnet 1
- 0.0.0.0/0 - all web;  select taget NAT GW 2; save
- switch to subnet associations tab and Private subnet 2

## Enable Auto Assign Public IP on subnets

Public IPs are required for application servers to be available over the internet. 

11. VPC/Subnets section 
- select Public Subnet 1 > modyfiy auto assign IP setting (subnet actions) => tick enable
- select Public Subnet 2 > modyfiy auto assign IP setting (subnet actions) => tick enable

# Application Server Setup 

## IAM Role - Identity and Access Management

Allows entities to call AWS services on one's behalf. 
Purpose: to allow instance to perform specific actions and stronger security as AWS will handle permissions behind the scenes. 

IAM => Roles - pick service that will be using the role: EC2
Always go for least previlleged access method. 

12. Create a role with no permissions as these will be established and added later. 
Name it Dev-1EC2-Role

## EC2 - Application Server 1 Setup

13. Launch new instance - EC2-Dev-1
- Amazon Linux
- t2 micro
- change network to VPC-Dev-1 
- select Public Subnet 1 (app server available in public domain) 
- ensure auto assign IP setting is enabled
- Select Role Dev-1-EC2-Role
- ensure shutdown behaviour is set to Stop
- Tenancy - shared
- keep network interface settings default
- go to next config steps - storage
- select default 8G general purpose storage
- ensure Delete on termination is ticked (to avoid extra costs wen not using it)
- go to add tags: (tags are needed also in terms of accountancy / billing - to recognize what instance added to costs)
  - Name: DEV-1 App Server 
  - Evironment: Development 
- configure new security group Dev-1 Public SG 
Leave SSH port open to single - own IP - for testing 
- Add Rule - HTTP 0.0.0.0/0 
- create and save new Key Pair - Dev-1-KeyPair.pem
when testing ssh - ensure pem lsfile permissions are set to min 400

## ssh testing
14. ssh -i /KeyPair/file/location.pem  ec2-user@x.x.x.x

## initila server setup / patch
15. 
sudo yum update -y  [option -y => promptless]
sudo yum install -y httpd php
sudo service httpd startig

## verify http appache server 
browser or curl to server public IP

## amend landig page - optional
```shell 
cd /var/www/html
echo "Piotr Testing Page" > index.html
``` 

## S3 configuration

16. configure standard S3 bucket, add new folder => builds and upload 
sample page / application  

## IAM Role adjustment
17. Create policy 
IAM>Roles>Dev-1-EC2-role> Add inline policy>JSON Tab
For assistance:
https://docs.aws.amazon.com/AmazonS3/latest/dev/using-with-s3-actions.html
```JSON
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "dev1s3accesspolicy",
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::piotr.szczepanski/Dev-1/*",
                "arn:aws:s3:::piotr.szczepanski/Dev-1"
            ]
        }
    ]
}
```
18. Attach above policy to Dev-1-EC2-role

## Adjust ec2 app server via ssh 
19. 
```shell
aws s3 cp help
aws s3 cp s3://piotr.szczepanski/Dev-1/builds/Dev-1-landingPage.zip Dev-1-landingPage.zip
rm index.html 
unzip Dev-1-landingPage.zip
rm -fR Dev-1-landingPage.zip
```
## update user data script

20. 
-y => promptless command flag
```shell
#!/bin/bash

sudo su
yum update -y
yum install -y httpd php
cd /var/www/html
aws s3 cp s3://piotr.szczepanski/Dev-1/builds/Dev-1-landingPage.zip Dev-1-landingPage.zip
unzip Dev-1-landingPage.zip
mv Dev-1-landingPage/* .
rm -rf Dev-1-landingPage.zip Dev-1-landingPage
sudo service httpd start
```

## Lunch Dev-1 EC2 Application Server 1 from scratch 
21. Add steps as before plus add script in advanced setting - user data

## Check Services and deamons
service --status-all

## EC2 - Application Server 2 Setup
22. To create App Server 2 use rigth clic > launch more like this
Adjust name and subnet => change to 2nd availibility 

## 
https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html

# Load balancer Configuration

https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html

![alt text](https://github.com/szczepanski/cloud-aws/blob/master/dev-1-architecture/load_balencer_architecture.png)

## Set up

23. Load balancer type => select Application LB
Network LP option = > select only for high performance networks, static IPs etc.  

Set name, internet facing, protocol http,select VPC,select avail. zones and related subnets:
eu-west-2a  	DMZ Subnet 1
eu-west-2b 	 DMZ Subnet 2

Add tags: name, and environment
24. Configure Security Groups => name and create new security group => leave default tcp 80 ; to be fine tuned later
25 Configure Routing and Health checks- name and create new target group
leave 

![alt text](https://github.com/szczepanski/cloud-aws/blob/master/dev-1-architecture/ALB%20config.png)

26. Register Targets
Add two app servers: 
Dev-1 EC2 Application Server 1
Dev-1 EC2 Application Server 2
Review and create

## Load balancer checks 

All stats (latency, requests, health and more) can be obtained Load Balancer or target groups. 
To access app servers via the App Load balancer, copy paste into browser DNS A record (Load Balancer/ description tab)

## Security Groups clean up 
Ensure client interaction is restricted only to Load balancers level - DMZ zones in following way:  

- ensure only load balancers are facing incoming web traffic

- ensure public subnet(app servers) can receive web traffic (port80) from App Load balancers ONLY. 
27. Amend Dev-1 Public SG (App servers SG) so it gets traffic from Dev-1-External-Application-Load-Balancer-SG by pasting ALB SG ID into source field. 

![alt text](https://github.com/szczepanski/cloud-aws/blob/master/dev-1-architecture/adjust%20public%20SG.png)

- 28. ensure traffic from load balances - outbound - goes to app servers ONLY (not further to DB) - by pasting Dev-1 Public SG (Apps Servers SG) ID into App Load Balancers SG - outbound - destination field 

![alt text](https://github.com/szczepanski/cloud-aws/blob/master/dev-1-architecture/adjust%20load%20balancer%20SG.png)

- ensure there is no wide open ssh access to app servers

# Auto Scaling Groups Introduction

hhttps://docs.aws.amazon.com/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html

![alt text](https://github.com/szczepanski/cloud-aws/blob/master/dev-1-architecture/autoscaling.png)

## Create launch configuration
29. EC2>Launch Autoscaling

![alt text](https://github.com/szczepanski/cloud-aws/blob/master/dev-1-architecture/LC%20setup.png)

Add default storage, tick delte on termination
Select existing public security group for the EC2 app servers - Dev-1 Public SG
Review, select .pem file and proceed to 

## Create Auto Scaling Group

30. 

![alt text](https://github.com/szczepanski/cloud-aws/blob/master/dev-1-architecture/AS%20group.png)

Save and uodate as follows:

![alt text](https://github.com/szczepanski/cloud-aws/blob/master/dev-1-architecture/update%20asg.png]
Save the update and check 
Verify wether the minimum 2 instances launched in correct 2 avail.zones.
ensure Application LB A DNS record works (browser check)

## Create Scaling Policies, Cloudwatch alarms, Simple Notification Services - SNS

31. Go to SNS and create 4 topics:
Dev-1 Scale up alarm 
Dev-1 Scale down alarm 
Dev-1-Service-Anomaly 
Dev-1-AutoScalingActivityAlarm

32. EC2>Auto Scaling Target GroupsScaling Policies> Add policy> create simple Scale policy

Create Scale UP policy 

![alt text](https://github.com/szczepanski/cloud-aws/blob/master/dev-1-architecture/create%20scaling%20policy.png)


and create new alarm - High cpu -to trigger/add one new instance)

![alt text](https://github.com/szczepanski/cloud-aws/blob/master/dev-1-architecture/new%20alarm%20scale%20up.png)


33. Create Scale down policy and alarm - low cpu - for cpu utilzation <=20%  - to remove one oldest instance, save

34. switch to notifications tab and create new notification

create new notification linked to Dev-1-AutoScalingActivityAlarm
This wil alarm whenever there is new launch,  termination, fail to launch or terminate.

# To be completed:

## finish auto scaling
## Create and Configure DB Instance - mySQL RDB in private subnet
## Manage & Cofigure DNS - Route 53
## Set Up Slaack


# Terraform

Open source by hashicorp 

Use cases: - infrastructure versin control and back up if prod config breaks; for multiple environments - minimalization of config drift. 

 3 Basic components:
- confif file.tf - written in hashicorp configuration language - HCL 
- cli => terraform plan
- cli => terraform apply

Easy variable declaratin. 
Good documentation. 
Referencing files  => such as user data. 



























 
