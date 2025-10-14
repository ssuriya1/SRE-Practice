# Hands-on Lab 7: Active-Passive Setups - Incident Management and Fault Tolerance

## Overview

This lab demonstrates how to implement redundant infrastructure using Terraform to provision active-passive setups and validate automatic traffic redirection. We'll create a fault-tolerant system using AWS services including EC2 instances, Route 53 health checks, and DNS failover mechanisms.

## Learning Objectives

- Understand active-passive architecture patterns
- Implement automatic failover mechanisms
- Configure Route 53 health checks and DNS failover
- Test incident response and fault tolerance
- Practice infrastructure as code with Terraform

## Prerequisites

### Required Software

Ensure you have the following installed on your system:

1. **Terraform** → [Install Guide](https://learn.hashicorp.com/tutorials/terraform/install-cli)
2. **AWS CLI** → [Install Guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
3. **IAM User** with Route 53 & EC2 permissions
4. **SSH Key Pair** (.pem or .ppk file) to access instances

### Verification Commands

Check if Terraform is installed:
```bash
terraform -v
```

Check if AWS CLI is configured:
```bash
aws configure list
```

### AWS Configuration

To connect your machine to AWS account, you need access key and secret key:

1. Create access key from **IAM > Users > Create Access Key**
2. Configure AWS CLI:
```bash
aws configure
```

## Architecture Overview

### Active-Passive Setup (Failover Mechanism)

- **Primary (Active) Instance**: Handles all traffic initially
- **Secondary (Passive) Instance**: Standby instance that takes over during failures
- **Route 53 Health Checks**: Monitor primary instance health
- **Automatic Failover**: Traffic redirects to passive instance when active fails

### Components

1. **Redundant Infrastructure**: Two EC2 instances in different subnets
2. **Health Monitoring**: Route 53 health checks on active instance
3. **DNS Failover**: Automatic traffic redirection via Route 53
4. **Security**: Security groups for SSH and HTTP access

## Implementation Steps

### Step 1: Create Test Hosted Zone

Create a temporary private hosted zone in Route 53:

```bash
aws route53 create-hosted-zone --name "test.local.com" --caller-reference "test-$(date +%s)"
```

**Important**: Note the Hosted Zone ID from the output - you'll need it for the Terraform configuration (line 141 in main.tf).

### Step 2: Create Terraform Configuration

Create the main Terraform file:

```bash
sudo vi main.tf
```

Add the following configuration:

```hcl
# AWS Provider Configuration
provider "aws" {
  region = "us-east-2" # Change as needed
}

# Fetch VPC & Subnets Dynamically
data "aws_vpc" "default" {
  default = true
}

data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

# Fetch latest Amazon Linux 2 AMI
data "aws_ami" "latest_amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*"]
  }
}

# Create Security Group (Allow SSH & HTTP)
resource "aws_security_group" "allow_ssh" {
  vpc_id = data.aws_vpc.default.id
  name   = "allow_ssh"

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

# Launch Active EC2 Instance
resource "aws_instance" "active" {
  ami                    = data.aws_ami.latest_amazon_linux.id
  instance_type          = "t2.micro"
  subnet_id              = element(data.aws_subnets.default.ids, 0)
  vpc_security_group_ids = [aws_security_group.allow_ssh.id]
  
  tags = {
    Name = "Active-Instance"
  }
}

# Launch Passive EC2 Instance
resource "aws_instance" "passive" {
  ami                    = data.aws_ami.latest_amazon_linux.id
  instance_type          = "t2.micro"
  subnet_id              = element(data.aws_subnets.default.ids, 1)
  vpc_security_group_ids = [aws_security_group.allow_ssh.id]
  
  tags = {
    Name = "Passive-Instance"
  }
}

# Route 53 Hosted Zone (Replace with your Hosted Zone ID)
data "aws_route53_zone" "primary" {
  zone_id = "Z02145102I3UGRWQH4143"  # Replace with actual Hosted Zone ID
}

# Create Route 53 DNS Failover Record - Primary
resource "aws_route53_record" "failover_dns" {
  zone_id = data.aws_route53_zone.primary.zone_id
  name    = "test.local.com"
  type    = "A"

  failover_routing_policy {
    type = "PRIMARY"
  }

  set_identifier  = "active-instance"
  health_check_id = aws_route53_health_check.ec2.id
  records         = [aws_instance.active.public_ip]
  ttl             = 30
}

# Create Route 53 DNS Failover Record - Secondary
resource "aws_route53_record" "failover_dns_passive" {
  zone_id = data.aws_route53_zone.primary.zone_id
  name    = "test.local.com"
  type    = "A"

  failover_routing_policy {
    type = "SECONDARY"
  }

  set_identifier = "passive-instance"
  records        = [aws_instance.passive.public_ip]
  ttl            = 30
}

# Route53 Health Check for Active Instance
resource "aws_route53_health_check" "ec2" {
  ip_address        = aws_instance.active.public_ip
  port              = 22  # Change to 80 for HTTP monitoring
  type              = "TCP"  # Use HTTP if monitoring a web service
  failure_threshold = 3
  request_interval  = 30

  tags = {
    Name = "EC2-Health-Check"
  }
}

# Output Values
output "active_public_ip" {
  value = aws_instance.active.public_ip
}

output "active_instance_id" {
  value = aws_instance.active.id
}

output "passive_public_ip" {
  value = aws_instance.passive.public_ip
}

output "passive_instance_id" {
  value = aws_instance.passive.id
}
```

### Step 3: Deploy Infrastructure

Execute the following commands to create the infrastructure:

```bash
# Initialize Terraform
terraform init

# Review the execution plan
terraform plan

# Validate configuration
terraform validate

# Apply configuration
terraform apply -auto-approve
```

## Validation and Testing

### Initial Setup Verification

#### 1. List Route 53 Hosted Zones
```bash
aws route53 list-hosted-zones --query 'HostedZones[*].Name'
```

#### 2. Get Hosted Zone ID
```bash
aws route53 list-hosted-zones --query 'HostedZones[*].[Id, Name]'
```

#### 3. Check A Record Exists
```bash
aws route53 list-resource-record-sets --hosted-zone-id $(aws route53 list-hosted-zones --query "HostedZones[?Name=='test.local.'].Id" --output text)
```

#### 4. Test Active Instance SSH Access
```bash
ssh -i your-key.pem ec2-user@$(terraform output -raw active_public_ip)
```

If successful, your active instance is operational.

## Failure Simulation and Failover Testing

### Step 1: Simulate Primary Instance Failure

Stop the active instance to simulate a failure:

```bash
aws ec2 stop-instances --instance-ids $(terraform output -raw active_instance_id)
```

### Step 2: Verify Automatic Failover

After a few minutes, check if DNS fails over to the passive instance:

```bash
dig test.local.com
```

**Expected Output:**
```
;; ANSWER SECTION:
test.local.com. 30 IN A 18.116.38.222
```

The IP address should now point to the passive instance, indicating successful failover.

### Step 3: Restore Primary Instance

Restart the active instance:

```bash
aws ec2 start-instances --instance-ids $(terraform output -raw active_instance_id)
```

Verify traffic returns to the primary instance:

```bash
dig test.local.com
```

## Cleanup

### Destroy Infrastructure

Once testing is complete, clean up all resources:

```bash
terraform destroy -auto-approve
```

### Delete Hosted Zone

Navigate to Route 53 in AWS Console and manually delete the hosted zone if it still exists.

## Troubleshooting

### DNS Resolution Issues

#### 1. Check Hosted Zone Nameservers

```bash
aws route53 get-hosted-zone --id <hosted-zone-id> --query "DelegationSet.NameServers"
```

**Example Output:**
```json
[
  "ns-2048.awsdns-64.com",
  "ns-2049.awsdns-65.net",
  "ns-2050.awsdns-66.org",
  "ns-2051.awsdns-67.co.uk"
]
```

#### 2. Query Route 53 Directly

```bash
dig @ns-2048.awsdns-64.com test.local.com
```

**Expected Output:**
```
;; ANSWER SECTION:
test.local.com. 300 IN A 203.0.113.10
```

#### 3. Check Local DNS Configuration

If DNS resolution fails, check your local resolver configuration:

```bash
cat /etc/resolv.conf
```

If using public DNS (1.1.1.1 or 8.8.8.8), consider temporarily changing to AWS Route 53 nameservers:

```
nameserver 49.205.72.130
nameserver 183.82.243.66
```

#### 4. Final Verification

```bash
dig test.local.com
```

**Expected Output:**
```
;; ANSWER SECTION:
test.local.com. 300 IN A 203.0.113.10
```

## Common Issues and Solutions

### Issue 1: Health Check Failures
- **Cause**: Security group blocking health check traffic
- **Solution**: Ensure security group allows traffic on the monitored port

### Issue 2: Slow Failover
- **Cause**: High TTL values or health check intervals
- **Solution**: Reduce TTL to 30 seconds and health check interval to 30 seconds

### Issue 3: DNS Not Resolving
- **Cause**: Incorrect nameserver configuration
- **Solution**: Use Route 53 nameservers directly or check local DNS settings

## Key Concepts Learned

1. **Active-Passive Architecture**: Understanding standby systems and failover mechanisms
2. **Health Monitoring**: Implementing automated health checks for infrastructure
3. **DNS Failover**: Using Route 53 for automatic traffic redirection
4. **Infrastructure as Code**: Managing infrastructure through Terraform
5. **Incident Response**: Testing and validating failover procedures

## Best Practices

1. **Regular Testing**: Periodically test failover mechanisms
2. **Monitoring**: Implement comprehensive monitoring and alerting
3. **Documentation**: Maintain updated runbooks for incident response
4. **Automation**: Automate recovery procedures where possible
5. **Security**: Implement least-privilege access controls

## Conclusion

This lab demonstrates the implementation of a robust active-passive setup for incident management and fault tolerance. The combination of AWS services (EC2, Route 53) with Infrastructure as Code (Terraform) provides a scalable and maintainable solution for high availability systems.

The automatic failover mechanism ensures minimal downtime during incidents, while the health checks provide early detection of failures. This setup forms the foundation for more complex disaster recovery and business continuity strategies.