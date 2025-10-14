# Security Incident Simulation using AWS, Nginx and hey

Simulate a high-traffic event (like a DDoS attack), monitor the spike in requests, and respond by blocking the source IP.

## Prerequisite : Installation

- Terraform
- Aws cli
- Python
- hey

## Install hey in ubuntu, windows, Mac

### Mac:

- brew install hey
- hey - - version

### Ubuntu:

```bash
sudo apt update
sudo apt install golang-go -y
go install github.com/rakyll/hey@latest
sudo cp ~/go/bin/hey /usr/local/bin/
hey —version
```

### Windows:

- Install Chocolatey (if you haven't already)
- Run the following command to install Chocolatey: `Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.SecurityProtocolType]::Tls12; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))`

```bash
choco install hey
hey --version
```

Tracking HTTP Requests to EC2 in CloudWatch (via Nginx + CloudWatch Agent)

Send a high number of requests per second to your Nginx app hosted on EC2 to simulate a DDoS attackPython script to count requests per IP and block from specified threshold.

## Simulate a DDoS Using hey

1. Terraform script – Create `EC2` + Deploy `Nginx` + `CloudWatch Agent` + Python Script to automatic block the IP.

Create and use `vi main.tf file`.

```tf
provider "aws" {
  region = "us-east-2"
}

resource "aws_security_group" "web_sg" {
  name        = "web-allow-http-ssh"
  description = "Allow HTTP and SSH traffic"

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

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_instance" "web" {
  ami             = "ami-0cb91c7de36eed2cb" # Ubuntu
  instance_type   = "t2.micro"
  key_name        = "lab8"
  security_groups = [aws_security_group.web_sg.name]

  # Explicitly specify the subnet ID to resolve the previous error

  subnet_id = "subnet-0183e21ebb210c46d"

  user_data = <<-EOF
#!/bin/bash
sudo apt update -y
sudo apt install -y nginx

# Start and enable Nginx

sudo systemctl start nginx
sudo systemctl enable nginx

# Create directory for CloudWatch Agent config

mkdir -p /opt/aws/amazon-cloudwatch-agent/etc

# Write CloudWatch Agent config to collect Nginx access logs

cat <<EOC > /opt/aws/amazon-cloudwatch-agent/etc/config.json
{
"logs": {
"logs_collected": {
"files": {
"collect_list": [
{
"file_path": "/var/log/nginx/access.log",
"log_group_name": "nginx-access-log",
"log_stream_name": "{instance_id}"
}
]
}
}
}
}
EOC

# Install CloudWatch Agent

wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
dpkg -i amazon-cloudwatch-agent.deb

# Fetch and start the CloudWatch Agent configuration

/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
 -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json -s

# Create Python script for IP blocking based on traffic threshold

cat <<EOPY > /opt/block_high_traffic_ips.py
import re
from collections import Counter
import subprocess

LOG_FILE = "/var/log/nginx/access.log"
THRESHOLD = 1000 # Max allowed requests per IP (customize this value)

try:
with open(LOG_FILE, "r") as f:
logs = f.readlines()
except FileNotFoundError:
print(f"Error: Log file not found at {LOG_FILE}")
exit()

# Extract IP addresses from logs

ips = []
ip_pattern = re.compile(r'^(\d+\.\d+\.\d+\.\d+)')
for line in logs:
match = ip_pattern.match(line)
if match:
ips.append(match.group(1))

counter = Counter(ips)
blocked = []

# Check IPs against the threshold and block using iptables

for ip, count in counter.items():
if count > THRESHOLD: # Note: This requires the instance to have an IAM role # that allows modification of iptables (which is usually possible for the root user)
subprocess.call(["iptables", "-A", "INPUT", "-s", ip, "-j", "DROP"])
blocked.append(ip)

if blocked:
print("Blocked IPs:")
for ip in blocked:
print(ip)
else:
print("No IPs exceeded threshold.")
EOPY

# Make the script executable

chmod +x /opt/block_high_traffic_ips.py
EOF

  tags = {
    Name = "NginxBlockIPServer"
  }
}
```

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

2. Run Load Test from Your Machine

Before running the load from your machine, lets check and verify if nginx is accessible.

```
http://<public ip>:80
```

If not accessible login to ec2 machine and verify the installation on nginx:
systemctl status nginx sudo apt install -y nginx systemctl start nginx
systemctl enable nginx

Now, lets run below command from base machine to generate traffic.

hey -z 60s -c 100 http://<EC2-Public-IP>

What actually happens:
hey creates 100 virtual users that repeatedly send requests to your EC2 instance for 60 seconds.
The tool records:
Average response time
Requests per second (throughput)
Fastest/slowest response
Status code distribution (e.g., 200 OK, 500 error)
Error types, if any

Check the logs of nginx in aws instance:

```bash
tail -f /var/log/nginx/access.log
```

3.

SSH Into EC2 and Run IP Blocking Script

ssh -i your-ec2-key.pem ubuntu@<EC2-Public-IP>

# Run the script (it should print the blocked IP)

```bash
sudo python3 /opt/block_high_traffic_ips.py
```

Now, you won’t be able to access nginx as it has been blocked.
