## EC2 Creation From AWSCLI




## Prerequisites for this project

- Needs AWS CLI access preferrably a user with admin access
- AWS CLI needs to be installed on your system
- Knowledge of AWS CLI commands

## Resources
- EC2
- Security Group
- AMI (Amazon Machine Image)
- Key-Pair
- Additional Volume

## Configure the AWS CLI User

```sh
#aws configure
# Provide AWS Access Key ID
# Provide AWS Secret Access Key
# Provide Default region name
# Provide Default output format
```

## Creating Security Group

 A security group is a virtual firewall. we can set rules to limit access to our instance.
 
```sh
aws ec2 create-security-group --group-name cli-securitygroup --description "22,80,443-Open"
```

Group ID is shows as output, we need the group id inorder to create our instance.

```sh

{
    "GroupId": "sg-0fd9098f110fa4e13"
}

```
Inorder to access our EC2 instance via SSH we are opening port 22 and also ports 80 & 443 as we need to access the webserver.

```sh
aws ec2 authorize-security-group-ingress --group-name cli-securitygroup --protocol tcp 
--port 22 --cidr 0.0.0.0/0
```

```sh
aws ec2 authorize-security-group-ingress --group-name cli-securitygroup --protocol tcp 
--port 80 --cidr 0.0.0.0/0
```

```sh
aws ec2 authorize-security-group-ingress --group-name cli-securitygroup --protocol tcp 
--port 443 --cidr 0.0.0.0/0
```

## Key-Pair

The proceeding line creates a 2048-bit RSA key pair. The aws ec2 command stores the public key and outputs the private key to save to a file.

```sh
aws ec2 create-key-pair --key-name cli-keypair --query "KeyMaterial" --output text > clikeypair-key.pem
```

## AMI (Amazon Machine Image)

When creating a EC2 instance from the command line, we specify the operating system using the amazon machine image (AMI) ID. To get the image ID we use the following command.

```sh

aws ec2 describe-images --owners amazon --filters "Name=name,Values=amzn2-ami-hvm-2.0.????????-x86_64-gp2" "Name=state,Values=available" --output json

```

From the output we get the AMI ID
```sh
 {
            "Architecture": "x86_64",
            "CreationDate": "2019-06-19T21:59:18.000Z",
            "ImageId": "ami-0d2692b6acea72ee6",
            "ImageLocation": "amazon/amzn2-ami-hvm-2.0.20190618-x86_64-gp2",
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

We provide the AMI ID, Security group id and the keypair name.
```sh

aws ec2 run-instances --image-id ami-0d2692b6acea72ee6 --security-group-ids sg-0fd9098f110fa4e13 --instance-type t2.micro --key-name cli-keypair

```

we can list our created EC2 instance with the following command
```sh

aws ec2 describe-instances

```

## Additional Volume creation & Attaching

we create an additional volume with a size of 2GB.
```sh

aws ec2 create-volume --size 2 --availability-zone ap-south-1b

```

We get the volume id when creating the additional volume.For attaching our volume we provide the following line
```sh

aws ec2 attach-volume --volume-id vol-0d264f40bf5f33044 --instance-id -0e51d766d76c82eab --device /dev/sdf

```

In order to SSH to our EC2 Instance we need to get the Public IP, we can provide the following command to fetch the public IP of our instance

```sh

aws ec2 describe-instances --instance-ids i-0e51d766d76c82eab --query "Reservations[0].Instances[0].PublicIpAddress"

```

Finally, we can connect to our EC2 instance
```sh

ssh -i clikeypair-key.pem ec2-user@YOUR_PUBLIC_IP

```
## Package Installation

```sh

#yum install httpd -y
#systemctl restart httpd
#systemctl enable httpd

```

## Creating Partition and Mounting it.

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

#mkfs -t xfs /dev/xvdf1

#mount /dev/xvdf1 /var/www/html

```
We can also add the necessary entries in the /etc/fstab file so the mount point persists even after a reboot.

## Fetching a Sample Website & Giving Necessary Permissions

```sh

#wget https://www.tooplate.com/zip-templates/2101_insertion.zip
#unzip 2101_insertion.zip
#cp -r 2101_insertion/* /var/www/html/
#chown -R apache.apache /var/www/html/

```

We now point our domain name to our public ip and load the site.

## Stopping our EC2 instance

```sh

aws ec2 stop-instances --instance-ids i-0e51d766d76c82eab

```
## Terminating our EC2 instance

```sh

aws ec2 terminate-instances --instance-ids i-0e51d766d76c82eab

```
