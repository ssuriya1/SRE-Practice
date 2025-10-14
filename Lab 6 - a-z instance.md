# Chaos Experiment - Instance Start and Stop Hands-on

## Objective: Resiliency against an AZ failure

The purpose of this experiment is to demonstrate infrastructure resiliency against an Availability Zone (AZ) failure using Chaos Engineering principles.

- Deploy two EC2 instances in separate AZs (us-east-2a and us-east-2b)
- Place an Application Load Balancer (ALB) in front of them
- Simulate a failure by stopping one instance using AWS Fault Injection Simulator (FIS)
- Observe how traffic automatically shifts to the healthy instance in the other AZ

This validates that the system can tolerate AZ-level disruptions without downtime.

## Architecture Overview

### Components created with Terraform:

- **VPC & Subnets** → Default VPC with subnets across AZs
- **Security Group** → Allows SSH (22) and HTTP (80)
- **EC2 Instances**:
  - Instance A → us-east-2a (serves "Hello from Instance A")
  - Instance B → us-east-2b (serves "Hello from Instance B")
- **Application Load Balancer (ALB)** → Routes traffic across instances with health checks
- **IAM Role + Policy for FIS** → Allows EC2 start/stop actions
- **FIS Experiment Template** → Stops one instance to simulate AZ failure

## Step-by-Step Implementation

### Step 1: Terraform Setup

1. Install Terraform and AWS CLI
2. Create a working directory
3. Add the Terraform code (main.tf)

```tf
provider "aws" {
  region = "us-east-2"
}

# Create a new VPC
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  enable_dns_support = true
  enable_dns_hostnames = true
  tags = {
    Name = "chaos-demo-vpc"
  }
}

# Create an Internet Gateway
resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id
  tags = {
    Name = "chaos-demo-igw"
  }
}

# Create two public subnets in different AZs
resource "aws_subnet" "public_a" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-2a"
  map_public_ip_on_launch = true
  tags = {
    Name = "chaos-demo-public-a"
  }
}

resource "aws_subnet" "public_b" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.2.0/24"
  availability_zone       = "us-east-2b"
  map_public_ip_on_launch = true
  tags = {
    Name = "chaos-demo-public-b"
  }
}

# Create a route table and associate with public subnets
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }
  tags = {
    Name = "chaos-demo-public-rt"
  }
}

resource "aws_route_table_association" "a" {
  subnet_id      = aws_subnet.public_a.id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "b" {
  subnet_id      = aws_subnet.public_b.id
  route_table_id = aws_route_table.public.id
}

resource "aws_security_group" "demo_sg" {
  name        = "chaos-demo-sg"
  description = "Allow SSH and HTTP"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}



# Instance A in us-east-2a
resource "aws_instance" "instance_a" {
  ami           = "ami-0cfde0ea8edd312d4"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public_a.id
  availability_zone = "us-east-2a"
  vpc_security_group_ids = [aws_security_group.demo_sg.id]
  user_data = <<-EOF
              #!/bin/bash
              sudo apt update -y
              sudo apt install -y nginx
              echo "Hello from Instance A (us-east-2a)" > /var/www/html/index.html
              sudo systemctl start nginx
              sudo systemctl enable nginx
              EOF
  tags = {
    Name = "Instance-A"
  }
}

# Instance B in us-east-2b
resource "aws_instance" "instance_b" {
  ami           = "ami-0cfde0ea8edd312d4"
  instance_type = "t2.micro"
  subnet_id     = aws_subnet.public_b.id
  availability_zone = "us-east-2b"
  vpc_security_group_ids = [aws_security_group.demo_sg.id]
  user_data = <<-EOF
              #!/bin/bash
              sudo apt update -y
              sudo apt install -y nginx
              echo "Hello from Instance B (us-east-2b)" > /var/www/html/index.html
              sudo systemctl enable nginx
              sudo systemctl start nginx
              EOF
  tags = {
    Name = "Instance-B"
  }
}

# Create Load Balancer
resource "aws_lb" "app_lb" {
  name               = "chaos-demo-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.demo_sg.id]
  subnets            = [aws_subnet.public_a.id, aws_subnet.public_b.id]
}

resource "aws_lb_target_group" "app_tg" {
  name     = "chaos-demo-tg"
  port     = 80
  protocol = "HTTP"
  vpc_id   = aws_vpc.main.id
  health_check {
    path = "/"
    port = "80"
  }
}

resource "aws_lb_listener" "app_listener" {
  load_balancer_arn = aws_lb.app_lb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app_tg.arn
  }
}

# Register instances in Target Group
resource "aws_lb_target_group_attachment" "a" {
  target_group_arn = aws_lb_target_group.app_tg.arn
  target_id        = aws_instance.instance_a.id
  port             = 80
}

resource "aws_lb_target_group_attachment" "b" {
  target_group_arn = aws_lb_target_group.app_tg.arn
  target_id        = aws_instance.instance_b.id
  port             = 80
}

# IAM Role for FIS
resource "aws_iam_role" "fis_role" {
  name = "ChaosFISRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = {
        Service = "fis.amazonaws.com"
      }
      Action = "sts:AssumeRole"
    }]
  })
}

# Attach Policy to FIS Role
resource "aws_iam_role_policy" "fis_policy" {
  name = "ChaosFISPolicy"
  role = aws_iam_role.fis_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = [
        "ec2:StopInstances",
        "ec2:StartInstances",
        "ec2:DescribeInstances"
      ]
      Resource = "*"
    }]
  })
}

# FIS Experiment to stop Instance A
resource "aws_fis_experiment_template" "chaos_experiment" {
  description = "Chaos Engineering Demo: Stop Instance A"
  role_arn    = aws_iam_role.fis_role.arn

  stop_condition {
    source = "none"
  }

  target {
    name   = "TargetInstances"
    resource_type = "aws:ec2:instance"
    selection_mode = "ALL"
    resource_arns = [aws_instance.instance_a.arn]
  }

  action {
    name = "stop-instances"
    action_id = "aws:ec2:stop-instances"

    target {
      key   = "Instances"
      value = "TargetInstances"
    }
  }

  tags = {
    Name = "ChaosExperiment"
  }
}

output "load_balancer_dns" {
  value = aws_lb.app_lb.dns_name
}
```

### Step 2: Deploy Infrastructure

```bash
terraform init
terraform plan
terraform apply
```

## Expected Chaos Demo Flow

1. Visit ALB URL → Served by Instance B
2. Run FIS Experiment → Instance B stops
3. Refresh browser → ALB automatically shifts traffic to Instance A

## Navigate to AWS FIS (Console)

1. Login to your AWS Management Console
2. In the top search bar, type "Fault Injection Simulator" or "FIS"
3. Click on Fault Injection Simulator from the search results
4. You'll now land on the AWS FIS Dashboard

### Left panel options:
- **Experiment templates** → Pre-defined chaos experiments
- **Experiments** → Run and monitor chaos experiments
- **Actions** → Start experiments

## Steps to Check in Browser

1. Open your Load Balancer DNS URL (example: `http://<ALB-DNS>`)
2. You'll see Instance A (Hello from Instance A)
3. **Start the FIS Experiment** (10 min duration)
4. Instance A will be stopped
5. **Refresh the Browser**
6. Now you'll see Instance B page (Hello from Instance B)

## Chaos Hands-on Summary

### Objective
- Demonstrate resilience of infrastructure by running a controlled failure experiment
- Show how traffic seamlessly shifts between instances when one fails

### Setup
- Two EC2 instances (Instance A in us-east-2a, Instance B in us-east-2b)
- Both registered under an Application Load Balancer (ALB)
- Both instances serve traffic simultaneously with health checks

### Experiment
- Used AWS Fault Injection Simulator (FIS) to stop one instance
- **Observed**:
  - Instance stopped
  - ALB automatically routed traffic to healthy instance
  - After FIS completion, system remained available without downtime

### Outcome
- Demonstrated high availability – application stayed online even when one instance failed
- Showed automatic traffic failover using ALB health checks
- Verified chaos experiment success – proved system can tolerate instance failure

### Key Learning
- Chaos Engineering is not about breaking things, but about building confidence that the system can recover gracefully
- With automation (FIS + ALB + EC2), the system becomes self-healing and resilient

## Visual Representation

```
┌──────────────────────┐
│ Application LB       │
│ (DNS entry point)    │
└─────────┬────────────┘
          │
┌─────────┼─────────────┐
│         │             │
▼         ▼             ▼
┌─────────────────┐ ┌─────────────────┐
│ Instance A      │ │ Instance B      │
│ us-east-2a      │ │ us-east-2b      │
│ "Hello from A"  │ │ "Hello from B"  │
└─────────────────┘ └─────────────────┘
         │                     │
         │ FIS Stops Instance  │
         │                     │
         X                     ✓

ALB automatically shifts traffic to healthy instance
→ System stays online
```

## Cleanup

```bash
terraform destroy
```