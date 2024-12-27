# AWS Infrastructure Setup Guide using CloudFormation

## Project Structure

```plaintext
infrastructure/
├── 1-network/
│   └── network.yaml       # VPC, Subnets, etc.
├── 2-container/
│   └── container.yaml     # ALB, ECS Cluster, Roles
├── 3-service/
│   └── service.yaml       # ECS Service, Auto Scaling
└── 4-pipeline/
    └── pipeline.yaml      # CI/CD Pipeline
```

## 1. Network Stack Setup

### Key Components:
- VPC
- Public Subnets
- Internet Gateway
- Route Tables

### Deployment Steps:
1. Create network.yaml:
```yaml
Parameters:
  AvailabilityZone1:
    Default: us-east-1a
  AvailabilityZone2:
    Default: us-east-1b

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 20.192.0.0/20
      EnableDnsSupport: true
      EnableDnsHostnames: true

  # Public Subnets
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 20.192.0.0/21
      AvailabilityZone: !Ref AvailabilityZone1

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      CidrBlock: 20.192.8.0/21
      AvailabilityZone: !Ref AvailabilityZone2
```

2. Deploy using AWS CLI:
```bash
aws cloudformation create-stack \
  --stack-name network \
  --template-body file://network.yaml
```

## 2. Container Infrastructure Stack

### Key Components:
- Application Load Balancer
- ECS Cluster
- IAM Roles
- Security Groups

### Deployment Steps:
1. Create container.yaml:
```yaml
Parameters:
  NetworkStack:
    Type: String
    Default: network

Resources:
  ECSCluster:
    Type: AWS::ECS::Cluster

  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  ApplicationLoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Subnets:
        - Fn::ImportValue: !Sub ${NetworkStack}-PublicSubnet1
        - Fn::ImportValue: !Sub ${NetworkStack}-PublicSubnet2
```

2. Deploy:
```bash
aws cloudformation create-stack \
  --stack-name container \
  --template-body file://container.yaml \
  --capabilities CAPABILITY_IAM
```

## 3. Service Stack Setup

### Key Components:
- ECS Service
- Task Definition
- Auto Scaling Configuration
- Target Groups

### Deployment Steps:
1. Create service.yaml:
```yaml
Parameters:
  ContainerStack:
    Type: String
    Default: container

Resources:
  TaskDefinition:
    Type: AWS::ECS::TaskDefinition
    Properties:
      Cpu: 256
      Memory: 512
      NetworkMode: awsvpc
      RequiresCompatibilities:
        - FARGATE

  ECSService:
    Type: AWS::ECS::Service
    Properties:
      Cluster: !ImportValue 
        Fn::Sub: ${ContainerStack}-Cluster
      DesiredCount: 2
      LaunchType: FARGATE
```

2. Deploy:
```bash
aws cloudformation create-stack \
  --stack-name service \
  --template-body file://service.yaml
```

## 4. CI/CD Pipeline Stack

### Key Components:
- CodePipeline
- CodeBuild
- ECR Repository
- GitHub Integration

### Deployment Steps:
1. Create pipeline.yaml:
```yaml
Parameters:
  GitHubOwner:
    Type: String
  GitHubRepo:
    Type: String
  GitHubToken:
    Type: String
    NoEcho: true

Resources:
  ECRRepository:
    Type: AWS::ECR::Repository

  CodeBuildProject:
    Type: AWS::CodeBuild::Project
    Properties:
      Environment:
        Type: LINUX_CONTAINER
        ComputeType: BUILD_GENERAL1_SMALL
        Image: aws/codebuild/amazonlinux2-x86_64-standard:3.0
        PrivilegedMode: true
```

2. Deploy:
```bash
aws cloudformation create-stack \
  --stack-name pipeline \
  --template-body file://pipeline.yaml \
  --capabilities CAPABILITY_IAM
```

## Required Files in Application Repository

### 1. Dockerfile
```dockerfile
FROM node:14
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
EXPOSE 80
CMD ["npm", "start"]
```

### 2. buildspec.yml
```yaml
version: 0.2
phases:
  pre_build:
    commands:
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ECR_REPOSITORY_URI
  build:
    commands:
      - docker build -t $ECR_REPOSITORY_URI:latest .
  post_build:
    commands:
      - docker push $ECR_REPOSITORY_URI:latest
```

### 3. appspec.yaml
```yaml
version: 0.0
Resources:
  - TargetService:
      Type: AWS::ECS::Service
      Properties:
        TaskDefinition: <TASK_DEFINITION>
        LoadBalancerInfo:
          ContainerName: "app"
          ContainerPort: 80
```

## Deployment Order

1. Deploy Network Stack
2. Deploy Container Stack
3. Deploy Service Stack
4. Deploy Pipeline Stack

## Validation Steps

### After Network Stack:
```bash
# Verify VPC
aws ec2 describe-vpcs --filters Name=tag:aws:cloudformation:stack-name,Values=network

# Verify Subnets
aws ec2 describe-subnets --filters Name=vpc-id,Values=<vpc-id>
```

### After Container Stack:
```bash
# Verify ECS Cluster
aws ecs list-clusters

# Verify ALB
aws elbv2 describe-load-balancers
```

### After Service Stack:
```bash
# Verify ECS Service
aws ecs list-services --cluster <cluster-name>

# Verify Task Definition
aws ecs list-task-definitions
```

### After Pipeline Stack:
```bash
# Verify Pipeline
aws codepipeline get-pipeline --name <pipeline-name>

# Verify ECR Repository
aws ecr describe-repositories
```

## Common Issues and Solutions

1. **VPC Creation Fails**
   - Check CIDR block conflicts
   - Verify AZ availability

2. **ALB Creation Fails**
   - Verify subnet configurations
   - Check security group rules

3. **ECS Service Fails**
   - Verify task definition
   - Check container health checks

4. **Pipeline Fails**
   - Verify GitHub token
   - Check CodeBuild role permissions

## Best Practices

1. **Security**
   - Use least privilege for IAM roles
   - Implement security groups properly
   - Enable encryption for sensitive data

2. **Networking**
   - Plan CIDR ranges carefully
   - Use multiple AZs
   - Implement proper routing

3. **Container**
   - Use proper task sizing
   - Implement health checks
   - Configure logging

4. **Pipeline**
   - Use proper branching strategy
   - Implement proper testing
   - Configure notifications

## Monitoring Setup

1. **CloudWatch Alarms**
```yaml
Resources:
  CPUUtilizationAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmDescription: CPU utilization above 75%
      MetricName: CPUUtilization
      Namespace: AWS/ECS
      Threshold: 75
```

2. **Logs**
```yaml
LogGroup:
  Type: AWS::Logs::LogGroup
  Properties:
    RetentionInDays: 30
```

## Cleanup

```bash
# Delete stacks in reverse order
aws cloudformation delete-stack --stack-name pipeline
aws cloudformation delete-stack --stack-name service
aws cloudformation delete-stack --stack-name container
aws cloudformation delete-stack --stack-name network
```
