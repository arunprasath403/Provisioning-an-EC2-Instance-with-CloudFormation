<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
</head>
<body>

  <h1>Setting Up a Web Application Stack on AWS with CloudFormation</h1>

  <p>This guide walks you through setting up a basic web application stack on AWS using CloudFormation. By following these steps, you'll create an EC2 instance, configure security groups for SSH and HTTP/HTTPS traffic, assign an Elastic IP address, and make the template dynamic for different environments.</p>

  <h2>Prerequisites</h2>

  <p>Before you begin, ensure you have the following:</p>
  <ul>
    <li>An AWS account (<a href="https://aws.amazon.com/free/">Create an AWS account</a> if you don't have one).</li>
    <li>AWS CLI installed (<a href="https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html">Installation instructions</a>).</li>
    <li>A key pair to use for SSH (<a href="https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html">Create a key pair</a>).</li>
  </ul>

  <p>If you're unfamiliar with CloudFormation, it's recommended to review the <a href="#intro-to-cloudformation">Intro to CloudFormation</a> before getting started.</p>

  <h2>Steps</h2>

  <h3>1. Create Basic Amazon EC2 Instance</h3>

  <p>Create a basic EC2 instance using CloudFormation.</p>

  <pre><code>

Resources:
  WebAppInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0d5eff06f840b45e9 # ImageID valid only in us-east-1 region
      InstanceType: t2.micro
</code></pre>

  <p>To create the stack using this template:</p>

  <pre><code>$ aws cloudformation create-stack --stack-name ec2-example --template-body file://01_ec2.yaml
</code></pre>

  <h3>2. Enable SSH and HTTP/HTTPS Traffic</h3>

  <p>Update the CloudFormation template to add a security group allowing SSH (port 22), and HTTP/HTTPS (ports 80 and 443) traffic.</p>

  <pre><code>
Resources:
  WebAppInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0d5eff06f840b45e9
      InstanceType: t2.micro
      KeyName: jenna
      SecurityGroupIds:
        - !Ref WebAppSecurityGroup

  WebAppSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Join ["-", [webapp-security-group, dev]]
      GroupDescription: "Allow HTTP/HTTPS and SSH inbound and outbound traffic"
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
</code></pre>

  <p>Update the stack:</p>

  <pre><code>$ aws cloudformation update-stack --stack-name ec2-example --template-body file://02_ec2.yaml
</code></pre>

  <h3>3. Assign an IP Address and Output the Website URL</h3>

  <p>Assign an Elastic IP address to the EC2 instance and output the website URL.</p>

  <pre><code> Assign Elastic IP and output URL

Resources:
  WebAppInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0d5eff06f840b45e9
      InstanceType: t2.micro
      KeyName: jenna
      SecurityGroupIds:
        - !Ref WebAppSecurityGroup

  WebAppSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Join ["-", [webapp-security-group, dev]]
      GroupDescription: "Allow HTTP/HTTPS and SSH inbound and outbound traffic"
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0

  WebAppEIP:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc
      InstanceId: !Ref WebAppInstance
      Tags:
        - Key: Name
          Value: !Join ["-", [webapp-eip, dev]]

Outputs:
  WebsiteURL:
    Value: !Sub http://${WebAppEIP}
    Description: WebApp URL
</code></pre>

  <p>Update the stack again:</p>

  <pre><code>$ aws cloudformation update-stack --stack-name ec2-example --template-body file://03_ec2.yaml
</code></pre>

  <h3>4. Make the Template Dynamic</h3>

  <p>Enhance the template by making it dynamic with parameters and mappings for environment type, instance type, AMI ID, etc.</p>

  <pre><code> Make the CloudFormation template dynamic

Parameters:
  AvailabilityZone:
    Type: AWS::EC2::AvailabilityZone::Name
  EnvironmentType:
    Description: "Specify the Environment type of the stack."
    Type: String
    Default: dev
    AllowedValues:
      - dev
      - test
      - prod
  AmiID:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Description: "The ID of the AMI."
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2
  KeyPairName:
    Type: String
    Description: The name of an existing Amazon EC2 key pair in this region to use to SSH into the Amazon EC2 instances.

Mappings:
  EnvironmentToInstanceType:
    dev:
      InstanceType: t2.nano
    test:
      InstanceType: t2.micro
    prod:
      InstanceType: t2.small

Resources:
  WebAppInstance:
    Type: AWS::EC2::Instance
    Properties:
      AvailabilityZone: !Ref AvailabilityZone
      ImageId: !Ref AmiID
      InstanceType:
        !FindInMap [
          EnvironmentToInstanceType,
          !Ref EnvironmentType,
          InstanceType,
        ]
      KeyName: !Ref KeyPairName
      SecurityGroupIds:
        - !Ref WebAppSecurityGroup

  WebAppSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Join ["-", [webapp-security-group, !Ref EnvironmentType]]
      GroupDescription: "Allow HTTP/HTTPS and SSH inbound and outbound traffic"
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0

  WebAppEIP:
    Type: AWS::EC2::EIP
    Properties:
      Domain: vpc
      InstanceId: !Ref WebAppInstance
      Tags:
        - Key: Name
          Value: !Join ["-", [webapp-eip, !Ref EnvironmentType]]

Outputs:
  WebsiteURL:
    Value: !Sub http://${WebAppEIP}
    Description: WebApp URL
</code></pre>

  <p>Update the stack with parameters:</p>

  <pre><code>$ aws cloudformation update-stack --stack-name ec2-example --template-body
