**AWS CloudFormation project to create a VPC setup** including a public and private subnet, Internet Gateway, NAT Gateway, route tables, and EC2 instances (one in each subnet).
```
git clone https://github.com/atulkamble/aws-vpc-cloudformation-project.git
cd aws-vpc-cloudformation-project
```
Create Keypair key.pem
Create Role >> AWSCloudformationRole

AWS Console >> Cloudformation >> Upload Tempale >> Select Template 

---

### ✅ **CloudFormation VPC Project Overview**

This CloudFormation template provisions:

* A new **VPC**
* **2 Subnets**: Public and Private
* **Internet Gateway** for public access
* **NAT Gateway** for private subnet internet access
* **Route Tables** with appropriate associations
* **Security Group**
* **2 EC2 Instances** (one in public, one in private subnet)

---

### 📁 **vpc-setup.yaml**

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: VPC Setup with Public and Private Subnets, IGW, NAT, EC2 Instances

Parameters:
  KeyName:
    Description: EC2 KeyPair to SSH
    Type: AWS::EC2::KeyPair::KeyName

Resources:

  MyVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: MyVPC

  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: MyIGW

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref InternetGateway

  PublicSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.1.0/24
      AvailabilityZone: !Select [ 0, !GetAZs '' ]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: PublicSubnet

  PrivateSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref MyVPC
      CidrBlock: 10.0.2.0/24
      AvailabilityZone: !Select [ 1, !GetAZs '' ]
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: PrivateSubnet

  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC

  PublicRoute:
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

  EIP:
    Type: AWS::EC2::EIP
    DependsOn: AttachGateway
    Properties:
      Domain: vpc

  NATGateway:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt EIP.AllocationId
      SubnetId: !Ref PublicSubnet

  PrivateRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref MyVPC

  PrivateRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NATGateway

  PrivateSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId: !Ref PrivateSubnet
      RouteTableId: !Ref PrivateRouteTable

  InstanceSG:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow SSH, HTTP
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  PublicInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      KeyName: !Ref KeyName
      SubnetId: !Ref PublicSubnet
      ImageId: ami-0c55b159cbfafe1f0 # Amazon Linux 2 AMI
      SecurityGroupIds:
        - !Ref InstanceSG
      Tags:
        - Key: Name
          Value: PublicInstance

  PrivateInstance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: t2.micro
      KeyName: !Ref KeyName
      SubnetId: !Ref PrivateSubnet
      ImageId: ami-0c55b159cbfafe1f0
      SecurityGroupIds:
        - !Ref InstanceSG
      Tags:
        - Key: Name
          Value: PrivateInstance

Outputs:
  VPCId:
    Description: VPC ID
    Value: !Ref MyVPC
  PublicIP:
    Description: Public Instance IP
    Value: !GetAtt PublicInstance.PublicIp
```

---

### 📦 To Deploy via AWS CLI:

1. Save the file as `vpc-setup.yaml`
2. Run the command:

```bash
aws cloudformation create-stack \
  --stack-name vpc-project \
  --template-body file://vpc-setup.yaml \
  --parameters ParameterKey=KeyName,ParameterValue=mykey \
  --capabilities CAPABILITY_NAMED_IAM
```

---
