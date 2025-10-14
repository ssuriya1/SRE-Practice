# Implementing Anomaly Detection for EC2 CPU Spikes using Terraform

This guide covers:

- Launching an EC2 instance and installing CloudWatch Agent
- Configuring CloudWatch metrics for CPU utilisation
- Enabling Anomaly Detection for high CPU spikes
- Simulating CPU stress using a load generator
- Observing and analysing anomalies in the CloudWatch Dashboard
- Send CPU Spike Alert to Email

## Step 1: Install Terraform and AWS CLI on Ubuntu

Ensure you have Terraform and AWS CLI installed and configured.

### 1.1 Install Terraform

```bash
sudo apt update && sudo apt install -y wget unzip
wget https://releases.hashicorp.com/terraform/1.7.0/terraform_1.7.0_linux_amd64.zip
unzip terraform_1.7.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/
terraform -v # Verify installation
```

### 1.2 Install AWS CLI

Reference: https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html

```bash
sudo apt install -y awscli
aws --version
aws configure
```

Provide:

- AWS Access Key
- AWS Secret Key
- Default region (e.g., us-east-1)
- Output format (json)

Create new key-pair (.pem or .ppk) from AWS console.

## Step 2: Create Terraform Configuration for EC2, CloudWatch and Alert

### 2.1 Create a New Terraform Project

```bash
mkdir ec2-anomaly-detection
cd ec2-anomaly-detection
```

### 2.2 Create the Terraform File (main.tf)

```hcl
provider "aws" {
  region = "us-east-1"
}

# SNS Topic for Email Alerts
resource "aws_sns_topic" "cpu_alerts" {
  name = "cpu-anomaly-alerts"
}

resource "aws_sns_topic_subscription" "email_subscription" {
  topic_arn = aws_sns_topic.cpu_alerts.arn
  protocol  = "email"
  endpoint  = "suriya.sundarrajan@cognizant.com" # Replace with your email
}

# Security Group for SSH access
resource "aws_security_group" "ec2_sg" {
  name_prefix = "ec2-anomaly-sg"

  ingress {
    from_port   = 22
    to_port     = 22
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

# EC2 Instance with CloudWatch Agent and Stress Tool
resource "aws_instance" "ec2_instance" {
  ami                    = "ami-0360c520857e3138f" # Replace with latest Amazon Linux AMI
  instance_type          = "t2.micro"
  key_name               = "test1" # Replace with your key pair
  vpc_security_group_ids = [aws_security_group.ec2_sg.id]

  user_data = <<-EOF
    #!/bin/bash
    sudo apt update -y
    sudo apt install -y stress

    # Install CloudWatch Agent
    wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
    sudo dpkg -i amazon-cloudwatch-agent.deb
    sudo systemctl enable amazon-cloudwatch-agent
    sudo systemctl start amazon-cloudwatch-agent
  EOF

  tags = {
    Name = "Anomaly-Detection-EC2"
  }
}

# Anomaly Detection for CPU Spikes
resource "aws_cloudwatch_metric_alarm" "cpu_anomaly" {
  alarm_name          = "CPU_Anomaly_Detection"
  comparison_operator = "GreaterThanUpperThreshold"
  evaluation_periods  = 2
  threshold_metric_id = "ad1"
  alarm_description   = "Triggers when CPU spikes beyond anomaly threshold"
  actions_enabled     = true
  alarm_actions       = [aws_sns_topic.cpu_alerts.arn]

  metric_query {
    id          = "m1"
    return_data = true
    metric {
      metric_name = "CPUUtilization"
      namespace   = "AWS/EC2"
      period      = 60
      stat        = "Average"
      dimensions = {
        InstanceId = aws_instance.ec2_instance.id
      }
    }
  }

  metric_query {
    id          = "ad1"
    return_data = true
    expression  = "ANOMALY_DETECTION_BAND(m1, 2)"
  }
}

# CPU Utilization > 75% Alarm (Sends Email)
resource "aws_cloudwatch_metric_alarm" "cpu_threshold" {
  alarm_name          = "CPU_Threshold_Exceeded"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  threshold           = 75.0
  alarm_description   = "Triggers when CPU utilization exceeds 75%"
  actions_enabled     = true
  alarm_actions       = [aws_sns_topic.cpu_alerts.arn]

  metric_name = "CPUUtilization"
  namespace   = "AWS/EC2"
  period      = 60
  statistic   = "Average"
  dimensions = {
    InstanceId = aws_instance.ec2_instance.id
  }
}

# Output instance details
output "instance_id" {
  value = aws_instance.ec2_instance.id
}

output "instance_public_ip" {
  value = aws_instance.ec2_instance.public_ip
}

output "ssh_command" {
  value = "ssh -i your-key.pem ubuntu@${aws_instance.ec2_instance.public_ip}"
}
```

### Initialize and Deploy Terraform

```bash
terraform init
terraform plan
terraform validate
terraform apply -auto-approve

# Note the output values:
# - instance_id: Your EC2 instance ID
# - instance_public_ip: Public IP address
# - ssh_command: Ready-to-use SSH command
```

## Step 3: Install and Configure CloudWatch Agent

### SSH into the EC2 Instance

```bash
chmod 400 "key.pem"
ssh -i your-key.pem ubuntu@<EC2-Public-IP>
```

**If SSH connection fails:**

```bash
# Wait 2-3 minutes for instance to fully boot
# Check instance status
aws ec2 describe-instances --instance-ids <INSTANCE-ID>

# Try SSH with verbose output
ssh -v -i your-key.pem ubuntu@<EC2-Public-IP>
```

### Install CloudWatch Agent (if automated provisioning fails)

```bash
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb
```

### Configure CloudWatch Agent

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```

Or create a basic config file manually:

```bash
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json > /dev/null <<EOF
{
  "metrics": {
    "namespace": "CWAgent",
    "metrics_collected": {
      "cpu": {
        "measurement": [
          "cpu_usage_idle",
          "cpu_usage_iowait",
          "cpu_usage_user",
          "cpu_usage_system"
        ],
        "metrics_collection_interval": 60
      }
    }
  }
}
EOF
```

### Start and Enable the Agent

```bash
sudo systemctl start amazon-cloudwatch-agent
sudo systemctl enable amazon-cloudwatch-agent
```

### Verify CloudWatch Agent is Running

```bash
sudo systemctl status amazon-cloudwatch-agent
```

## Step 4: Simulate CPU Load

### 4.1 SSH into the EC2 Instance

```bash
ssh -i your-key.pem ubuntu@<EC2-Public-IP>
```

### 4.2 Install and Run CPU Stress Test

```bash
# Install stress tool
sudo apt update && sudo apt install -y stress

# Verify installation
stress --version

# Run CPU stress test (2 cores for 5 minutes)
sudo stress --cpu 2 --timeout 300

# Or use all available cores for 10 minutes
sudo stress --cpu $(nproc) --timeout 600
```

## Step 5: Cleanup (Destroy Resources)

```bash
terraform destroy -auto-approve
```

### Reference: https://github.com/AshishSharmaPrivate/ChaosLab/blob/main/Chaos-Engineering-Validation/CPUStress_Handson/cpu_stress.txt
