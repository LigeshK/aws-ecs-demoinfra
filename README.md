# AWS Containerisation - ECS Workshop Infrastructure

A fully integrated AWS infrastructure for deploying a two-tier microservices application on Amazon ECS (Fargate), leveraging an Application Load Balancer and a comprehensive set of AWS services.

# Summary

In this workshop, we will walk through how to deploy a simple full-stack containerized application on AWS using ECS with Fargate. The application consists of a lightweight frontend and backend, designed primarily to help you understand the deployment workflow rather than provide full business functionality. The frontend is a basic React or HTML page that sends a request to a .NET Core backend API running on port 3000. Its role is to perform a health check and confirm whether the backend is reachable. If the API responds successfully, the frontend displays an “OK” message.

This application runs inside an Amazon ECS cluster, illustrated in the architecture using a dotted boundary. Within ECS, workloads run as Tasks, which are similar to Kubernetes Pods. Each Task contains one or more containers. For compute, we will be using AWS Fargate, a fully managed serverless platform for running containers. With Fargate, you do not manage any EC2 instances, operating systems, patching, or scaling infrastructure. Instead, AWS provisions and manages the compute environment. Although this drastically simplifies operations, Fargate does come with technical limitations, so you should always ensure it aligns with your application’s requirements. As an alternative, ECS can also run Tasks on self-managed EC2 instances, giving more control but requiring more maintenance.

To expose the application to users, the frontend is integrated with an Elastic Load Balancer (ELB). The ECS Service handles the registration of running backend Tasks with the load balancer so traffic flows smoothly to the container instances. When the application must be publicly accessible, the load balancer is deployed in a public subnet. For internal-only traffic within an organization or VPC, a private subnet is recommended.

Deployment begins on the developer’s machine using Docker. Developers build and test the application locally, then package it into a container image. The image is pushed to Amazon ECR (Elastic Container Registry), which serves as a secure repository for container images. The ECS cluster then pulls the latest image and deploys updated containers. This workflow integrates naturally with CI/CD systems. In this workshop, we will use GitHub Actions to automate image builds, pushes, and ECS deployments, demonstrating a complete end-to-end container deployment pipeline on AWS.

## Architecture Overview

Designed for high availability, scalability, and security, this infrastructure runs containerized applications across multiple Availability Zones.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                  Internet                                        │
└─────────────────────────────┬───────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         Internet Gateway                                        │
└─────────────────────────────┬───────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    VPC (10.0.0.0/16) - ecs-vpc                                 │
│                                                                                 │
│  ┌─────────────────────────┐              ┌─────────────────────────┐          │
│  │   Public Subnet 1       │              │   Public Subnet 2       │          │
│  │   (10.0.1.0/24)         │              │   (10.0.2.0/24)         │          │
│  │   AZ-1                  │              │   AZ-2                  │          │
│  │                         │              │                         │          │
│  │  ┌─────────────────┐    │              │  ┌─────────────────┐    │          │
│  │  │  NAT Gateway    │    │              │  │                 │    │          │
│  │  │  (EIP)          │    │              │  │                 │    │          │
│  │  └─────────────────┘    │              │  └─────────────────┘    │          │
│  └─────────────────────────┘              └─────────────────────────┘          │
│                              │                          │                       │
│                              └──────────┬───────────────┘                       │
│                                         │                                       │
│                              ┌─────────────────────────┐                       │
│                              │  Application Load       │                       │
│                              │  Balancer (ALB)         │                       │
│                              │  Port 80                │                       │
│                              │  Routes: / → Frontend   │                       │
│                              │         /api* → Backend │                       │
│                              └─────────────────────────┘                       │
│                                         │                                       │
│  ┌─────────────────────────┐              ┌─────────────────────────┐          │
│  │   Private Subnet 1      │              │   Private Subnet 2      │          │
│  │   (10.0.3.0/24)         │              │   (10.0.4.0/24)         │          │
│  │   AZ-1                  │              │   AZ-2                  │          │
│  │                         │              │                         │          │
│  │  ┌─────────────────┐    │              │  ┌─────────────────┐    │          │
│  │  │ ECS Tasks       │    │              │  │ ECS Tasks       │    │          │
│  │  │ Frontend:80     │    │              │  │ Frontend:80     │    │          │
│  │  │ Backend:3000    │    │              │  │ Backend:3000    │    │          │
│  │  │ (Fargate)       │    │              │  │ (Fargate)       │    │          │
│  │  └─────────────────┘    │              │  └─────────────────┘    │          │
│  └─────────────────────────┘              └─────────────────────────┘          │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────────┐
│                            Supporting Services                                   │
│                                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                │
│  │ ECR Repository  │  │ CloudWatch Logs │  │ Service         │                │
│  │ my-app-repo     │  │ /ecs/frontend   │  │ Discovery       │                │
│  │ - frontend:tag  │  │ /ecs/backend    │  │ ecsworkshop.    │                │
│  │ - backend:tag   │  │                 │  │ local           │                │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                │
│                                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                │
│  │ ECS Cluster     │  │ IAM Roles       │  │ Security Groups │                │
│  │ my-ecs-cluster  │  │ - Task Exec     │  │ - ALB (Port 80) │                │
│  │ - Frontend Svc  │  │ - Service Role  │  │ - ECS (80,3000) │                │
│  │ - Backend Svc   │  │                 │  │                 │                │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘                │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Infrastructure Components

### Network Layer (network-stack.cf.yml)

- **VPC**: Custom Virtual Private Cloud (10.0.0.0/16)
- **Public Subnets**: 2 subnets across different AZs for high availability
  - Public Subnet 1: 10.0.1.0/24 (hosts ALB and NAT Gateway)
  - Public Subnet 2: 10.0.2.0/24 (hosts ALB for redundancy)
- **Private Subnets**: 2 subnets for ECS tasks
  - Private Subnet 1: 10.0.3.0/24 (ECS tasks)
  - Private Subnet 2: 10.0.4.0/24 (ECS tasks)
- **Internet Gateway**: Enables internet access for public subnets
- **NAT Gateway**: Provides outbound internet access for private subnets
- **Route Tables**: Proper routing for public and private traffic

### Application Layer (ecs-stack.cf.yml)

- **ECS Cluster**: Container orchestration platform (my-ecs-cluster)
- **Application Load Balancer**: Routes traffic between frontend and backend
  - Default route (/) → Frontend service
  - API routes (/api\*) → Backend service
- **ECS Services**: Manages container deployment and scaling
  - Frontend Service: Runs on port 80
  - Backend Service: Runs on port 3000
- **Task Definitions**: Container specifications using Fargate
- **ECR Repository**: Stores Docker images for both services
- **Service Discovery**: Enables service-to-service communication
- **Security Groups**: Network access control
- **CloudWatch Logs**: Centralized logging with 7-day retention
- **IAM Roles**: Secure access permissions for ECS tasks and services

## Key Features

- **High Availability**: Multi-AZ deployment across 2 availability zones
- **Security**: Private subnets for application workloads, security groups for network isolation
- **Scalability**: ECS Fargate for serverless container management
- **Monitoring**: CloudWatch integration for logs and metrics
- **Service Discovery**: Internal DNS for service communication
- **Load Balancing**: Application Load Balancer with path-based routing
- **Container Registry**: ECR for secure image storage with vulnerability scanning

## Deployment

### Prerequisites

- AWS CLI configured with appropriate permissions
- GitHub repository with AWS credentials stored as secrets:
  - `AWS_ACCESS_KEY_ID`
  - `AWS_SECRET_ACCESS_KEY`

### Automated Deployment

The infrastructure deploys automatically via GitHub Actions when code is pushed to the `main` branch:

1. **Network Stack**: Creates VPC, subnets, gateways, and routing
2. **ECS Stack**: Deploys container services, load balancer, and supporting resources

### Manual Deployment

```bash
# Deploy network stack
aws cloudformation deploy \
  --template-file templates/network-stack.cf.yml \
  --stack-name ecs-network-stack \
  --region ap-southeast-2

# Deploy ECS stack (after network stack completes)
aws cloudformation deploy \
  --template-file templates/ecs-stack.cf.yml \
  --stack-name ecs-app-stack \
  --capabilities CAPABILITY_IAM \
  --region ap-southeast-2 \
  --parameter-overrides \
    VpcId=$(aws cloudformation describe-stacks --stack-name ecs-network-stack --query "Stacks[0].Outputs[?OutputKey=='VpcId'].OutputValue" --output text) \
    PublicSubnet1Id=$(aws cloudformation describe-stacks --stack-name ecs-network-stack --query "Stacks[0].Outputs[?OutputKey=='PublicSubnet1Id'].OutputValue" --output text) \
    PublicSubnet2Id=$(aws cloudformation describe-stacks --stack-name ecs-network-stack --query "Stacks[0].Outputs[?OutputKey=='PublicSubnet2Id'].OutputValue" --output text) \
    PrivateSubnet1Id=$(aws cloudformation describe-stacks --stack-name ecs-network-stack --query "Stacks[0].Outputs[?OutputKey=='PrivateSubnet1Id'].OutputValue" --output text) \
    PrivateSubnet2Id=$(aws cloudformation describe-stacks --stack-name ecs-network-stack --query "Stacks[0].Outputs[?OutputKey=='PrivateSubnet2Id'].OutputValue" --output text)
```

## Project Structure

```
ecsworkshop-infra/
├── templates/
│   ├── network-stack.cf.yml    # VPC and networking resources
│   └── ecs-stack.cf.yml        # ECS cluster and application resources
├── .github/
│   └── workflows/
│       └── deploy.yml          # CI/CD pipeline
└── README.md                   # This file
```

## Outputs

After successful deployment, the ECS stack provides:

- **ALB DNS Name**: Public endpoint for accessing the application
- **ECR Repository URI**: Location for pushing Docker images

## Cost Optimization

- **Fargate**: Pay only for running containers
- **NAT Gateway**: Single NAT Gateway for cost efficiency
- **Log Retention**: 7-day retention to minimize storage costs
- **Resource Sizing**: Minimal CPU/memory allocation (256 CPU, 512 MB memory)

## Security Best Practices

- Private subnets for application workloads
- Security groups with least privilege access
- IAM roles with minimal required permissions
- ECR vulnerability scanning enabled
- No direct internet access for ECS tasks

## Monitoring and Logging

- CloudWatch log groups for each service
- Application Load Balancer access logs
- ECS service and task metrics
- Container insights for detailed monitoring

## Troubleshooting

### Common Issues

- **Network Stack Fails**: Check CloudFormation events for CIDR conflicts or YAML syntax errors
- **ECS Stack Fails**: Ensure network stack is complete and verify IAM permissions
- **GitHub Actions Fails**: Verify AWS credentials are properly configured in repository secrets
- **Tasks Not Starting**: Check that Docker images exist in ECR repository

### Verification Steps

1. Confirm VPC and subnets are created in AWS Console
2. Verify ECS cluster exists with services defined
3. Check ALB is running and target groups are healthy
4. Validate ECR repository is accessible

## Application Deployment

To deploy the two-tier application (frontend and backend) on this ECS cluster, follow the instructions in the companion repository:

**📦 [ECS Workshop - Application Repository](https://github.com/LigeshK/aws-ecs-demoapp.git)**

This repository includes Dockerized frontend and backend applications, along with CI/CD pipelines for building, pushing to Amazon ECR, and deploying to the ECS cluster provisioned by this infrastructure.
