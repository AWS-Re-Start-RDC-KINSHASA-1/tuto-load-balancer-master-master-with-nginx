# tuto-load-balancer-master-master-with-nginx
configurer le fichier yml
voici la configuration  

AWSTemplateFormatVersion: "2010-09-09"

Resources:
  MyEC2Instance:
    Type: "AWS::EC2::Instance"
    Properties:
      ImageId: ami-07761f3ae34c4478d    
      InstanceType: t2.micro
      KeyName: test2
      SubnetId: subnet-0d9213955e664b972
      SecurityGroupIds:
        - sg-0846155c2b7199f98

  MyTargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      HealthCheckPath: "/"
      HealthCheckProtocol: "HTTP"
      HealthCheckPort: 80
      Port: 80
      Protocol: "HTTP"
      Targets:
        - Id: !Ref MyEC2Instance
          Port: 80
      VpcId: vpc-04548a09aa672938f # Replace with your existing VPC ID

  MyLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: MyLoadBalancer
      Subnets:
        - subnet-0d9213955e664b972 # Subnet in Availability Zone A
        - subnet-0dcfd45b1d52a5312 # Subnet in Availability Zone B
      SecurityGroups:
        - sg-0846155c2b7199f98
      Scheme: internet-facing
      LoadBalancerAttributes:
        - Key: idle_timeout.timeout_seconds
          Value: "60"
      Tags:
        - Key: Name
          Value: MyLoadBalancer
      IpAddressType: ipv4

  MyListener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref MyTargetGroup
      LoadBalancerArn: !Ref MyLoadBalancer
      Port: 80
      Protocol: HTTP

  MyLaunchConfiguration:
    Type: AWS::AutoScaling::LaunchConfiguration
    Properties:
      ImageId: ami-07761f3ae34c4478d
      InstanceType: t2.micro
      SecurityGroups:
        - sg-0846155c2b7199f98
      UserData:
        Fn::Base64: !Sub |
          #!/bin/bash
          sudo yum update -y
          sudo yum install httpd -y
          sudo systemctl enable httpd
          sudo systemctl start httpd
          sudo echo '<html><body><h1>Hello, World!</h1></body></html>' > /var/www/html/index.html
          # Additional setup and configurations here

  MyAutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      AutoScalingGroupName: MyAutoScalingGroup
      VPCZoneIdentifier:
        - subnet-0d9213955e664b972 # Replace with your existing Subnet ID
      LaunchConfigurationName: !Ref MyLaunchConfiguration
      MinSize: 2
      MaxSize: 4
      DesiredCapacity: 2
      HealthCheckType: EC2
      HealthCheckGracePeriod: 300
      TargetGroupARNs:
        - !Ref MyTargetGroup
