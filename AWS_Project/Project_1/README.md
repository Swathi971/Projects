## End to end web deployment

#### Create VPC  
<img src=".github/images/img_1.png" alt="Create VPC 1" width="700"/>  
<img src=".github/images/img.png" alt="Create VPC 2" width="700"/>

Now go to VPC settings - Enable DNS Hostnames - Save  
<img src=".github/images/img_4.png" alt="DNS Hostnames" width="700"/>

#### Create 4 subnets  
<img src=".github/images/img_2.png" alt="Subnets" width="700"/>

      
Created 2 public subnets in us east n.verginea 1a and 2 private subnets in u.s east n.verginea 1b

##### Enable public ip address to both public subnets: 

Select the my-vpc-public-subnet1- Actions- Edit subnet settings- select Enable auto-assign public IPv4 address. 
<img src=".github/images/img_5.png" alt="Subnets" width="50%"/>

##### Create Route tables: 
I need two route tables; one is for private, and another is for public subnet. 
###### creating route table for public subnet:

<img src=".github/images/img_6.png" alt="routetable" width="50%"/>

##### Adding public subnets to the route table: 
<img src=".github/images/img_7.png" alt="subnet association" width="50%"/>

<img src=".github/images/img_8.png" alt="adding public subnets" width="50%"/>

###### Creating route table for private subnet:
<img src=".github/images/img_9.png" alt="adding public subnets" width="50%"/>

##### Adding private subnets to the route table: 
<img src=".github/images/img_10.png" alt="adding public subnets" width="50%"/>

##### Creating an Internet gateway:
<img src=".github/images/img_11.png" alt="adding public subnets" width="50%"/>

##### Attach the created internet gateway to the VPC: 
<img src=".github/images/img_12.png" alt="attaching igw" width="50%"/>

<img src=".github/images/img_13.png" alt="attaching igw" width="50%"/>

##### Configuring internet gateway to the public subnet:  
<img src=".github/images/img_14.png" alt="configuring" width="50%"/>

<img src=".github/images/img_15.png" alt="configuring" width="50%"/>

#### Launch an Ec2 instance
     Name- My-Webapp 
     AMI- Ubuntu Server 24.04 LTS (HVM),EBS General Purpose (SSD) Volume Type
____________________








