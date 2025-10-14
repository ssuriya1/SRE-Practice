
# Self-Healing Automation on AWS with Terraform & Locust

This beginner-friendly guide will help you set up a self-healing web server infrastructure on AWS using Terraform, CloudWatch, Auto Scaling, and Locust for traffic simulation.

---

## Overview
You will:
- Deploy auto-scaling web servers with Terraform
- Monitor and replace unhealthy EC2 instances automatically
- Simulate traffic spikes using Locust
- (Optional) Use AWS Lambda for automated recovery

---

## Step 1: Prepare Your Environment

### 1.1 Install Required Tools

**Terraform** (macOS):
```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
```

**AWS CLI** (macOS):
```bash
brew install awscli
aws --version
```

**Locust (Traffic Simulator):**
```bash
pip install locust
```

---

### 1.2 Configure AWS Credentials
Run:
```bash
aws configure
```
Enter your AWS Access Key, Secret Key, region (e.g. `us-east-1`), and output format (`json`).

---

## Step 2: Set Up Your Project Folder

Create a new folder for your Terraform project:
```bash
mkdir self-healing-terraform
cd self-healing-terraform
```

---

## Step 3: Create Locust Traffic Simulation File

Create a file named `locustfile.py`:
```python
from locust import HttpUser, task

class LoadTest(HttpUser):
        @task
        def index(self):
                self.client.get("/")
```

---

## Step 4: Create Lambda Function (Optional)

Create a folder and file for Lambda:
```bash
mkdir lambda
cd lambda
touch lambda_function.py
```

Paste this code into `lambda_function.py`:
```python
import boto3

ec2 = boto3.client('ec2')
autoscaling = boto3.client('autoscaling')

def lambda_handler(event, context):
        instance_id = event['detail']['instance-id']
        response = ec2.describe_instance_status(InstanceIds=[instance_id])
        status = response['InstanceStatuses'][0]['InstanceState']['Name']
        if status in ["stopped", "terminated"]:
                print(f"Instance {instance_id} is down. Replacing...")
                asg_response = autoscaling.describe_auto_scaling_instances(InstanceIds=[instance_id])
                asg_name = asg_response['AutoScalingInstances'][0]['AutoScalingGroupName']
                autoscaling.terminate_instance_in_auto_scaling_group(
                        InstanceId=instance_id,
                        ShouldDecrementDesiredCapacity=False
                )
                print(f"Instance {instance_id} replaced in ASG: {asg_name}")
        return {"statusCode": 200, "body": "Self-Healing Completed!"}
```

Zip the Lambda function:
```bash
zip lambda_function.zip lambda_function.py
cd .. # Go back to project root
```

---

## Step 5: Write Terraform Files

Create the main Terraform files:
```bash
touch main.tf output.tf variable.tf
```

### 5.1 main.tf (Infrastructure Definition)
Paste the following into `main.tf`:
```hcl
provider "aws" {
    region = "us-east-1"
}

data "aws_vpc" "default" {
    default = true
}

data "aws_subnets" "public" {
    filter {
        name   = "vpc-id"
        values = [data.aws_vpc.default.id]
    }
}

resource "aws_iam_role" "lambda_exec" {
    name = "lambda_exec_role"
    assume_role_policy = jsonencode({
        Version = "2012-10-17"
        Statement = [{
            Action = "sts:AssumeRole"
            Effect = "Allow"
            Principal = { Service = "lambda.amazonaws.com" }
        }]
    })
}

resource "aws_launch_configuration" "web_lc" {
    name          = "web-lc"
    image_id      = "ami-0360c520857e3138f" # Change to your region's Ubuntu AMI
    instance_type = "t2.micro"
    security_groups = [aws_security_group.web_sg.id]
    user_data = <<-EOF
        #!/bin/bash
        apt update -y
        apt install -y apache2
        systemctl start apache2
        systemctl enable apache2
    EOF
}

resource "aws_autoscaling_group" "web_asg" {
    desired_capacity     = 2
    min_size             = 1
    max_size             = 3
    vpc_zone_identifier  = data.aws_subnets.public.ids
    launch_configuration = aws_launch_configuration.web_lc.id
    health_check_type    = "EC2"
    health_check_grace_period = 300
    force_delete         = true
    # Add a Name tag so instances show up with a name in the AWS console
    tag {
        key                 = "Name"
        value               = "web-server-instance"
        propagate_at_launch = true
    }
}

resource "aws_security_group" "web_sg" {
    name        = "web-sg"
    description = "Allow inbound HTTP and SSH traffic"
    vpc_id      = data.aws_vpc.default.id
    ingress {
        from_port   = 80
        to_port     = 80
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
        from_port   = 22
        to_port     = 22
        protocol    = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }
}

resource "aws_lb" "web_alb" {
    name               = "web-load-balancer"
    internal           = false
    load_balancer_type = "application"
    security_groups    = [aws_security_group.web_sg.id]
    subnets            = data.aws_subnets.public.ids
    enable_deletion_protection     = false
    enable_cross_zone_load_balancing = true
    idle_timeout       = 60
}

resource "aws_lb_target_group" "web_tg" {
    name     = "web-target-group"
    vpc_id   = data.aws_vpc.default.id
    port     = 80
    protocol = "HTTP"
}

resource "aws_lb_listener" "web_listener" {
    load_balancer_arn = aws_lb.web_alb.arn
    port              = 80
    protocol          = "HTTP"
    default_action {
        type = "fixed-response"
        fixed_response {
            status_code  = 200
            content_type = "text/plain"
            message_body = "OK"
        }
    }
}

resource "aws_autoscaling_attachment" "web_asg_attachment" {
    autoscaling_group_name = aws_autoscaling_group.web_asg.id
    lb_target_group_arn    = aws_lb_target_group.web_tg.arn
}

resource "aws_autoscaling_policy" "scale_out" {
    name                   = "scale-out-policy"
    scaling_adjustment     = 1
    adjustment_type        = "ChangeInCapacity"
    cooldown               = 300
    autoscaling_group_name = aws_autoscaling_group.web_asg.name
}

resource "aws_cloudwatch_metric_alarm" "high_cpu" {
    alarm_name          = "high-cpu-alarm"
    comparison_operator = "GreaterThanThreshold"
    evaluation_periods  = 1
    metric_name         = "CPUUtilization"
    namespace           = "AWS/EC2"
    period              = 300
    statistic           = "Average"
    threshold           = 80
    alarm_description   = "Triggered when CPU usage is above 80%"
    insufficient_data_actions = []
    alarm_actions = [aws_autoscaling_policy.scale_out.arn]
    dimensions = {
        AutoScalingGroupName = aws_autoscaling_group.web_asg.name
    }
}
```

### 5.2 output.tf
```hcl
output "alb_dns_name" {
    value = aws_lb.web_alb.dns_name
}
```

### 5.3 variable.tf
```hcl
variable "region" {
    default = "us-east-2"
}

variable "instance_type" {
    default = "t2.micro"
}
```

---

## Step 6: Deploy Infrastructure

Run these commands in your project folder:
```bash
terraform init
terraform plan
terraform validate
terraform apply -auto-approve
```

---

## Step 7: Simulate Traffic with Locust

Start Locust:
```bash
locust -f locustfile.py
```
Open [http://0.0.0.0:8089](http://0.0.0.0:8089) in your browser. Enter the Load Balancer DNS name (from AWS Console or Terraform output) as the host.

To stop Locust, press `Ctrl+C` in the terminal.

---

## Step 8: Test Self-Healing

Stop an EC2 instance to trigger self-healing:
```bash
aws ec2 stop-instances --instance-ids <your-instance-id>
```
Auto Scaling will replace the unhealthy instance automatically.

---

## Step 9: Cleanup Resources

To remove all resources:
```bash
terraform destroy -auto-approve
```

---


## Instance Naming in AWS Console

To ensure your EC2 instances launched by the Auto Scaling Group have a visible name in the AWS console, the `Name` tag must be set as shown above. This makes it easier to identify and manage your instances.

## Summary

You have successfully deployed a self-healing infrastructure on AWS using Terraform:
- Auto Scaling replaces unhealthy EC2 instances
- CloudWatch monitors health and CPU
- Locust simulates traffic
- (Optional) Lambda for automated recovery

---

**Congratulations!**
