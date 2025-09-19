# Using AWS Cloudformation to automate the setup of a 3-tier WebApp with Amazon RDS and High Availability
## Overview

This project offers a solution to a specific scenario, deploying a WebApp in a private VPC with Load balancing and an autoscaling group connected to a RDS Aurora cluster . With Aws CloudFormation, it was possible to automate the creation and provision of AWS infrastructure deployments, using template files to create or delete resources in a single unit as a stack.

Once the stack is created, we can access to the WebApp and test the load balancer, the connection with the RDS database, and the autoscaling group adding load to the current cpu instances.

To meet the goal of this project, it has been divided into 3 sections. The first one to create an AMI with the web application files and an Key pair to connect to the instances. The second is in charge of designing a template to create our stack in CloudFormation. Finally, the third section allow us to test the website capabilities.

#### Initial setup
+ Step 1. **Creating a key pair.**
+ Step 2. **Creating an AMI.**
#### CloudFormation template 
+ Step 3. **Creating a VPC.**
+ Step 4. **Creating an Application Load Balancer.**
+ Step 5. **Creating an Autoscaling Group.**
+ Step 6. **Creating an Aurora Database Cluster.**
+ Step 7. **Creating a Secret.**
+ Step 8. **Creating the stack with Cloudformation.**
#### Final steps
+ Step 9. **Testing the autoscaling group.**
+ Step 10. **Connecting to RDS.**

As you can see, most of the steps will be automated in an AWS CloudFormation template, which will be explained throughout this document. The goal is automate the process of deploying the resources used to host the solution.

<br>

## Architecture

The following image provides a view of the proposed architecture for deploying website resources.

![Image description](https://github.com/Sjleal/aws-3tier-ha-sc-webapp/blob/main/images/diagram/aws-3tier-ha-sc-webapp.png)

#### The User Experience
The client only needs to know a single public address, which belongs to the Application Load Balancer (ALB). Every interaction with the application begins there, ensuring simplicity and transparency.

The ALB distributes requests across all available instances in a fault-tolerant way. The client doesn’t notice which server is responding. Responses are always consistent.

If an instance fails, the Auto Scaling Group automatically provisions a new one in another AZ. The client never experiences downtime—the service remains continuous and reliable.

When user traffic grows, the infrastructure automatically scales out horizontally. From the client’s perspective, the application continues to respond smoothly, regardless of the number of concurrent users.

The application interacts with a managed Aurora database that provides replication and high availability. For the client, this means data is always accessible and consistent, even during high demand or replica failures

From the client’s point of view, connecting to a single URL gives access to a highly available and scalable service. Behind the scenes, AWS infrastructure automatically adapts to failures and traffic spikes, ensuring a fast, reliable, and seamless user experience.

<br>

#### The Architect’s perspective
The entire solution is deployed using AWS CloudFormation. This guarantees infrastructure as code, repeatability, and consistency across environments.

The stack is launched from a single YAML template. The template defines networking, compute, scaling, and database resources in a standardized manner. Updates and modifications can be applied via change sets, reducing the risk of configuration drift.

When launching the stack, the maintainer provides only a minimal set of parameters:
- Key Pair → Enables secure SSH access to EC2 instances (if needed).
- Custom AMI → Pre-baked image containing the web application and necessary runtime environment, ensuring faster bootstrapping.
- Database Inputs → Connection strings or credentials required to configure the Aurora cluster and allow the application layer to connect.

This design makes the deployment flexible, but also controlled and predictable. The CloudFormation template configures:

- A VPC with public and private subnets across multiple Availability Zones for fault tolerance.
- An Internet-facing ALB distributing requests to private subnets where the Auto Scaling Group resides.
- Auto Scaling Group policies to automatically scale in/out based on load.
- Aurora cluster with replicas to maintain high availability and durability.

As a maintainer, I don’t need to manually provision or intervene when traffic grows or instances fail — the infrastructure adapts automatically. Instances remain in private subnets; only the ALB is exposed publicly. Security groups and IAM roles enforce least privilege. Updating the AMI with a new application version or applying database schema changes can be rolled out through stack updates, ensuring controlled lifecycle management.

From the architect’s perspective, the solution is not only designed for high availability and scalability, but also for operational efficiency, automation, and maintainability. CloudFormation provides a consistent, reproducible deployment model that reduces manual work and risk, while allowing the system to evolve as business and application needs grow.
 
<br>

## Services

- [Amazon VPC](https://docs.aws.amazon.com/vpc/) enables you to provision a logically isolated section of the AWS Cloud where you can launch AWS resources in a virtual network that you've defined.
- [AWS Identity and Access Management (IAM)](https://docs.aws.amazon.com/iam/) helps you securely manage access to your AWS resources by controlling who is authenticated and authorized to use them.
- [Amazon Elastic Compute Cloud (EC2)](https://docs.aws.amazon.com/ec2/) is a web service that provides resizable computing capacity—literally, servers in Amazon's data centers—that you use to build and host your software systems.
- [Amazon Relational Database Service (Amazon RDS)](https://docs.aws.amazon.com/rds/) is a web service that makes it easier to set up, operate, and scale a relational database in the cloud. It provides cost-efficient, resizeable capacity for an industry-standard relational database and manages common database administration tasks. Amazon Aurora is a fully managed relational database engine that's built for the cloud and compatible with MySQL and PostgreSQL. 
- [Amazon Secrets Manager](https://docs.aws.amazon.com/secretsmanager/) helps you to securely encrypt, store, and retrieve credentials for your databases and other services. Instead of hardcoding credentials in your apps, you can make calls to Secrets Manager to retrieve your credentials whenever needed.
- [AWS CloudFormation](https://docs.aws.amazon.com/cloudformation/) helps you set up AWS resources, provision them quickly and consistently, and manage them throughout their lifecycle. You can use a template to describe your resources and their dependencies, and launch and configure them together as a stack, instead of managing resources individually. You can manage and provision stacks across multiple AWS accounts and AWS Regions.


<br>

## Step by Step

>**Initial setup:**<br>
In these initial steps, we will focus on preparing the ec2 instance with the webapp, creating the AMI, and the key pair.

**1. Creating a key pair**

A key pair is needed when creating an EC2 Launch Template, in order to streamlines the process of launching instances, as you don't need to manually select a key pair each time.

To create a [Key pair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/create-key-pairs.html) you need to follow these steps:
1. Logging in to the AWS Management console and open the EC2 console.
2. Choose Key Pairs under Network & Security in the left column and click on Create key pair to start the Create key pair wizard.
   1. __Name:__ enter a descriptive name for the key pair.
   2. __Key Pair type:__ choose RSA.
   3. __Private key file format:__ select .pem for use Openssh.
   4. __Tags - optional:__ you can assign a tag for this resource.
3. Click on __Create key pair__ the file is automatically downloaded by your browser.

**2. Creating an AMI**

A custom AMI ensures that scaling events are fast, reliable, secure, and predictable. New instances launch with the application and dependencies already installed, reducing bootstrap time. Every instance is identical, avoiding issues from failed or changing installation scripts. The AMI includes pre-hardened OS settings, updates, and monitoring tools, ensuring compliance by default.

Before creating the custom AMI, you need a base instance that contains the web application, its dependencies, and the necessary configuration to connect to the database and simulate load. For this purpose, we used the sample application from the [AWS Immersion Day Advanced](https://catalog.us-east-1.prod.workshops.aws/workshops/869a0a06-1f98-4e19-b5ac-cbb1abdfc041/en-US/advanced-modules/compute/launching#create-a-custom-ami), which provides a ready-to-use setup.

The base instance should be launched from an Amazon Linux AMI, within the default VPC, and secured with a security group that allows only SSH and HTTP access. A key pair is not required for this setup. Additionally, it is recommended to enable Instance Metadata Service v2 (IMDSv2) and include the following initialization script in the user data section:

~~~
#!/bin/sh
​
#Install a LAMP stack
dnf install -y httpd wget php-fpm php-mysqli php-json php php-devel
dnf install -y mariadb105-server
dnf install -y httpd php-mbstring
​
#Start the web server
chkconfig httpd on
systemctl start httpd
​
#Install the web pages for our lab
if [ ! -f /var/www/html/immersion-day-app-php7.zip ]; then
   cd /var/www/html
   wget -O 'immersion-day-app-php7.zip' 'https://static.us-east-1.prod.workshops.aws/public/93678d5d-ac27-459f-a7f2-088ae25e5522/assets/immersion-day-app-php7.zip'
   unzip immersion-day-app-php7.zip
fi
​
#Install the AWS SDK for PHP
if [ ! -f /var/www/html/aws.zip ]; then
   cd /var/www/html
   mkdir vendor
   cd vendor
   wget https://docs.aws.amazon.com/aws-sdk-php/v3/download/aws.zip
   unzip aws.zip
fi
​
# Update existing packages
dnf update -y
~~~

Once the base instance is properly configured and running the application, you can generate a custom AMI from it. This ensures that every new instance launched by the Auto Scaling Group will already contain the application, dependencies, and configurations required to operate.

To create an [AMI from an Instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/creating-an-ami-ebs.html) you need to follow these steps:
1. Logging in to the AWS Management console and open the EC2 console.
2. Choose Instances under Instances in the left column and select the instance.
3. Choose Actions, Image and templates, Create image.
   1. __Name:__ enter a descriptive and unique name for the AMI.
   2. __Image description:__ enter an optional description of the image.
   3. __Reboot:__ clear the checkbox.
   4. __Instance volumes:__ select __Delete on termination__.
   5. __Tags:__ select __Tag image and snapshots together__.
4. When you're ready to create your AMI, choose Create image.
5. When the process is fully completed (AMI with status Available), you can terminate de base instance.


<br>

---


>**CloudFormation Template:**<br>
Infrastructure as Code (Iac) is the best way to maintain infrastructure integrity, reduce errors, speed up deployments, troubleshoot issues, and track changes over time. Taking advantage of Cloudformation's ability to automate resource deployment, a template will be designed to handle the creation and configuration of the resources involved in the solution.<br>
In the following steps, part of the code used in each section will be shown, the complete template is available in a public repository called [3tier-ha-sc-webapp](https://github.com/Sjleal/aws-3tier-ha-sc-webapp/blob/main/dev/3tier-ha-sc-webapp.yaml).<br>
Some variables will be requested at the stack creation and others will be captured during template execution. The YAML format was chosen for this template.


**3. Creating a VPC**

The Virtual Private Cloud (VPC) is the foundation of this 3-tier solution, providing logical network isolation, segmentation, and secure communication between layers. By carefully structuring the VPC, we ensure that each component of the architecture is deployed in the right place with the appropriate level of exposure and control.

Within the VPC, the following components are integrated:
- __Public Subnets:__ Host internet-facing resources such as the Application Load Balancer (ALB).
- __Private Subnets:__ Host the Auto Scaling Group running the web application, as well as the Aurora database cluster, ensuring that the core application logic and data remain inaccessible from the internet.
- __Internet Gateway (IGW):__ Provides outbound connectivity for public resources.
- __Public and Private Route Tables:__ Control the flow of traffic between subnets and external networks.
- __NAT Gateway:__ Allows instances in private subnets to initiate outbound connections (for example, downloading updates or packages) without being directly exposed to the internet.

For this Proof of Concept (PoC), only one NAT Gateway was deployed in a single Availability Zone to reduce costs. While this is sufficient to demonstrate the solution, a fully highly available design would require deploying one NAT Gateway per Availability Zone to eliminate any single points of failure. The following code was used to create and configure the resources via cloudformation template:

~~~
Resources:
  MainVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      InstanceTenancy: default
      Tags:
        - Key: Name
          Value: !Ref punitag
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Ref punitag
  VPCGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref MainVPC
  MainSubnetPublic:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: us-east-1a
      CidrBlock: 10.0.10.0/24
      VpcId: !Ref MainVPC
      Tags:
        - Key: Name
          Value: !Ref punitag
  MainSubnetPrivate:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: us-east-1a
      CidrBlock: 10.0.100.0/24
      VpcId: !Ref MainVPC
      Tags:
        - Key: Name
          Value: !Ref punitag
  SecondSubnetPublic:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: us-east-1b
      CidrBlock: 10.0.20.0/24
      VpcId: !Ref MainVPC
      Tags:
        - Key: Name
          Value: !Ref punitag
  SecondSubnetPrivate:
    Type: AWS::EC2::Subnet
    Properties:
      AvailabilityZone: us-east-1b
      CidrBlock: 10.0.200.0/24
      VpcId: !Ref MainVPC
      Tags:
        - Key: Name
          Value: !Ref punitag
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MainVPC
      Tags:
        - Key: Name
          Value: !Ref punitag
  RouteIGW:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
  SubnetMainPubRTA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref MainSubnetPublic
      RouteTableId: !Ref PublicRouteTable
  SubnetSecondPubRTA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref SecondSubnetPublic
      RouteTableId: !Ref PublicRouteTable
  NATGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NATGatewayEIP.AllocationId
      SubnetId: !Ref MainSubnetPublic
      Tags:
        - Key: Name
          Value: !Ref punitag
  NATGatewayEIP:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc
      Tags:
        - Key: Name
          Value: !Ref punitag
  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MainVPC
      Tags:
        - Key: Name
          Value: !Ref punitag
  RouteNATGateway:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NATGateway
  SubnetMainPrivRTA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref MainSubnetPrivate
      RouteTableId: !Ref PrivateRouteTable
  SubnetSecondPrivRTA:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref SecondSubnetPrivate
      RouteTableId: !Ref PrivateRouteTable
~~~


<br>

**4. Creating an Application Load Balancer**

The Application Load Balancer (ALB) serves as the single entry point for client traffic into the solution. It is deployed in the public subnets of the VPC and distributes incoming HTTP/HTTPS requests across the web application instances running in private subnets.

By integrating the ALB into the architecture, we achieve:
- Traffic Distribution: Requests are intelligently routed across multiple instances in the Auto Scaling Group.
- High Availability: By spanning multiple Availability Zones, the ALB ensures that client requests continue to be served even if one AZ becomes unavailable.
- Security: The ALB is the only internet-facing component; application servers and databases remain in private subnets. Security group and listener control what kind of traffic is accepted and where it is forwarded.

In the CloudFormation template, this section will provision:
- The ALB itself, attached to public subnets.
- Listeners (HTTP/HTTPS) that define how incoming requests are processed.
- Target Groups where the Auto Scaling Group instances will register.
- Security group to restrict and control inbound and outbound traffic.

This configuration ensures that the ALB not only provides an internet-facing endpoint but also enforces resilience, scalability, and secure access to the application tier. The following code was used to create and configure the resource via cloudformation template:

~~~
Resources:
  WebALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable HTTP access via port 80
      VpcId: !Ref MainVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
      Tags:
        - Key: Name
          Value: !Ref punitag
  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: Web-ALB
      Subnets:
        - !Ref MainSubnetPublic
        - !Ref SecondSubnetPublic
      SecurityGroups:
        - !Ref WebALBSecurityGroup
      Scheme: internet-facing
      Tags:
        - Key: Name
          Value: !Ref punitag
      Type: application
  ApplicationLBTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      TargetType: instance
      HealthCheckEnabled: true
      HealthCheckIntervalSeconds: 10
      HealthCheckPath: /
      HealthCheckProtocol: HTTP
      HealthCheckTimeoutSeconds: 8
      HealthyThresholdCount: 2
      Name: WebTargetGroup
      Port: 80
      Protocol: HTTP
      ProtocolVersion: HTTP1
      UnhealthyThresholdCount: 2
      VpcId: !Ref MainVPC
      Tags:
        - Key: Name
          Value: !Ref punitag
  ApplicationLBListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref ApplicationLBTargetGroup
      LoadBalancerArn: !Ref ApplicationLoadBalancer
      Port: 80
      Protocol: HTTP
  ApplicationLBListenerRule:
    Type: AWS::ElasticLoadBalancingV2::ListenerRule
    Properties:
      Actions:
        - Type: forward
          TargetGroupArn: !Ref ApplicationLBTargetGroup
      Conditions:
        - Field: path-pattern
          Values:
            - /
      ListenerArn: !Ref ApplicationLBListener
      Priority: 1
~~~


<br>

**5. Creating an Autoscaling Group**

This section of the template provisions the Auto Scaling Group (ASG) responsible for hosting the web application instances in private subnets. The configuration ensures elasticity, availability, and secure integration with other AWS services.

The following components are created:

- __Auto Scaling Group:__ Manages the fleet of EC2 instances and ensures that the desired capacity is maintained.
- __Scaling Policy:__ Automatically adjusts the number of instances based on demand.
- __Launch Template:__ Defines the instance configuration, including the custom AMI, networking, and user data.
- __Security Group:__ Restricts inbound and outbound traffic, allowing only the necessary communication paths.
- __Instance Profile with IAM Role:__ Grants EC2 instances permissions through an attached policy, specifically to retrieve secrets from AWS Secrets Manager (for database authentication), which will be explained in detail later.

This design guarantees that the web tier can scale efficiently while maintaining consistency, security, and controlled access to sensitive resources. The following code was used to create and configure the resources via cloudformation template:

~~~
Resources:
  SSMInstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref SSMRole
  SSMRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: 2012-10-17
        Statement:
          - Effect: Allow
            Principal:
              Service:
                - ec2.amazonaws.com
            Action:
              - sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
      Path: /
      Policies:
        - PolicyName: ReadSecrets
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Resource: '*'
                Action: secretsmanager:GetSecretValue
      Tags:
        - Key: Name
          Value: !Ref punitag
  LaunchTemplateWeb:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateData:
        ImageId: !Ref AMIImage
        InstanceType: !Ref InstanceType
        KeyName: !Ref PkeyName
        IamInstanceProfile:
          Arn: !GetAtt SSMInstanceProfile.Arn
        SecurityGroupIds:
          - !Ref AutoScalGroupWebInsSecurityGroup
      LaunchTemplateName: WebServerLaunchTemplate
      TagSpecifications:
        - ResourceType: launch-template
          Tags:
            - Key: Name
              Value: !Ref punitag
  AutoScalGroupWebInsSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable HTTP access via port 80
      VpcId: !Ref MainVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref WebALBSecurityGroup
      Tags:
        - Key: Name
          Value: !Ref punitag
  AutoScalGroupWebIns:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      DesiredCapacity: 2
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplateWeb
        Version: !GetAtt LaunchTemplateWeb.LatestVersionNumber
      TargetGroupARNs:
        - !Ref ApplicationLBTargetGroup
      HealthCheckGracePeriod: 60
      MaxInstanceLifetime: 432000
      MaxSize: 4
      MinSize: 2
      MetricsCollection:
        - Granularity: 1Minute
          Metrics:
            - GroupMinSize
            - GroupMaxSize
      VPCZoneIdentifier:
        - !Ref MainSubnetPrivate
        - !Ref SecondSubnetPrivate
      Tags:
        - Key: Name
          Value: !Ref punitag
          PropagateAtLaunch: true
  AutoScalPolicyWeb:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref AutoScalGroupWebIns
      PolicyType: TargetTrackingScaling
      EstimatedInstanceWarmup: 60
      TargetTrackingConfiguration:
        PredefinedMetricSpecification:
          PredefinedMetricType: ASGAverageCPUUtilization
        TargetValue: 30
~~~



<br>

**6. Creating an Aurora Database Cluster**

This section provisions the Aurora cluster that serves as the database tier of the 3-tier architecture. The cluster is deployed in private subnets to ensure secure, internal-only access, and it provides high availability and managed scalability.

The main components created are:
- __Aurora Cluster:__ The core database resource managing connectivity and replication.
- __Two Aurora Instances:__ One writer and one reader instance to support high availability and read scalability.
- __DB Subnet Group:__ Ensures the cluster is deployed across multiple Availability Zones within private subnets.
- __Security Group:__ Controls inbound access, allowing only traffic from the application layer.
- __DB Parameter Groups:__ Define database engine configurations and custom parameters for fine-tuned behavior.

This configuration provides a secure, highly available, and scalable database foundation for the application layer. The following code was used to create the resources via cloudformation template:

~~~
Resources:
  DBaseSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Enable MySQL access
      VpcId: !Ref MainVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !Ref AutoScalGroupWebInsSecurityGroup
      Tags:
        - Key: Name
          Value: !Ref punitag
  RDSSubnetGroup:
    Type: AWS::RDS::DBSubnetGroup
    Properties:
      DBSubnetGroupDescription: RDS Subnet Group
      SubnetIds:
        - !Ref MainSubnetPrivate
        - !Ref SecondSubnetPrivate
      Tags:
        - Key: Name
          Value: !Ref punitag
  RDSDataBaseCluster:
    Type: AWS::RDS::DBCluster
    Properties:
      DatabaseName: immersionday
      DBClusterIdentifier: rdscluster
      Engine: aurora-mysql
      EngineVersion: !Ref RDSEngineVersion
      StorageType: aurora
      MasterUsername: admin
      MasterUserPassword: !Ref RDSMasterPassword
      DBClusterParameterGroupName: !Ref RDSDBClusterParameterGroup
      DBSubnetGroupName: !Ref RDSSubnetGroup
      VpcSecurityGroupIds:
        - !Ref DBaseSecurityGroup
      Tags:
        - Key: Name
          Value: !Ref punitag
  RDSDBInstance1:
    Type: AWS::RDS::DBInstance
    Properties:
      DBParameterGroupName: !Ref RDSDBParameterGroup
      Engine: aurora-mysql
      DBClusterIdentifier: !Ref RDSDataBaseCluster
      PubliclyAccessible: false
      DBInstanceClass: db.r6i.large
      Tags:
        - Key: Name
          Value: !Ref punitag
  RDSDBInstance2:
    Type: AWS::RDS::DBInstance
    Properties:
      DBParameterGroupName: !Ref RDSDBParameterGroup
      Engine: aurora-mysql
      DBClusterIdentifier: !Ref RDSDataBaseCluster
      PubliclyAccessible: false
      DBInstanceClass: db.r6i.large
      Tags:
        - Key: Name
          Value: !Ref punitag
  RDSDBClusterParameterGroup:
    Type: AWS::RDS::DBClusterParameterGroup
    Properties:
      Description: RDS Database Cluster Parameter Group
      Family: aurora-mysql5.7
      Parameters:
        time_zone: US/Eastern
      Tags:
        - Key: Name
          Value: !Ref punitag
  RDSDBParameterGroup:
    Type: AWS::RDS::DBParameterGroup
    Properties:
      Description: RDS Database Parameter Group
      Family: aurora-mysql5.7
      Parameters:
        sql_mode: IGNORE_SPACE
        max_allowed_packet: 1024
        innodb_buffer_pool_size: '{DBInstanceClassMemory*3/4}'
      Tags:
        - Key: Name
          Value: !Ref punitag
~~~


<br>

**7. Creating a Secret**

This section provisions the AWS Secrets Manager secret that securely stores the username and password required for the Aurora database. The secret will be accessed by the application instances through the IAM role defined in the instance profile, ensuring that credentials are never hard-coded or exposed.

The main components are:
- __Secret:__ Holds the database credentials in an encrypted and managed form.
- __Secret Target Attachment:__ Links the secret to the Aurora cluster, enabling seamless rotation and integration.

This approach centralizes credential management, improves security, and allows application instances to retrieve authentication data on demand without manual intervention. The following code was used to create the resource via cloudformation template:

~~~
Resources:
  SecretRDSMasterPassword:
    Type: AWS::SecretsManager::Secret
    Properties:
      Name: mysecret
      SecretString: !Sub '{"username": "admin","password": "${RDSMasterPassword}"}'
      Tags:
        - Key: Name
          Value: !Ref punitag
  SecretRDSAttach:
    Type: AWS::SecretsManager::SecretTargetAttachment
    Properties:
      SecretId: !Ref SecretRDSMasterPassword
      TargetId: !Ref RDSDataBaseCluster
      TargetType: AWS::RDS::DBCluster
    DependsOn: RDSDataBaseCluster
~~~


<br>

**8. Creating the stack with Cloudformation**

Once we have finished designing the template for our stack, it is time to build it. As I mentioned before, the complete template is available in a public repository on GitHub called [3tier-ha-sc-webapp](https://github.com/Sjleal/aws-3tier-ha-sc-webapp/blob/main/dev/3tier-ha-sc-webapp.yaml).

You can use a tool named [Application Composer](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/app-composer-for-cloudformation.html) in CloudFormation console mode to validate your template and also you can drag, drop, configure, and connect a variety of resources onto a visual canvas. The following image shows the resources involved in the template and a canvas representation of them.

![Image description](https://github.com/Sjleal/aws-3tier-ha-sc-webapp/blob/main/images/diagram/aws-composer-3tier-ha-sc-webapp.png)

To [create the stack](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-create-stack.html) you need to follow these steps:

1. Logging in to the AWS Management console and open the AWS CloudFormation console. 
2. Choose Create Stack to start the Create Stack wizard.
3. __Selecting a stack template.__ On the Specify template page, choose Upload a template file to select the CloudFormation template designed.
4. __Specifying stack parameters.__ On the Specify stack details page, type a stack name in the Stack name box. In the Parameters section, specify parameters availability zones, AMI, instance type, key pair, RDS engine, RDS password, and the unique tag.
5. __Setting AWS CloudFormation stack options.__ On _Permissions - optional_, select a role that allow to this stack create, update or delete the resources involved, __IMPORTANT:__ you can create this role on IAM console but remember using the least privilege principle. In _Stack failure options_ you can set the stack behavior in case of provisioning failure.
6. __Reviewing your stack.__ Here You can review the selected options and press Submmit button to start the creation process.

<br>

---

>**Final steps:**<br>
With the infrastructure fully deployed, the final phase focuses on validating the behavior of the solution under real conditions. Two key aspects are tested:

- __Auto Scaling Group behavior__ – Verifying that the web tier responds to increased load by automatically scaling across Availability Zones.

- __Database connectivity__ – Ensuring that the web application can securely retrieve credentials via the instance profile and interact with the Aurora cluster to perform basic operations (query, insert, update, delete).

These tests provide practical confirmation that the architecture is not only correctly provisioned but also operationally resilient, scalable, and secure.

**9. Testing the autoscaling group**

To validate the behavior of the Auto Scaling Group, the deployed web service exposes useful metadata and load information directly in its response. Each instance displays its Instance ID, the Availability Zone where it is running, and the current CPU utilization. This allows you to confirm which instance is serving a request and to observe load distribution in real time.

The application also provides an option to generate artificial CPU stress on the host, driving utilization up to 100%. This controlled load test triggers the scaling policy, since the baseline threshold is defined at 30% average CPU utilization. As a result, the Auto Scaling Group begins to launch additional instances in other Availability Zone, ensuring both scalability and high availability of the web tier.

For a detailed step-by-step guide on how to run these tests, you can refer to the original [AWS workshop](https://catalog.us-east-1.prod.workshops.aws/workshops/869a0a06-1f98-4e19-b5ac-cbb1abdfc041/en-US/advanced-modules/compute/test-service).

This process demonstrates how the architecture dynamically adapts to increased demand, maintaining performance and resilience without manual intervention.

<br>

**10. Connecting to RDS**

Although the connection to the Aurora RDS cluster may seem straightforward, it is important to highlight that this access is enabled through the instance profile attached to the EC2 instances. The profile’s IAM role includes a policy that allows the instances to retrieve database credentials from AWS Secrets Manager rather than storing them locally. This ensures that the application can securely obtain the username and password needed to establish the connection.

Once connected, the web application is able to query, insert, update, and delete records in the Aurora database, demonstrating full interaction with the persistence layer in a secure and controlled manner.

<br>

## Summary

When the resources created for this solution are no longer required, it is important to perform an appropriate cleanup of the resources created by the stack. To do this, you can use the CloudFormation console and follow this [guide](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-console-delete-stack.html). Remember to review the retention policies in the template and the events in the stack deletion process to avoid additional charges to the AWS account.

This project demonstrated the design and implementation of a highly available 3-tier architecture on AWS, fully deployed through CloudFormation. The solution integrated key components such as a VPC with public and private subnets, an Application Load Balancer, an Auto Scaling Group with custom AMIs, an Aurora database cluster, and supporting resources like NAT Gateways, route tables, security groups, IAM roles, and AWS Secrets Manager.

Beyond showcasing the technical robustness of the design, this exercise has been an instructive and enriching experience. It highlights not only how AWS services can be orchestrated to deliver scalability, resilience, and security, but also how Infrastructure as Code enables reproducibility, maintainability, and operational efficiency.

In essence, this proof of concept has provided both a practical and academic perspective on modern cloud architectures, reinforcing best practices while deepening understanding of the AWS ecosystem.

## References

- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-snippets.html
- https://catalog.us-east-1.prod.workshops.aws/workshops/869a0a06-1f98-4e19-b5ac-cbb1abdfc041/en-US/advanced-modules
- https://github.com/aws-cloudformation/aws-cloudformation-templates/tree/main
