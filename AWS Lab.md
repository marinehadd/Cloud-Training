 - 

# Lab 1 : Create an environment

 Go to your AWS account and choose the region.

## Setup a VPC

 Go to VPC services then Your VPCs and create VPC
	- Click Create
	- Enter a name i.e. VPC-YourName
	- IPv4 CIDR : 10.0.0.0/16

You now have a VPC and an associated routing table created. Your VPC will need to be able to host instances accessible from both private environment and the Internet.

Go to Subnets and create a public and a private subnets.
- Subnet 1:
	- Name : YourName-Pub-Sub
	- Select your VPC
	- Select the first availability zone of your region (i.e. 1a)
	- IPv4 CIDR 10.0.1.0/24
- Subnet 2:
	- Name : YourName-Priv-Sub-1
	- Select your VPC
	- Select the second availability zone of your region (i.e. 1b)
	- IPv4 CIDR 10.0.2.0/24

Select Subnet 1, go to Actions and modify the auto-assign IP settings to allow the subnet to obtain public IP addresses dynamically.
In Routes Tables, select your VPC route table and notice the two subnets associated. They should find the ones you created above.

The public subnet needs access to the Internet which will be provided by an Internet Gateway.
Go to Internet Gateways and create one YourName-IGW. Then select it, Actions, and attach to your VPC.

Go back to your route table and notice that you have a new route 0.0.0.0/0 which is the association of your internet gateway. This means that both /24 subnets can now access the internet via this gateway.
However, only the public subnet (subnet 1) should be able to access the Internet so we will need to make a different route table for each type of access (public and private).

Create a route table:
- Name : YourName-Pub-RT
- VPC : Your VPC

Add an Internet Gateway:
- Edit routes then Add routes
- Destination 0.0.0.0/0
- Target : Internet Gateway and select your IGW
Save Routes

Associate the public subnet to the public route table:
- Edit subnet associations
- Select the public subnet
Save

## Setup a bastion

A bastion is a jump box that will allow you to connect to any private instance of your VPC from the Internet. It will act as an intermediate between the public and the private environments.

Go to EC2 Dashboard, launch an instance:
- AMI Amazon Linux 2
- Leave default Instance Type
- Instance details:
	- Network : your VPC
	- Subnet : your public subnet
	- Auto-assign public IP : YES
- Leave default storage
- Tag : Bastion-YourName
- Create a new Security Group
	- Name : Bastion-SG-YourName
	- Setup SSH and HTTP with source 0.0.0.0/0
- Review and Launch then Launch
- Create a new key pair EC2-KP-YourName and save your pem file on your local drive

Download PuttyGen (and Putty if you haven't already):
[https://puttygen.com/download.php?val=49](https://puttygen.com/download.php?val=49)  (select manual download)

Open PuttyGen:
- Load your pem file (select "all files")
- Save private key (no paraphrase) as a ppk file

Go back to the AWS console, and copy the public IP of your Bastion instance.
Open Putty:
- Hostname : ec2-user@publicIP
- Expand SSH, click Auth and upload your ppk file
- Open

You are now logged into your Bastion instance.

## Setup a private instance

Go to EC2 Dashboard, launch an instance:
- AMI Amazon Linux
- Leave default Instance Type
- Instance details:
	- Network : your VPC
	- Subnet : your private subnet
	- Auto-assign public IP : leave default
- Leave default storage
- Tag : Server-YourName
- Select the default existing Security Group
- Review and Launch then Launch
- Select the existing key pair EC2-KP-YourName, acknowledge and launch.

Follow the same step to SSH to the private instance with Putty. You should get a connection error. This is due to the fact that:
- the subnet is private and has no public IP assigned
- the subnet route table has no internet gateway
- the security group does not allow the subnet to be reachable from anywhere

Therefore, you need to use the bastion (jump box) in order to login to the private instance from your network.
Open the pem file with notepad and copy the content (all of it).
Go back to the Bastion CLI and load the private key to be able to authenticate the bastion to the private server:
- Type *sudo su*
- Type *nano yourname-pk.pem*
- Right click (that will paste the content you copied)
- Ctrl+X then enter
- Type *ssh ec2-user@privateIP -i yourname-pk.pem*

You should now be logged into the private instance CLI.


# Lab 2 : Create a Web Server

We will use the private instance to create our web server. Therefore, the instance needs Internet access and a public IP to browse to it.

Given the instance is already running, we cannot edit its settings so we will map an elastic (public) IP address to it.
In VPC, go to Elastic IPs and create a new one for your VPC. Then edit and attach it to your private EC2 instance.

Go to Route Tables and associate the private subnet to the public route table where you initially configured the Internet Gateway.

Go to Security Groups and add Custom TCP (port 5000) from 0.0.0.0/0 on the private instance security group. Also, rename it with web instead of private and do the same for the EC2 instance.

Login to the web server CLI (via the bastion since SSH access is restricted for mgmt purpose), run the below to install server packages:
*sudo su

yum install git

yum install python 2.7

yum install python-pip

pip install flask

pip show flask

git clone https://github.com/MonkeyNadz/MyFirstFlask.git

cd MyFirstFlask/app/

python app.py*


On chrome or IE, browse to HTTP://ElasticIP:5000.

# Lab 3 : Create a S3 bucket

Go to S3 services and create a bucket:
- Name : S3-YourName
- Region should be the same as your EC2 instances

Skip the next steps and click Create.

On the web server CLI, type *aws s3 ls*to display the S3 buckets list. 
The command fails because your instance has no right to access the S3 services.

Go to IAM services and create a role to give your EC2 instance access to the S3 services:
- Trusted Entity : AWS Services
- Used by : EC2 instance
- Policy : AmazonS3FullAccess
- Tag : S3 admin access
- Role Name : S3adminYourName

Go to EC2 services and attach the IAM role to the web server.

Retype the same command to check if you now have access to S3. The command should list all the S3 buckets, including the one you created above.

Create a txt file:
*echo "This is my new file." > yourname.txt

s3 aws cp yourname.txt s3://yourbucketname*

Go back to the console and check that the new file has been uploaded. Select it, and click on the URL displayed on the right panel to check the content of the file.
If you cannot access it, it means that your bucket has no public access.

Go back to the bucket list :
- Select your bucket
- Edit public access settings
- Make sure all boxes are unticked and Save

Click on your bucket, click on your txt file and click "make public" in the Overview section.

Click again on the file URL and the content should now be displayed.


# Lab 4 : Create an ELB

In this lab, you will create a second web server in order to load balance the traffic between them.

Create an EC2 instance with the below settings:
- AMI Amazon Linux
- Leave default Instance Type
- Instance details:
	- Network : your VPC
	- Subnet -> click create a new subnet (directed to a new page)
		- Name : YourName-Priv-Sub-2
		- Select your VPC
		- Select third availability zone of your region (i.e. 1c)
		- IPv4 CIDR 10.0.3.0/24
		- Create and go back to EC2 launch page
	- Auto-assign public IP : enable
- Leave default storage
- Tag : WebServer02-YourName
- Select the existing security group of your web server 01
- Review and Launch then Launch
- Select the existing key pair EC2-KP-YourName, acknowledge and launch.

Go to Route Tables in VPCs, select the public route table where your web server 01 and your bastion subnets are associated, and associate the web server 02 subnet to the table.

Login to both CLI web servers and run the below commands to install an http server :
*yum install httpd -y

service httpd start

chkconfig httpd on

cd /var/www/html

echo "< html >< h1 >Hello World X< /h1 ></html>" > index.html* ** 
**(X being 1 for webserver 1 and 2 for webserver2)****

Browse to their public IP to HTTP://IP/index.html

Go to Load Balancers and create a Classic Load Balancer:
- Name : WebServer-LB-Name
- Select your VPC
- Listener on HTTP 80
- Select the two web server subnets
- Security Group : use the same as the web servers
- Ping path : /index.html
- Edit healthchecks thresholds to 2sec everywhere
- Leave the rest as default and create

When you select it once ready, the web servers load balanced are displayed at the bottom. They should both come "in-service".
On the description tab, copy the DNS name of the load balancer and browse to it. Keep refreshing with CTRL+F5 to notice that the content is coming alternatively from web server 1 and web server 2.

Go to EC2 instances and stop the web server 1. Then go back to Load Balancers, the server should show as out of service. 
Refresh the web page again several times, only the content of web server 2 should get displayed.

Create a friendly name for your load balancer. Copy the current name on a notepad.
Go to Route53 then hosted zones, click on ici-training.net and create a new record set:
-Name : choose a new friendly name 
- CNAME record
- Value : your DNS LB name

Now browse to the new name you choose, you should get the same web page as before.


# Lab 5 : Create an auto-scaling group

In EC2, go to your instances and stop your two web servers.

Then go to Auto-scaling group and create a Launch Configuration:
- AMI Amazon Linux 2
- Name - Launch-yourname
- IAM role : choose the S3 IAM role you created previsouly
- Click on Advanced details and paste the below text into the box:

#!/bin/bash

yum update -y

yum install httpd -y

service httpd start

chkconfig httpd on

cd /var/www/html

echo "< html >< h1>Hello world, this is my new auto-scaling group!< /h1>< /html >" > index.html

- select "Assign a public IP address to every instance"
- Select your web server security group (or default)
- Choose your existing key pair
- Create auto-scaling group using this launch configuration
	- Name : AS-yourname
	- Size : 2
	- Select both private (web server) subnets one by one
	- Click Advanced Details
		- Tick the Load Balancing box and select your classic load balancer
		- Leave the rest as default
	- Tick "Leave this group as its initial size"
	- Tag : Name -> description
	- Review and Create

Go to EC2 instances, you should have 2 new instances created.
SSH to your bastion and login to both new instances private IPs to make sure they are reachable.
Then, in your browser, for each new instance (new web server) type the public IP to make sure you get the new web content.

In your browser, type your load balancer URL (either the one displayed in the load balancer configuration, or the DNS CNAME name you created in Route53). You should get the new web content displayed.

Go back to your instances, terminate one of the two web servers from the auto-scaling group.

Wait for a few minutes and refresh the page, a new auto-scaling instance should appear.

Once it is up, re-test your load balancer URL access.








 







<!--stackedit_data:
eyJoaXN0b3J5IjpbMTM0ODQzMTg2OSwtMTYyMTExNjQ5OSwtMT
g5MTM1OTA1OCwxMTUyNDAzNjY5LC0xODQyNTI2MzkwLDE2MDY3
MTc0MTcsLTY2MDc5ODk2OCwxNzM3NzE4ODAsNzM2NTYwNDc1LC
0xOTU1NTAxODU5LC0xNzQ2MTcwODksLTc3NTc1NjQ2MCw3MDk3
MDg4NDQsLTE0MTYyMTkxOCwxNzQ4MTM2MzcyLDE1NTgxMzQyMj
csMTQ4NTgwNDI4NywtNzU0ODYzNzgwLC0xNzEyNzUxNDQ5LC03
ODg1NDU4OTldfQ==
-->
