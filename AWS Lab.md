 - 

# Lab 1 : Create an environment

 Go to [https://aws.amazon.com/login](https://aws.amazon.com/free/) and choose the region

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
	- Name : YourName-Priv-Sub
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
- AMI Amazon Linux
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








<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0MTYyMTkxOCwxNzQ4MTM2MzcyLDE1NT
gxMzQyMjcsMTQ4NTgwNDI4NywtNzU0ODYzNzgwLC0xNzEyNzUx
NDQ5LC03ODg1NDU4OTksMTYyMTgwNTU0MCwyMDAzODY0MTM0LC
0xMTUxMTE0MDUwLDExODQ0MTYzNzIsMTY1MDcyNjU1NSwtMjA3
NjgzMTQ1OSwtMjAyMDc1MjEwLDE1NTIxMjkyMDUsMTI0NDQ0Nz
YzMywxMzUzNzE0OTA0LDIxOTQxNTA3NywtNTU4OTU2NzUyLC0x
MDQzODkyODA3XX0=
-->