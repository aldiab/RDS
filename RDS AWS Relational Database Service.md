---
tags:
  - aws
---
# RDS AWS Relational Database Service



Learning outcomes

- Look at features of RDS, What and why

- Create an RDS instance using a CloudFormation template

- Connect to an RDS instance from an EC2 instance

- Create a backup using `mysqldump`

- Create a manual snapshot with AWS-CLI

- Create automated snapshot

- Restore from a snapshot

- Look at a few RDS features that can be used to create a production ready DB



## AWS Database services



### RDS

AWS RDS includes several popular relational databases, including MySQL. AWS also offer their own Aurora database, which includes MySQL and PostreSQL compatibility options.

Today we are going to look at setting up a MySQL database on RDS.



**Resources:**

- [Amazon RDS - Info](https://aws.amazon.com/rds/)

- [RDS Docs](https://docs.aws.amazon.com/rds/index.html)



![aws-rds-services.png](./RDS%20AWS%20Relational%20Database%20Service-assets/aws-rds-services.png)

## Why use a managed database?



In assignment 1 you created a database by installing MySQL on an EC2 instance.

However, you weren’t going to maintain the database for an extended period of time. We don’t need to worry about migrating to a new database, or making sure that our database is always available, or what happens when our organization grows, and we need to store more data.

Performing all of these tasks overtime is a lot of work, and mistakes can (**will**) happen. If you break some of your application code, some users may not be able to use your application for a short period of time. This is inconvenient, but not catastrophic. However, if you break your database customers may lose important data, passwords, all of their photos… They probably won’t be customers after this.

As we look at some of the features offered by AWS RDS, we will look at some of the ways in which a managed database can help us avoid things like accidentally losing all of our customers photos.

## RDS Attributes

Required Attributes for an RDS instance. Using the AWS in Action as a reference, complete the table below.




| Attribute | Description | 
|---|---|
| `AllocatedStorage` |  | 
| `DBInstanceClass` |  | 
| Engine |  | 
| DBName |  | 
| `MasterUsername` |  | 
| `MasterUserPassword` |  | 

## Create an RDS instance using an AWS CloudFormation template.



Use the below example CloudFormation template to create a public EC2 instance and a private RDS Instance. 



We are going to use the web console to build a CloudFormation stack today, but you can use the AWS-CLI for this as well.



The template below will create a stack that contains:

- an ec2 instance running Ubuntu, with the ‘mysql-client’ package installed

- A new SSH key ‘fortytwo’

- An RDS instance

   - db engine MySQL

   - User ‘fortytwo’

   - pass ‘fortytwo’



```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'RDS example for 3640'

Parameters:

  LatestAmiId:
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: '/aws/service/canonical/ubuntu/server/lunar/stable/current/amd64/hvm/ebs-gp2/ami-id'

  VpcCidrBlock:
    Description: CIDR block for the VPC
    Type: String
    Default: 10.0.0.0/16

Resources:
  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidrBlock
      EnableDnsSupport: true
      EnableDnsHostnames: true

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      AvailabilityZone: us-west-2a
      CidrBlock: 10.0.1.0/24
      MapPublicIpOnLaunch: true

  RDSSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      AvailabilityZone: us-west-2a
      CidrBlock: 10.0.2.0/24

  RDSSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      AvailabilityZone: us-west-2b
      CidrBlock: 10.0.3.0/24

  InternetGateway:
    Type: AWS::EC2::InternetGateway

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref InternetGateway

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC

  DefaultRoute:
    Type: AWS::EC2::Route
    DependsOn: AttachGateway
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PublicSubnet
      RouteTableId: !Ref PublicRouteTable

  KeyPair:
    Type: AWS::EC2::KeyPair
    Properties:
      KeyName: fortytwo
      KeyType: ed25519

  PublicInstance:
    Type: AWS::EC2::Instance
    Properties:
      SubnetId: !Ref PublicSubnet
      KeyName: !Ref KeyPair
      ImageId: !Ref LatestAmiId
      InstanceType: t2.micro
      SecurityGroupIds: 
        - !Ref PublicSecurityGroup
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          apt update
          apt install mysql-client -y

  PublicSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable SSH access from any IP
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0

  DatabaseSecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupDescription: allow port 3306
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 3306
        ToPort: 3306
        SourceSecurityGroupId: !Ref PublicSecurityGroup

  DatabaseSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: Subnet group for RDS database
      SubnetIds:
        - !Ref RDSSubnet1
        - !Ref RDSSubnet2

  RDSInstance:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Delete
    Properties:
      Engine: MySQL
      DBInstanceIdentifier: rds-database-instance
      MasterUsername: fortytwo
      MasterUserPassword: fortytwo
      DBInstanceClass: db.t2.micro
      AllocatedStorage: 5
      BackupRetentionPeriod: 0
      DBSubnetGroupName: !Ref DatabaseSubnetGroup
      VPCSecurityGroups:
        - !Ref DatabaseSecurityGroup
      DBName: fortytwo

Outputs:

  VPCId:
    Description: VPC ID
    Value: !Ref MyVPC

  PublicSubnetId:
    Description: Public Subnet ID
    Value: !Ref PublicSubnet

  PublicInstanceId:
    Description: ID of the Public EC2 Instance
    Value: !Ref PublicInstance
```



**Resources:**

- [AWS Docs](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-rds-dbinstance.html)

## Connect to an RDS instance from an EC2 instance



Using the infrastructure created with the CloudFormation template above you are going to connect to your RDS instance from your EC2 instance using the mysql-client.



Before you can do this you need a few things:

- Your database host address

- Your ec2 instances public IP address

- The private key in a .pem file

### Retrieve the RDS connection string with AWS-CLI 

You want the value of the “Address: “ key

```bash
aws rds describe-db-instances --query "DBInstances[0].Endpoint"
```

### Retrieve the EC2 public IP with AWS-CLI

The command below will display public IP addresses and IDs for all EC2 instances in the region specified in your `~/.aws/config`

```bash
aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId, PublicIpAddress]' --output table
```

### Retrieve pem file for SSH connection with AWS-CLI

First get the key ID

```bash
aws ec2 describe-key-pairs --filters Name=key-name,Values=fortytwo --query KeyPairs[*].KeyPairId --output text 
```



After output the private key file to a local pem file. You will need to replace ‘key-EXAMPLE’ with the id output from the previous command.

```bash
aws ssm get-parameter --name /ec2/keypair/key-EXAMPLE --with-decryption --query Parameter.Value --output text > rds-key.pem
```



Using the EC2 instance IP address and .pem file connect to your EC2 instance.



Finally you can connect to your database

Replace the database host with your own database host string. This will prompt you for a password. The password is 'fortytwo‘.

```bash
mysql -h rds-database-instance.c2yvkst9lxf9.us-west-2.rds.amazonaws.com -u fortytwo -p
```



After connecting to your database you can perform regular DB admin tasks using the mysql client.

## Creating a backup with mysqldump



First let’s create something to backup.



Connect to your database.



Switch to database fortytwo.



```sql
USE fortytwo;
```



Create a table for 



```sql
CREATE TABLE dr_who_companions
(
  id              INT unsigned NOT NULL AUTO_INCREMENT, # Unique ID for the record
  name            VARCHAR(150) NOT NULL,                # Name of the companion
  first_appeard   DATE NOT NULL,                        # Date of first appearance
  PRIMARY KEY     (id)                                  # Make the id the primary key
);

```



Populate the table with a few of the dr’s companions.



```sql
INSERT INTO dr_who_companions ( name, first_appeard) VALUES
  ( 'Ruby Sunday', '2023-12-25' ),
  ( 'Amy Pond', '2010-04-03' ),
  ( 'Yasmin Khan', '2018-10-07' );

```



Check that everything succeeded so far



```sql
SELECT * FROM dr_who_companions;
```



exit from your database connection with the `exit` command and create the backup.



Don’t forget to use your host string instead of the example in the command below.



```sql
mysqldump -h rds-database-instance.c2yvkst9lxf9.us-west-2.rds.amazonaws.com -u fo
rtytwo -p fortytwo > fortytwo.sql
```



after this you can transfer your backup to your host with `sftp`



**References:**

- [MySQL Docs, Getting Started](https://dev.mysql.com/doc/mysql-getting-started/en/#mysql-getting-started-connecting)

- [MySQL Docs, mysqldump](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html)

## Backing up and restoring an RDS database from a snapshot



> Creating a snapshot requires all disk activity to be briefly frozen. Requests to the database may be delayed or even fail because of a timeout, so we recommend that you choose a time frame for the snapshot that has the least effect on applications and users (e.g., late at night).
>
> - AWS in Action 3rd



Create a manual snapshot

```bash
aws rds describe-db-instances --output text \
  --query "DBInstances[0].DBInstanceIdentifier"
```



```bash
aws rds create-db-snapshot --db-snapshot-identifier \
  db-snapshot \
  --db-instance-identifier $DBInstanceIdentifier
```



It will take a few minutes to create the snapshot, check status with this command

```bash
aws rds describe-db-snapshots \
  --db-snapshot-identifier db-snapshot
```



When you see `Status: available` in the output your snapshot is ready.



### Restore from the snapshot you just created



First get the subnet 



```bash
aws cloudformation describe-stack-resource \
 --stack-name <your-stack-name> --logical-resource-id DatabaseSubnetGroup \
 --query "StackResourceDetail.PhysicalResourceId" --output text
```



```bash
aws rds restore-db-instance-from-db-snapshot \
 --db-instance-identifier rds-database-instance \
 --db-snapshot-identifier db-snapshot \
 --db-subnet-group-name <output-of-above>
```



RDS does not delete manual snapshots, you need to manually delete them!



```bash
aws rds delete-db-snapshot \
  --db-snapshot-identifier db-snapshot
```



## Setting up automatic snapshots



If you were to update the original CloudFormation template with the following lines.



```yaml
      BackupRetentionPeriod: 3
      PreferredBackupWindow: '01:00-02:00'
```



A snapshot of your database would automatically be created between 01:00-02:00 and be saved for 3 days.



The original template has the backup retention period set to “0” which turns off automatic snapshots.



Example of updated template, rds instance resource.

```yaml
  RDSInstance:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Delete
    Properties:
      Engine: MySQL
      DBInstanceIdentifier: rds-database-instance
      MasterUsername: fortytwo
      MasterUserPassword: fortytwo
      DBInstanceClass: db.t2.micro
      AllocatedStorage: 5
      BackupRetentionPeriod: 3
      PreferredBackupWindow: '01:00-02:00'
      DBSubnetGroupName: !Ref DatabaseSubnetGroup
      VPCSecurityGroups:
        - !Ref DatabaseSecurityGroup
      DBName: fortytwo
```

## Cost of RDS snapshots

- snapshots are billed based on storage they use. You can store snapshots up to the size of your database for free. On top of that you pay for GB



## Controlling access to a database



![10-04.png](./RDS%20AWS%20Relational%20Database%20Service-assets/10-04.png)



IAM policies define who can perform actions like, creating, updating and deleting an RDS instance. 



For example you could create an IAM policy that would prevent certain users or groups of users from deleting or removing a database



```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "rds:*",           
    "Resource": "*"                              
  }, {
    "Effect": "Deny",                            
    "Action": ["rds:Delete*", "rds:Remove*"],    
    "Resource": "*"                              
  }]
}
```



> IAM doesn’t manage access inside the database; that’s the job of the database engine. IAM policies define which configuration and management actions an identity is allowed to execute on RDS.



### Controlling network access to an RDS database



The example template provided includes security groups for the EC2 instance and the RDS instances



```yaml
  PublicSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable SSH access from any IP
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0

  DatabaseSecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupDescription: allow port 3306
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 3306
        ToPort: 3306
        SourceSecurityGroupId: !Ref PublicSecurityGroup
```



The last line specifies that only resources behind the “PublicSecurityGroup” can connect to the RDS instance using port 3306



## A few tweaks that can be made to create a production ready RDS instance.



RDS is relatively expensive, so we are just going to look at how you can do these things, to save a little $$.



### Failover



![10-05.png](./RDS%20AWS%20Relational%20Database%20Service-assets/10-05.png)



Failover is when the Primary or Master database in a cluster fails, and a secondary database becomes the primary.

“Normally, with relational databases, only one database can hold the *master* status, meaning that data can be both written to and read from it.”

> If the master copy of your database fails, then AWS will perform a failover operation to the standby copy of the database. The standby copy will be promoted to become the new master, and the previous master will be terminated and replaced with another standby copy.

- AWS Certified Cloud Practitioner Exam Guide

> 

An AWS RDS database can be setup to run in multiple availability zones within a region. If one zone goes down, failover occurs and a database in another AZ becomes master. This will likely result in a small amount of downtime, but data shouldn’t be lost.



### Horizontal scaling



AWS RDS also offers a mechanism to scale your database horizontally. Historically, this isn’t something that relational databases do well. RDS achieves this using something called r*ead replicas*. Asynchronous replication is used to create a copy of your data on a database that can handle read requests. This allows your master copy to handle requests that write to your database.

Vertical scaling is increasing resources (more powerful servers).

Horizontal scaling is increasing the number of servers.

[Horizontal Vs. Vertical Scaling Comparison Guide](https://www.mongodb.com/basics/horizontal-vs-vertical-scaling)



### Enable a highly available database

To enable a highly available database you only have to edit one line in the template example, `MultiAZ: true`



```yaml
  RDSInstance:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Delete
    Properties:
      Engine: MySQL
      DBInstanceIdentifier: rds-database-instance
      MasterUsername: fortytwo
      MasterUserPassword: fortytwo
      DBInstanceClass: db.t2.micro
      AllocatedStorage: 5
      BackupRetentionPeriod: 0
      DBSubnetGroupName: !Ref DatabaseSubnetGroup
      VPCSecurityGroups:
        - !Ref DatabaseSecurityGroup
      DBName: fortytwo
      MultiAZ: true
```



In addition to handling failover, a multi AZ database will switch to the secondary during many maintenance activities, which also helps to prevent down time.



## Increase database resources



To increase database resources we can change two lines in the original configuration



```yaml
      DBInstanceClass: db.m3.large # increase to faster cores and more memory
      AllocatedStorage: 200 # increases storage to 200 GB
```

## Read replication



![10-07.png](./RDS%20AWS%20Relational%20Database%20Service-assets/10-07.png)



> A database suffering from too many read requests can be scaled horizontally by adding additional database instances for read traffic and enabling replication from the primary (writable) copy of the database instance. As figure 10.7 shows, changes to the database are asynchronously replicated to an additional read-only database instance. The read requests can be distributed between the primary database and its read-replication databases to increase read throughput. Be aware that you need to implement the distinction between read and write requests on the application level.
>
> [AWS in Action 3rd](https://learning.oreilly.com/library/view/amazon-web-services/9781633439160/OEBPS/Text/10.htm#:-:text=10.6.2%20Using%20read%20replication%20to%20increase%20read%20performance)



To create a read replication database you can use the AWS-CLI



```bash
aws rds create-db-instance-read-replica \
 --db-instance-identifier awsinaction-db-read \
 --source-db-instance-identifier $DBInstanceIdentifier
```



RDS automatically triggers the following steps in the background:

- Creating a snapshot from the source database, also called the primary database instance

- Launching a new database based on that snapshot

- Activating replication between the primary and read-replication database instances

- Creating an endpoint for SQL read requests to the read-replication database instances



After create a read replication endpoint you can handle read requests with this new endpoint.

## Monitoring a database



Complete the following table using the textbook as a reference



Important metrics to monitor

| Name | Description | 
|---|---|
| FreeStorageSpace |  | 
| CPUUtilization |  | 
| FreeableMemory |  | 
| DiskQueueDepth |  | 
| SwapUsage |  | 

## Cleanup

Don’t forget to delete the stack you created!



**Resources:**



<https://aws.amazon.com/rds/pricing/>