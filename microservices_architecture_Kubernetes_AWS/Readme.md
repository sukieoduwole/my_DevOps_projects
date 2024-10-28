# Microservices Architecture with Kubernetes on AWS

This project demonstrates deploying a microservices-based architecture on AWS using Kubernetes (EKS), Terraform, Auto Scaling, and IAM. This guide includes a complete setup, from configuring AWS infrastructure to provisioning a Kubernetes cluster, setting up worker nodes, and enabling scaling and monitoring.

### Project Overview

1. Infrastructure as Code (IaC): Automate the setup of AWS resources with Terraform.
2. Kubernetes (EKS): Deploy and manage microservices using AWS EKS.
3. Auto Scaling: Configure worker nodes to scale based on demand.
4. IAM Role Management: Control permissions for accessing AWS services.

### Prerequisites

1. AWS Account with IAM user credentials.
2. Terraform installed (Download Terraform).
3. AWS CLI installed and configured (AWS CLI Installation Guide).
4. kubectl installed to manage EKS (Install kubectl).
5. Helm installed for Kubernetes package management (Helm Installation Guide).

### Project Structure

- `provider.tf`: AWS provider configuration.
- `vpc.tf`: Virtual Private Cloud (VPC) setup with subnets, route tables, and an internet gateway.
- `eks.tf`: EKS cluster setup with IAM roles.
- `workers.tf`: Worker nodes configuration, including Auto Scaling and IAM policies.

### Step-by-Step Setup

**Step 1: AWS Provider Configuration** `provider.tf`
1. Create a new directory for your project and navigate into it:
>   
    mkdir aws-eks-microservices
    cd aws-eks-microservices

2. Define the AWS provider and specify your region.

>
    provider "aws" {
      region = "us-east-1"  # Update to your desired AWS region
    }

**Step 2: VPC and Networking Resources** `vpc.tf`

A Virtual Private Cloud (VPC) is essential for isolating the infrastructure. We’ll define subnets across two Availability Zones to ensure high availability, and add an internet gateway and a public route table.

1. Define the VPC with CIDR `10.0.0.0/16`
2. Add Subnets in two different Availability Zones.
3. Add an Internet Gateway to allow outbound internet access.
4. Define a Route Table to route internet traffic.

>
    resource "aws_vpc" "main_vpc" {
    cidr_block = "10.0.0.0/16"
    }

    resource "aws_subnet" "public_subnet_1" {
    vpc_id            = aws_vpc.main_vpc.id
    cidr_block        = "10.0.1.0/24"
    availability_zone = "us-east-1a"
    }

    resource "aws_subnet" "public_subnet_2" {
    vpc_id            = aws_vpc.main_vpc.id
    cidr_block        = "10.0.3.0/24"
    availability_zone = "us-east-1b"
    }

    resource "aws_internet_gateway" "main_igw" {
    vpc_id = aws_vpc.main_vpc.id
    }

    resource "aws_route_table" "public_route" {
    vpc_id = aws_vpc.main_vpc.id
    route {
        cidr_block = "0.0.0.0/0"
        gateway_id = aws_internet_gateway.main_igw.id
    }
    }

**Step 3: Set Up the EKS Cluster** `eks.tf`
Create the EKS cluster and the necessary IAM role.

1. Create an IAM Role for EKS to allow Kubernetes to manage AWS resources.
2. Define the EKS Cluster.

>
    resource "aws_iam_role" "eks_role" {
    name = "eks-role"

    assume_role_policy = jsonencode({
        "Version": "2012-10-17",
        "Statement": [{
        "Action": "sts:AssumeRole",
        "Effect": "Allow",
        "Principal": {
            "Service": "eks.amazonaws.com"
        }
        }]
    })
    }

    resource "aws_eks_cluster" "main_eks_cluster" {
    name     = "microservices-eks-cluster"
    role_arn = aws_iam_role.eks_role.arn

    vpc_config {
        subnet_ids = [
        aws_subnet.public_subnet_1.id,
        aws_subnet.public_subnet_2.id
        ]
    }
    }

**Step 4: Configure Worker Nodes with Launch Template** `workers.tf`
Set up worker nodes with a Launch Template and Auto Scaling Group. Ensure to use an EKS-optimized AMI for the correct region.

1. Create IAM Role for Worker Nodes.
2. Attach Policies for EKS, EC2 Registry, and CNI.
3. Create Instance Profile for Worker Nodes.
4. Define the Launch Template with an EKS-optimized AMI.
5. Create Auto Scaling Group for Worker Nodes.

>
    # IAM Role for Worker Nodes
    resource "aws_iam_role" "worker_role" {
    name = "eks-worker-role"

    assume_role_policy = jsonencode({
        "Version": "2012-10-17",
        "Statement": [{
        "Action": "sts:AssumeRole",
        "Effect": "Allow",
        "Principal": {
            "Service": "ec2.amazonaws.com"
        }
        }]
    })
    }

    # Attach necessary policies
    resource "aws_iam_role_policy_attachment" "worker_node_eks" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
    role       = aws_iam_role.worker_role.name
    }

    resource "aws_iam_role_policy_attachment" "worker_node_ec2" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
    role       = aws_iam_role.worker_role.name
    }

    resource "aws_iam_role_policy_attachment" "worker_node_cni" {
    policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
    role       = aws_iam_role.worker_role.name
    }

    # Instance Profile for Worker Nodes
    resource "aws_iam_instance_profile" "worker_instance_profile" {
    name = "eks-worker-instance-profile"
    role = aws_iam_role.worker_role.name
    }

    # Launch Template for Worker Nodes
    resource "aws_launch_template" "worker_template" {
    name_prefix   = "eks-worker-template"
    image_id      = "ami-xxxxxxxxxxxxxxxxx"  # Use a valid EKS-optimized AMI ID
    instance_type = "t3.medium"

    iam_instance_profile {
        name = aws_iam_instance_profile.worker_instance_profile.name
    }

    monitoring {
        enabled = true
    }

    tag_specifications {
        resource_type = "instance"
        tags = {
        Name = "eks-worker"
        }
    }
    }

    # Auto Scaling Group for Worker Nodes
    resource "aws_autoscaling_group" "worker_group" {
    vpc_zone_identifier = [
        aws_subnet.public_subnet_1.id,
        aws_subnet.public_subnet_2.id
    ]
    desired_capacity    = 2
    max_size            = 2
    min_size            = 1

    launch_template {
        id      = aws_launch_template.worker_template.id
        version = "$Latest"
    }

    health_check_type          = "EC2"
    health_check_grace_period  = 300
    }

**Step 5: Deploy with Terraform**
1. Initialize Terraform to download necessary providers:
>
    terraform init

2. Check the Execution Plan to verify resources:
>
    terraform plan

3. Run Terraform Plan to check resources to be created:
>
    terraform apply

3. Apply Terraform Configuration:
>
    terraform apply

### Accessing and Using the EKS Cluster

After provisioning, you can connect to the EKS cluster using `kubectl` to deploy and manage applications.

1. Configure kubectl with the cluster context:
>
    aws eks --region us-east-1 update-kubeconfig --name microservices-eks-cluster

2. Verify the Cluster by listing nodes:
>
    kubectl get nodes

### Notes
Ensure the AMI ID for the worker nodes is an EKS-optimized AMI specific to your region.
Use kubectl to manage and deploy applications on the EKS cluster after provisioning.

### Conclusion

This setup creates a scalable Kubernetes-based architecture on AWS that is ready for deploying and managing microservices. You can extend this setup to include CI/CD pipelines, monitoring, and logging for comprehensive DevOps workflows.