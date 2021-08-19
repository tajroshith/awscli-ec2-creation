## EC2 - Configuration & Deployment Using AWS-CLI

This is a project i did for a client, the requirement was to build an infrastucture for a webserver with no access to the AWS console. 
Further requirement was to create and attach an additional volume to the ec2 instance. An IAM user with programmatic access was also provided.
Below is a detailed summary of  steps that i have performed to complete the project.


## Pre-requisites for this project

- Needs AWS CLI access preferrably a user with admin access
- AWS CLI needs to be installed on your system
- Knowledge of AWS CLI commands

## Resources

List of AWS resources created via AWS-CLI

- EC2
- Security Group
- AMI (Amazon Machine Image)
- Key-Pair
- Additional EBS Volume
- Internet Gateway
- Route Table
- VPC

## Configuring the AWS CLI User

```sh
aws configure --profile=taj-awscli
```
Output

```sh
AWS Access Key ID [None]: Provide AWS Access Key ID
AWS Secret Access Key [None]: Provide AWS Secret Access Key
Default region name [None]: Provide region name
Default output format [None]: Provide output format
```
We configured the AWS-CLI admin user with the access key and secrey key.

## Creating Infrastructure for EC2 Deployment

We create a VPC with CIDR-Block of 172.17.0.0/16

```sh
aws ec2 create-vpc --cidr-block 172.17.0.0/16
```

Output

```sh
{
    "Vpc": {
        "CidrBlock": "172.17.0.0/16",
        "DhcpOptionsId": "dopt-37b8c65c",
        "State": "pending",
        "VpcId": "vpc-0f8eb530f20a64a39",
        "OwnerId": "898279796273",
        "InstanceTenancy": "default",
        "Ipv6CidrBlockAssociationSet": [],
        "CidrBlockAssociationSet": [
            {
                "AssociationId": "vpc-cidr-assoc-016cd714c07a59db3",
                "CidrBlock": "172.17.0.0/16",
                "CidrBlockState": {
                    "State": "associated"
                }
            }
        ],
        "IsDefault": false
    }
}
```

Assigning the VPC a tag "webserver-vpc" for proper identification.

```sh

aws ec2 create-tags --resources vpc-0f8eb530f20a64a39 --tags Key=Name,Value=Webserver-vpc

```

We now create 3 subnets for our infrasturcture, the subnets will be assigned with the following CIDR blocks

```sh

aws ec2 create-subnet --vpc-id vpc-0f8eb530f20a64a39 --cidr-block 172.17.0.0/18
```

```sh
aws ec2 create-subnet --vpc-id vpc-0f8eb530f20a64a39 --cidr-block 172.17.64.0/18
```

```sh
aws ec2 create-subnet --vpc-id vpc-0f8eb530f20a64a39 --cidr-block 172.17.128.0/18
```

To provide access to the internet we create an internet gateway for our VPC.

```sh
aws ec2 create-internet-gateway
```

Output

```sh

{
    "InternetGateway": {
        "Attachments": [],
        "InternetGatewayId": "igw-0264fa7f3b74797b8",
        "OwnerId": "898279796273",
        "Tags": []
    }
}

```

As with the previous steps we assign tags for proper identification.

```sh
aws ec2 create-tags --resources igw-0264fa7f3b74797b8 --tags Key=Name,Value=igw-webserver

```

Now we need to attach the internet gateway to our VPC

```sh
aws ec2 attach-internet-gateway --internet-gateway-id igw-0264fa7f3b74797b8 
--vpc-id vpc-0f8eb530f20a64a39

```

We create a custom route table so as to route the traffic to our internet gateway.

```sh
aws ec2 create-route-table --vpc-id vpc-0f8eb530f20a64a39

```

Output

```sh

{
    "RouteTable": {
        "Associations": [],
        "PropagatingVgws": [],
        "RouteTableId": "rtb-0e17ff7cacbe06aac",
        "Routes": [
            {
                "DestinationCidrBlock": "172.17.0.0/16",
                "GatewayId": "local",
                "Origin": "CreateRouteTable",
                "State": "active"
            }
        ],
        "Tags": [],
        "VpcId": "vpc-0f8eb530f20a64a39",
        "OwnerId": "898279796273"
    }
}

```

Now pointing our route table to our internet gateway to route all traffic.

```sh
aws ec2 create-route --route-table-id rtb-0e17ff7cacbe06aac --destination-cidr-block 0.0.0.0/0 
--gateway-id igw-0264fa7f3b74797b8

```

Tagging the route table.

```sh
aws ec2 create-tags --resources rtb-0e17ff7cacbe06aac --tags Key=Name,Value=rtb-webserver
```

To verify the routes we created in route table we use the following command to describe the route table and its contents.

```sh
aws ec2 describe-route-tables --route-table-id rtb-0e17ff7cacbe06aac

```

Output

```sh

{
    "RouteTables": [
        {
            "Associations": [],
            "PropagatingVgws": [],
            "RouteTableId": "rtb-0e17ff7cacbe06aac",
            "Routes": [
                {
                    "DestinationCidrBlock": "172.17.0.0/16",
                    "GatewayId": "local",
                    "Origin": "CreateRouteTable",
                    "State": "active"
                },
                {
                    "DestinationCidrBlock": "0.0.0.0/0",
                    "GatewayId": "igw-0264fa7f3b74797b8",
                    "Origin": "CreateRoute",
                    "State": "active"
                }
            ],
            "Tags": [
                {
                    "Key": "Name",
                    "Value": "rtb-webserver"
                }
            ],
            "VpcId": "vpc-0f8eb530f20a64a39",
            "OwnerId": "898279796273"
        }
    ]
}

```

We now associate the created subnets with our route table which in turn is now connected to the internet gateway. First we use the following command to identify our subnets.

```sh
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-0f8eb530f20a64a39" 
--query "Subnets[*].{ID:SubnetId,CIDR:CidrBlock}"

```

Output

```sh
[
    {
        "ID": "subnet-00c0a8f7de82f11a8",
        "CIDR": "172.17.128.0/18"
    },
    {
        "ID": "subnet-0e84cc4af28cca9ef",
        "CIDR": "172.17.64.0/18"
    },
    {
        "ID": "subnet-0032eb27a14442a1e",
        "CIDR": "172.17.0.0/18"
    }
]

```

Associating subnets with our route table

```sh

aws ec2 associate-route-table  --subnet-id subnet-00c0a8f7de82f11a8 
--route-table-id rtb-0e17ff7cacbe06aac

aws ec2 associate-route-table  --subnet-id subnet-0e84cc4af28cca9ef 
--route-table-id rtb-0e17ff7cacbe06aac

aws ec2 associate-route-table  --subnet-id subnet-0032eb27a14442a1e 
--route-table-id rtb-0e17ff7cacbe06aac

```

Instances launched using these subnets require public-IP so we modify the subnets attribute as such it auto-assigns a public-IP.

```sh

aws ec2 modify-subnet-attribute --subnet-id subnet-00c0a8f7de82f11a8 
--map-public-ip-on-launch

aws ec2 modify-subnet-attribute --subnet-id subnet-0e84cc4af28cca9ef 
--map-public-ip-on-launch

aws ec2 modify-subnet-attribute --subnet-id subnet-0032eb27a14442a1e 
--map-public-ip-on-launch


```

Now our basic infrastructure is completed and we move onto creation of EC2.

## Creating Security Group

We create the security group in our VPC and tag it for proper identification.

```sh
aws ec2 create-security-group --group-name web-sg --description "Security group for webserver" 
--vpc-id vpc-0f8eb530f20a64a39

```

```sh

aws ec2 create-tags --resources sg-08e4bfd92f793c6a5 --tags Key=Name,Value=web-sg

```

Output

```sh

{
    "GroupId": "sg-08e4bfd92f793c6a5"
}

```

Inorder to access our EC2 instance via SSH we are opening port 22 and also ports 80 & 443 as we need to access the webserver.

```sh
aws ec2 authorize-security-group-ingress --group-id sg-08e4bfd92f793c6a5 --protocol tcp 
--port 22 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress --group-id sg-08e4bfd92f793c6a5 --protocol tcp 
--port 80 --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress --group-id sg-08e4bfd92f793c6a5 --protocol tcp 
--port 443 --cidr 0.0.0.0/0

```

## Creating Key-Pair

The proceeding line creates a 2048-bit RSA key pair. The aws ec2 command stores the public key and outputs the private key to save to a file.

```sh
aws ec2 create-key-pair --key-name cli-web-keypair --query "KeyMaterial" 
--output text > cli-web-keypair-key.pem

```

## AMI (Amazon Machine Image)

When creating a EC2 instance from the command line, we specify the operating system using the amazon machine image (AMI) ID. To get the image ID we use the following command which lists the latest AMI Image Id.

```sh
aws ec2 describe-images --owners amazon --filters "Name=name,Values=amzn2-ami-hvm-2.0.????????-x86_64-gp2" 
"Name=state,Values=available" --output json

```

From the output we get the AMI ID

Output

```sh

 {
            "Architecture": "x86_64",
            "CreationDate": "2019-02-19T22:42:34.000Z",
            "ImageId": "ami-0ed72083dbed1d548",
            "ImageLocation": "amazon/amzn2-ami-hvm-2.0.20190218-x86_64-gp2",
            "ImageType": "machine",
            "Public": true,
            "OwnerId": "137112412989",
            "PlatformDetails": "Linux/UNIX",
            "UsageOperation": "RunInstances",
            "State": "available",
            "BlockDeviceMappings": [
                {
                
```

## EC2 Creation

We provide the AMI ID, Security group id, Subnet id and the keypair name.

```sh

aws ec2 run-instances --image-id ami-0ed72083dbed1d548 --count 1 --instance-type t2.micro 
--key-name cli-web-keypair 
--security-group-ids sg-08e4bfd92f793c6a5 --subnet-id subnet-00c0a8f7de82f11a8

```

We can list our created EC2 instance with the following command

```sh

aws ec2 describe-instances

```

We now have to create an additional storage of 2GB. We have to create the additional EBS volume in the same availability zone. The following command will provide details of the availability zone.

```sh

aws ec2 describe-availability-zones

```

We now create the additional EBS volume in the same availabilitz zone of the EC2 instance.

```sh

aws ec2 create-volume --region us-east-2 --availability-zone us-east-2b --size 2 --volume-type gp2

```

Output

```sh

{
    "AvailabilityZone": "us-east-2b",
    "CreateTime": "2021-08-19T07:21:56.000Z",
    "Encrypted": false,
    "Size": 2,
    "SnapshotId": "",
    "State": "creating",
    "VolumeId": "vol-027b77beb837467a8",
    "Iops": 100,
    "Tags": [],
    "VolumeType": "gp2",
    "MultiAttachEnabled": false
}

```

We get the volume id when creating the additional volume. For attaching our volume we provide the following line.

```sh

aws ec2 attach-volume --volume-id vol-027b77beb837467a8 
--instance-id i-049e0957acea10d47 --device /dev/sdf

```
Output

```sh

{
    "AttachTime": "2021-08-19T07:28:49.947Z",
    "Device": "/dev/sdf",
    "InstanceId": "i-049e0957acea10d47",
    "State": "attaching",
    "VolumeId": "vol-027b77beb837467a8"
}

```

In order to SSH to our EC2 Instance we need to get the Public IP, we can provide the following command to fetch the public IP of our instance.

```sh

aws ec2 describe-instances --instance-ids i-049e0957acea10d47 
--query 'Reservations[0].Instances[0].PublicIpAddress'

```

Finally, we can connect to our EC2 instance.

```sh

ssh -i cli-web-keypair.pem ec2-user@YOUR_PUBLIC_IP

```

## Package Installation

```sh

yum install httpd -y
systemctl restart httpd
systemctl enable httpd

```

## Creating Partition and Mounting it

```sh

# fdisk /dev/xvdf
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (1-4, default 1):
First sector (2048-4194303, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-4194303, default 4194303):

Created a new partition 1 of type 'Linux' and of size 2 GiB.

Command (m for help): w

```

We have created the partition now we format it with a filesystem and mount the partition.

```sh

mkfs -t xfs /dev/xvdf1
mount /dev/xvdf1 /var/www/html
df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvdf1      2.0G  6.0M  1.9G   1% /var/www/html

```

We can also add the necessary entries in the /etc/fstab file so the mount point persists even after a reboot.

```sh

echo "/dev/xvdf1      /var/www/html   xfs defaults   0       2" >> /etc/fstab

```
Now we add the web files to the document root, apply necessary permissions and map the domain name to our public Ip.

## For Stopping our EC2 Instance

```sh
aws ec2 stop-instances --instance-ids i-049e0957acea10d47
```

## For Terminating our EC2 Instance

```sh
aws ec2 terminate-instances --instance-ids i-049e0957acea10d47
```
## Conclusion

We completed the project by using aws-cli commands and aws resources to achieve our goals.
