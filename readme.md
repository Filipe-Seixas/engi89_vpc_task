# Virtual Private Cloud (VPC) Task

<p align=center>
	<img src=vpc_diagram.PNG>
</p>

## Creating a VPC

- On AWS Console > VPC > Create VPC
	- Name: eng89_infra_filipe_vpc
	- IPv4: 10.202.0.0/16   255.255.0.0
	- Leave rest as default and create

## Creating an Internet Gateway

- Go to Internet Gateways > Create IG
	- Name: eng89_infra_filipe_ig
- > actions > attach vpc
	- select eng89_infra_filipe_vpc

## Creating the subnets

- Go to Subnet > Create subnet

### App Subnet
- VPC ID: Select eng89_infra_filipe_vpc
- Name: eng89_infra_filipe_app_server
- Availability zone: eu-west-1a
- IPv4 CIDR: 10.202.1.0/24
- > add new subnet

### DB Subnet
- Name: eng89_infra_filipe_db_server
- Availability zone: eu-west-1a
- IPv4 CIDR: 10.202.2.0/24
- > add new subnet

### Bastion Subnet
- Name: eng89_infra_filipe_bastion_server
- Availability zone: eu-west-1a
- IPv4 CIDR: 10.202.3.0/24
- > create subnet

## Creating a Routing Table

- Now we want to create a routing to allow subnets to communicate through the internet

- Go to Route Tables > Create Route Table
	- Name: eng89_infra_filipe_route_internet
	- VPC: Select eng89_infra_filipe_vpc
	- > Create route table
- > Edit routes
	- > Add route
	- Destination: 0.0.0.0/0
	- Target: eng89_infra_filipe_ig

### Associating Routing Table to Subnets

- Go to Subnets > select app subnet
	- > Route table > Edit route table association
	- Change route table id to our new route table
- Repeat for bastion subnet, NOT DB since it doesn't need to access the internet

## Creating the Security Groups

- A security group is connected to ec2 instance, it's the firewall, used to allow and deny access to ports, we want to protect the instance. Network ACL is for whole network

- Inbound Rules: who can access the instance, control what's coming in
- Outbound Rules: What the instance can access, control what's going out

- Stateful firewall: port open in one direction and the firewall will remember to allow the response to pass in the opposite direction
	- SG are stateful
	- ACLs are stateless. The network ACL checks all rules, if it gets to the end and doesn't get a match, it denies it

- Go to Security Groups > Create SG

### App SG
- Name: eng89_infra_filipe_app_sg
- VPC: eng89_infra_filipe_vpc
- Inbound Rules:
	- HTTP / Anywhere-IPv4
	- SSH / My IP
- Outbound Rules:
	- All traffic / Anywhere-IPv4
- Add tag: name - eng89_infra_filipe_app_sg

### DB SG
- Name: eng89_infra_filipe_db_sg
- VPC: eng89_infra_filipe_vpc
- Inbound Rules:
	- Custom TCP / port 27017 / source Custom / eng89_infra_filipe_app_sg
	- SSH / Custom / eng89_infra_filipe_bastion_sg
- Outbound Rules:
	- All traffic / Anywhere-IPv4
- Add tag: name - eng89_infra_filipe_db_sg

### Bastion SG
- Name: eng89_infra_filipe_bastion_sg
- VPC: eng89_infra_filipe_vpc
- Inbound Rules:
	- SSH / My IP
- Outbound Rules:
	- All traffic / Anywhere-IPv4
- Add tag: name - eng89_infra_filipe_bastion_sg

## Creating the Network Access Control Lists (ACL)

- Go to Network ACL > Create ACL

### App ACL
- Name: eng89_infra_filipe_app_acl
- VPC: eng89_infra_filipe_vpc
- > Create acl
- > Subnet associations > Edit subnet associations
	- Select app subnet
- Inbound Rules:
	- 100 / HTTP / Anywhere-IPv4
	- 110 / SSH / My IP/24
	- 120 / Custom TCP / port 1024-65535 / 0.0.0.0/0
- Outbound Rules:
	- 100 / HTTP / Anywhere-IPv4
	- 110 / Custom TCP / port 27017 / 10.202.2.0/24 (db ip)
	- 120 / Custom TCP / port 1024-65535 / 0.0.0.0/0

### DB ACL
- Name: eng89_infra_filipe_db_acl
- VPC: eng89_infra_filipe_vpc
- > Create acl
- > Subnet associations > Edit subnet associations
	- Select db subnet
- Inbound Rules:
	- 100 / Custom TCP / port 27017 / 10.202.1.0/24 (app ip)
	- 110 / Custom TCP / port 1024-65535 / 0.0.0.0/0
	- 120 / SSH / 10.202.3.0/24 (bastion ip)
- Outbound Rules:
	- 100 / HTTP / Anywhere-IPv4
	- 110 / Custom TCP / port 1024-65535 / 10.202.1.0/24 (app ip)
	- 120 / Custom TCP / port 1024-65535 / 10.202.3.0/24 (bastion ip)

### Bastion ACL
- Name: eng89_infra_filipe_bastion_acl
- VPC: eng89_infra_filipe_vpc
- > Create acl
- > Subnet associations > Edit subnet associations
	- Select bastion subnet
- Inbound Rules:
	- 100 / SSH / My IP/32
	- 110 / Custom TCP / port 1024-65535 / 10.202.2.0/24 (db ip)
- Outbound Rules:
	- 100 / SSH / 10.202.2.0/24 (db ip)
	- 110 / Custom TCP / 1024-65535 / 0.0.0.0/0

## Creating the Instances

- Go to ec2 Instances > Launch Instance

### App Instance
1. AMI > My AMI > eng89_filipe_app_ami
2. Free tier
3.
	- Network: eng89_infra_filipe_vpc
	- Subnet: eng89_infra_filipe_app_server
	- Auto Assign Public IP: Enable
4. Next
5. Name: eng89_infra_filipe_app_ec2
6. Existing SG: app_sg

### DB Instance
1. AMI > My AMI > eng89_filipe_db_ami
2. Free tier
3.
	- Network: eng89_infra_filipe_vpc
	- Subnet: eng89_infra_filipe_db_server
	- Auto Assign Public IP: Disable
4. Next
5. Name: eng89_infra_filipe_db_ec2
6. Existing SG: db_sg

### Bastion Instance
1. AMI > Ubuntu
2. Free tier
3.
	- Network: eng89_infra_filipe_vpc
	- Subnet: eng89_infra_filipe_bastion_server
	- Auto Assign Public IP: Enable
4. Next
5. Name: eng89_infra_filipe_bastion_ec2
6. Existing SG: bastion_sg

## Connecting to the Instances

- Copy key into bastion server `scp -i eng89_devops.pem eng89_devops.pem ubuntu@BASTION_IP:~/`
- SSH into Bastion `ssh -i "eng89_devops.pem" ubuntu@BASTION_IP`
- Move key to .ssh folder `mv eng89_devops.pem .ssh/`
- YOU HAVE TO BE IN .SSH FOLDER TO CONNECT TO DB `cd .ssh/`
- Change key permissions `chmod 400 eng89_devops.pem`
- SSH into db `ssh -i "eng89_devops.pem" ubuntu@DB_IP `

## Deploying the app

- Make sure all these points below are correct in App:
	- App SG needs inbound rule for port 3000
	- nginx default file (app ip)
		- sudo systemctl restart nginx
		- sudo systemctl enable nginx
	- DB_HOST env (db ip, can be private ip)
	- seed db
