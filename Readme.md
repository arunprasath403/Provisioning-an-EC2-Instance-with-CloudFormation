Prerequisites
To work through the examples in this post, you’ll need:

an AWS account (you can create your account here if you don’t already have one),
the AWS CLI installed (you can find instructions for installing the AWS CLI here), and
a key-pair to use for SSH (you can create a key-pair following these instructions).
Unfamiliar with CloudFormation or feeling a little rusty? Check out my Intro to CloudFormation post before getting started.

Creating the CloudFormation Template
I’m of the mindset of “Make it work. Make it right. Make it fast.” so we’ll iterate to get to our final template and and make it better at the end. At the end of this post, we’ll delete the stack we’ve created so that you don’t incur any charges and then you can (quickly) recreate the stack when we move on to the next post.

Make it work. Make it right. Make it fast. — Kent Beck

Just want the code? Grab it here.

Let’s get started!

1. Create Basic Amazon EC2 Instance
First, we’ll create a basic EC2 instance with CloudFormation.

AWSTemplateFormatVersion: 2010-09-09
Description: Part 1 - Build a webapp stack with CloudFormation

Resources:
  WebAppInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0d5eff06f840b45e9 # ImageID valid only in us-east-1 region
      InstanceType: t2.micro
In the template above, we have one resource with a type of AWS::EC2::Instance. We’ve hardcoded both the ImageId (AMI) and InstanceType. Note this ImageId will only work in the us-east-1 region.

To create the stack using this template, run the create-stack command-line:

$ aws cloudformation create-stack --stack-name ec2-example --template-body file://01_ec2.yaml
You now have an EC2 instance in the us-east-1 region! But we have no way to access this instance. We cannot SSH into it yet because we didn’t assign it a security group allowing SSH traffic or specify a key-pair name. There’s nothing running on HTTP or HTTPS ports yet, but even if there were, we wouldn’t be able to access that either. Let’s fix that now.

2. Enable SSH and HTTP/HTTPS Traffic
Now, we’ll update the CloudFormation template to add a security group resource that allows traffic in on port 22 for SSH and ports 80 and 443 for HTTP and HTTPS traffic. Here, we’ve allowed all IP addresses to access these ports, but you may want to lock this down further (especially the SSH rule) to IP addresses you trust.

AWSTemplateFormatVersion: 2010-09-09
Description: Part 1 - Build a webapp stack with CloudFormation

Resources:
  WebAppInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0d5eff06f840b45e9 # ImageID valid only in us-east-1 region
      InstanceType: t2.micro
      KeyName: jenna # <-- Change to use your key-pair name
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
Again, we’ve hardcoded some values in our template, but we’ll fix that up soon. Before updating the stack with this template, you’ll need to make one small change to use your key-pair name.

You can update your stack using the update-stack command:

$ aws cloudformation update-stack --stack-name ec2-example --template-body file://02_ec2.yaml
Now your EC2 instance should be accessible with SSH using your key-pair. To test this out, first navigate to your new stack in the AWS CloudFormation Console to find the instance you created.

Resources in the CloudFormation stack

Then go to the instance and copy the public DNS for your instance.

images/cf-ssh-example-2.png

Then, SSH into the instance like this:

$ ssh -i "YOUR_KEY_PAIR_NAME.pem" ec2-user@PUBLIC_DNS
Your command will look something like this:

$ ssh -i "jenna.pem" ec2-user@ec2-18-212-186-244.compute-1.amazonaws.com
Hint: You can also grab the command directly by viewing the "Connect" details at the top of the instance and copying the example command at the bottom of the "SSH client" tab.

There is nothing being served on port 80 or 443 yet, so you won’t be able to test HTTP/HTTPS access yet.

3. Assign an IP Address and Output the Website URL
We also need to give our EC2 instance an elastic IP address (EIP). An elastic ip address is a static IP address that won’t change every time we re-provision the instance. We’ll also output the website URL.

AWSTemplateFormatVersion: 2010-09-09
Description: Part 1 - Build a webapp stack with CloudFormation

Resources:
  WebAppInstance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: ami-0d5eff06f840b45e9 # ImageID valid only in us-east-1 region
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
In this template update, we’ve created a resource AWS::EC2::EIP for an elastic IP address and assigned it to our EC2 instance. We also added the Outputs section and out the website URL, which at this point is our elastic IP address.

You can run the same update-stack command from above (make sure to reference the right file name). When it completes (the EIP takes a few minutes to provision), your EC2 instance will have a static IP address and you’ll be able to see both the IP address and URL in the Outputs section of the CloudFormation stack in the Console.

images/cf-outputs-example-1.png

4. Make the Template Dynamic
Now that we’ve made it work, the last task we have to do is to make it better. Remember that our template has hardcoded values for a number of configurations that should really be dynamic. We want these values to be dynamic for a few reasons. Maybe we want our development team to create a stack in their own AWS account for development purposes. And because it’s development, maybe we only want t2.micro instances, whereas in production we need t2.medium instances. We also might want to create this stack in other regions, so we’ll need to use the region-specific ImageId (AMI).

We can make our template more dynamic by using parameters and mappings. Let’s add them now!

AWSTemplateFormatVersion: 2010-09-09
Description: Part 1 - Build a webapp stack with CloudFormation

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
We’ve added parameters for AvailabilityZone, EnvironmentType, AmiID, and KeyPairName. AvailabilityZone will pull from AWS::EC2::AvailabilityZone::Name. EnvironmentType will be one of dev, test, or prod and default to dev. The ImageId will be an AWS::SSM::Parameter::ValueAWS::EC2::Image::Id type. By using the public parameter /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2, it will use the region-specific AMI, the value of which is stored in the AWS Systems Manager Parameter Store. In this case, we'll rely on the default value. And the KeyPairName is the name of an existing key-pair.
The other section we’ve added is the Mappings section. We’ll use the EnvironmentToInstanceType mapping to lookup the instance type for the selected environment.

You can update the stack with this command, passing in the parameter values:

$ aws cloudformation update-stack --stack-name ec2-example --template-body file://04_ec2.yaml \
--parameters ParameterKey=AvailabilityZone,ParameterValue=us-east-1a \
ParameterKey=EnvironmentType,ParameterValue=dev \
ParameterKey=KeyPairName,ParameterValue=jenna
Now it will be easier to reuse this template for other environments and in other regions!

Wrapping Up
Delete Your Stack
Don’t forget to delete your stack so you don’t accrue charges. You can do that with the delete-stack command:

$ aws cloudformation delete-stack --stack-name ec2-example
