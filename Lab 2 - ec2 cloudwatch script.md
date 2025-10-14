## Prerequisites

1. **Install Terraform**
   Download and install Terraform from [terraform.io](https://www.terraform.io/downloads.html).

2. **Install AWS CLI**
   Download and install AWS CLI from [aws.amazon.com/cli](https://aws.amazon.com/cli/).

3. **Configure AWS Credentials**
   - Go to AWS Console → IAM → Users → Security credentials.
   - Create an access key (Access Key ID & Secret Access Key).
   - Run in your terminal:
     ```sh
     aws configure
     ```
   - Enter your AWS Access Key, Secret Key, region (e.g., `us-east-2`), and output format (e.g., `json`).

---

## 1. Create Project Directory

Open your terminal and run:

```sh
mkdir ec2_monitoring
cd ec2_monitoring
```

---

## 2. Create Terraform Configuration

Create a file named `main.tf`:

```sh
touch main.tf
```

Open `main.tf` in your editor and copy-paste the following script:

```hcl
provider "aws" {
  region = "us-east-2"
}

resource "tls_private_key" "ec2_key" {
  algorithm = "RSA"
  rsa_bits = 4096
}

resource "aws_key_pair" "ec2_key" {
  key_name   = "ec2-key"
  public_key = tls_private_key.ec2_key.public_key_openssh
}

resource "local_file" "private_key" {
  content        = tls_private_key.ec2_key.private_key_pem
  filename       = "ec2-key.pem"
  file_permission = "0400"
}

resource "aws_security_group" "ec2_sg" {
  name        = "ec2-sg"
  description = "Allow SSH and HTTP"

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

resource "aws_iam_role" "cw_agent_role" {
  name = "cloudwatch-agent-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Effect = "Allow",
      Principal = {
        Service = "ec2.amazonaws.com"
      },
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "cw_attach" {
  role       = aws_iam_role.cw_agent_role.name
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
}

resource "aws_iam_instance_profile" "cw_instance_profile" {
  name = "cloudwatch-agent-profile"
  role = aws_iam_role.cw_agent_role.name
}

resource "aws_instance" "example" {
  ami                    = "ami-0cb91c7de36eed2cb" # Ubuntu 22.04 LTS in us-east-2
  instance_type          = "t2.micro"
  key_name               = aws_key_pair.ec2_key.key_name
  monitoring             = true
  vpc_security_group_ids = [aws_security_group.ec2_sg.id]
  iam_instance_profile   = aws_iam_instance_profile.cw_instance_profile.name

  user_data = <<-EOF
#!/bin/bash
sudo apt update
sudo apt install -y awscli unzip

# Download and install CloudWatch Agent
curl https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb -o amazon-cloudwatch-agent.deb
sudo dpkg -i -E ./amazon-cloudwatch-agent.deb

# Create config directory
sudo mkdir -p /opt/aws/amazon-cloudwatch-agent/etc

# Create CloudWatch config file
cat <<EOC > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root"
  },
  "metrics": {
    "append_dimensions": {
      "InstanceId": "$${aws:InstanceId}"
    },
    "metrics_collected": {
      "cpu": {
        "measurement": [
          "cpu_usage_idle",
          "cpu_usage_iowait",
          "cpu_usage_user"
        ],
        "metrics_collection_interval": 60
      },
      "mem": {
        "measurement": [
          "mem_used_percent"
        ],
        "metrics_collection_interval": 60
      }
    }
  }
}
EOC

# Start the CloudWatch Agent
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
EOF

  tags = {
    Name = "cloudwatch-monitored-instance"
  }
}
```

### Debugging changes

- **Error on AMI** - Change the AMI ID and make sure you are using Ubuntu image.

---

## 3. Initialize Terraform

In the `ec2_monitoring` directory, run:

```sh
terraform init
```

This downloads the required plugins.

---

## 4. Validate the Configuration

Run:

```sh
terraform validate
```

This checks for errors in your configuration.

---

## 5. Review the Execution Plan

Run:

```sh
terraform plan
```

This shows what resources will be created.

---

## 6. Apply the Configuration

Run:

```sh
terraform apply -auto-approve
```

This creates the EC2 instance and sets up CloudWatch monitoring.

---

## 7. Check CloudWatch Metrics

- Go to AWS Console → CloudWatch → Metrics → All Metrics → CWAgent.
- You should see metrics like `mem_used_percent`, `cpu_usage_user`, etc.

---

## 8. Validate CloudWatch Agent on EC2

**Connect to your EC2 instance via SSH:**

```sh
ssh -i ec2-key.pem ubuntu@<EC2_PUBLIC_IP>
```

Replace `<EC2_PUBLIC_IP>` with your instance's public IP.

**Check agent status:**

```sh
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
```

Expected output:

```json
{
  "status": "running",
  "starttime": "...",
  "configstatus": "configured",
  ...
}
```

---

## 9. Troubleshooting & Manual Start

**Check config file:**

```sh
cat /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

**Start agent manually:**

```sh
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
```

**If needed, update InstanceId in config:**

```sh
sudo sed -i 's/"InstanceId": ""/"InstanceId": "${aws:InstanceId}"/' /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

**Start the service:**

```sh
sudo systemctl start amazon-cloudwatch-agent
sudo systemctl status amazon-cloudwatch-agent
```

---

You have now deployed an EC2 instance with CloudWatch monitoring using Terraform. If you have any issues, check the AWS Console for error messages or review your configuration.
