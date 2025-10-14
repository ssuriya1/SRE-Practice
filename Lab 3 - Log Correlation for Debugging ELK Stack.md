# Step-by-Step Guide: Deploy Centralized Logging with ELK Stack using Terraform

---

## Prerequisites

- AWS account and AWS CLI configured
- Terraform installed
- AWS Key Pair (.pem or .ppk)

---

## 1. What You'll Build

You will launch two EC2 instances:

- **Node 1 (ELK Stack):** Runs Elasticsearch, Logstash, Kibana
- **Node 2 (App + Filebeat):** Runs a sample Node.js app and Filebeat

Logs from Node 2 will be sent to Node 1 and visualized in Kibana.

---

## 2. Prepare Your Terraform Script

Create a file named `main.tf` and copy the following configuration. Change `key_name` to match your AWS key pair name.

````hcl
provider "aws" {
    region = "us-east-1"
}

resource "aws_security_group" "elk_sg" {
    name        = "elk_sg"
    description = "Allow traffic for ELK"

    ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
    from_port   = 5601
    to_port     = 5601
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
    from_port   = 9200
    to_port     = 9200
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
    from_port   = 5044
    to_port     = 5044
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

resource "aws_instance" "elk_node" {
    ami           = "ami-0cb91c7de36eed2cb" # Ubuntu AMI
    instance_type = "t2.medium"
    key_name      = "elk" # change as per your key
    security_groups = [aws_security_group.elk_sg.name]
    tags = {
    Name = "elk-node"
    }
    user_data = <<-EOF
#!/bin/bash
sudo apt update -y
sudo apt install -y openjdk-11-jre curl

# Install Elasticsearch
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update -y && sudo apt install -y elasticsearch

# Configure Elasticsearch
echo "network.host: 0.0.0.0" | sudo tee -a /etc/elasticsearch/elasticsearch.yml
echo "discovery.type: single-node" | sudo tee -a /etc/elasticsearch/elasticsearch.yml
sudo systemctl enable elasticsearch && sudo systemctl start elasticsearch

# Install Kibana
sudo apt install -y kibana
echo "server.port: 5601" | sudo tee -a /etc/kibana/kibana.yml
echo "server.host: 0.0.0.0" | sudo tee -a /etc/kibana/kibana.yml
echo "elasticsearch.hosts: [\"http://localhost:9200\"]" | sudo tee -a /etc/kibana/kibana.yml
sudo systemctl enable kibana && sudo systemctl start kibana

# Install Logstash
sudo apt install -y logstash
echo '
input {
    beats {
    port => "5044"
    }
}
filter {
    json {
    source => "message"
    }
}
output {
    elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "logs-%{+YYYY.MM.dd}"
    }
}
' | sudo tee /etc/logstash/conf.d/logstash.conf
sudo systemctl enable logstash && sudo systemctl start logstash

# Centralized Logging with ELK Stack (Elasticsearch, Logstash, Kibana) using Terraform

---

## Prerequisites

- AWS account and AWS CLI configured
- Terraform installed
- AWS Key Pair (.pem or .ppk)

---

## What You'll Build

You will launch two EC2 instances:
- **Node 1 (ELK Stack):** Runs Elasticsearch, Logstash, Kibana
- **Node 2 (App + Filebeat):** Runs a sample Node.js app and Filebeat

Logs from Node 2 will be sent to Node 1 and visualized in Kibana.

---

## Step 1: Prepare Your Terraform Script

Create a file named `main.tf` and copy the following configuration. Change `key_name` to match your AWS key pair name.

```hcl
provider "aws" {
    region = "us-east-1"
}

resource "aws_security_group" "elk_sg" {
    name        = "elk_sg"
    description = "Allow traffic for ELK"

    ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
    from_port   = 5601
    to_port     = 5601
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
    from_port   = 9200
    to_port     = 9200
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    }
    ingress {
    from_port   = 5044
    to_port     = 5044
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

resource "aws_instance" "elk_node" {
    ami           = "ami-0360c520857e3138f" # Ubuntu AMI
    instance_type = "t2.medium"
    key_name      = "elk" # change as per your key
    security_groups = [aws_security_group.elk_sg.name]
    tags = {
    Name = "elk-node"
    }
    user_data = <<-EOF
#!/bin/bash
sudo apt update -y
sudo apt install -y openjdk-11-jre curl

# Install Elasticsearch
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update -y && sudo apt install -y elasticsearch

# Configure Elasticsearch
echo "network.host: 0.0.0.0" | sudo tee -a /etc/elasticsearch/elasticsearch.yml
echo "discovery.type: single-node" | sudo tee -a /etc/elasticsearch/elasticsearch.yml
sudo systemctl enable elasticsearch && sudo systemctl start elasticsearch

# Install Kibana
sudo apt install -y kibana
echo "server.port: 5601" | sudo tee -a /etc/kibana/kibana.yml
echo "server.host: 0.0.0.0" | sudo tee -a /etc/kibana/kibana.yml
echo "elasticsearch.hosts: [\"http://localhost:9200\"]" | sudo tee -a /etc/kibana/kibana.yml
sudo systemctl enable kibana && sudo systemctl start kibana

# Install Logstash
sudo apt install -y logstash
echo '
input {
    beats {
    port => "5044"
    }
}
filter {
    json {
    source => "message"
    }
}
output {
    elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "logs-%{+YYYY.MM.dd}"
    }
}
' | sudo tee /etc/logstash/conf.d/logstash.conf
sudo systemctl enable logstash && sudo systemctl start logstash

echo "ELK stack setup completed!"
EOF
}

resource "aws_instance" "app_node" {
    ami           = "ami-0cb91c7de36eed2cb" # Ubuntu AMI
    instance_type = "t2.micro"
    key_name      = "elk"
    security_groups = [aws_security_group.elk_sg.name]
    tags = {
    Name = "app-node"
    }
    user_data = <<-EOF
#!/bin/bash
sudo apt update -y

# Install Filebeat
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update -y && sudo apt install -y filebeat

# Configure Filebeat
echo '
filebeat.inputs:
    - type: log
    enabled: true
    paths:
        - /var/log/sample_app.log
    - type: log
    enabled: true
    paths:
        - /var/log/syslog
        - /var/log/messages
output.logstash:
    hosts: ["${aws_instance.elk_node.private_ip}:5044"]
' | sudo tee /etc/filebeat/filebeat.yml
sudo systemctl enable filebeat && sudo systemctl start filebeat

# Install Sample Node.js App
sudo apt install -y nodejs npm
mkdir /home/ubuntu/app && cd /home/ubuntu/app
echo 'console.log("Sample App Running"); setInterval(() => console.log("Log Entry"), 5000);' > app.js
node app.js > /var/log/sample_app.log 2>&1 &
EOF
}

output "elk_ip" {
    value = aws_instance.elk_node.public_ip
}

output "app_ip" {
    value = aws_instance.app_node.public_ip
}

output "elk_ssh" {
    value = "ssh -i my-key.pem ubuntu@${aws_instance.elk_node.public_ip}"
}

output "app_ssh" {
    value = "ssh -i my-key.pem ubuntu@${aws_instance.app_node.public_ip}"
}
````

---

## Step 2: Deploy the Infrastructure

Run these commands in your Terraform directory:

```sh
terraform init
terraform plan
terraform validate
terraform apply -auto-approve
```

---

## Step 3: Get Instance IPs and Connect

After apply, get public IPs and SSH commands:

```sh
terraform output
```

SSH into nodes:

```sh
ssh -i my-key.pem ubuntu@<ELK_NODE_PUBLIC_IP>
ssh -i my-key.pem ubuntu@<APP_NODE_PUBLIC_IP>
```

---

## Step 4: Verify the Setup

- Access Kibana UI: Open `http://<ELK_NODE_PUBLIC_IP>:5601` in your browser
- Wait 1-2 minutes for services to start
- In Kibana, go to **Discover** and select `logs-*` index to view logs

---

## Step 5: Troubleshooting

- Check service status:
  ```sh
  sudo systemctl status elasticsearch
  sudo systemctl status logstash
  sudo systemctl status kibana
  sudo systemctl status filebeat
  ```
- If Java is missing:
  ```sh
  sudo apt update && sudo apt install -y openjdk-11-jdk
  ```
- Restart services if needed:
  ```sh
  sudo systemctl restart elasticsearch
  sudo systemctl restart kibana
  sudo systemctl restart logstash
  sudo systemctl restart filebeat
  ```
- Check logs for errors:
  ```sh
  sudo journalctl -u elasticsearch --no-pager | tail -50
  sudo journalctl -u logstash --no-pager | tail -50
  sudo journalctl -u kibana --no-pager | tail -50
  sudo journalctl -u filebeat --no-pager | tail -50
  ```
- Check Elasticsearch health:
  ```sh
  curl -X GET "http://localhost:9200/_cluster/health?pretty"
  ```
- Edit config files if needed:
  - Elasticsearch: `/etc/elasticsearch/elasticsearch.yml`
  - Kibana: `/etc/kibana/kibana.yml`
  - Logstash: `/etc/logstash/conf.d/logstash.conf`
  - Filebeat: `/etc/filebeat/filebeat.yml`

---

## Summary

- ELK Stack is fully configured and logs are indexed in Elasticsearch
- View logs in Kibana UI
- Correlate application logs with transaction traces
- Fully automated ELK Stack deployment using Terraform
  EOF
  }

resource "aws_instance" "app_node" {
ami = "ami-0cb91c7de36eed2cb" # Ubuntu AMI
instance_type = "t2.micro"
key_name = "elk"
security_groups = [aws_security_group.elk_sg.name]
tags = {
Name = "app-node"
}
user_data = <<-EOF
#!/bin/bash
sudo apt update -y

# Install Filebeat

wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo apt-key add -
echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-7.x.list
sudo apt update -y && sudo apt install -y filebeat

# Configure Filebeat

echo '
filebeat.inputs: - type: log
enabled: true
paths: - /var/log/sample_app.log - type: log
enabled: true
paths: - /var/log/syslog - /var/log/messages
output.logstash:
hosts: ["${aws_instance.elk_node.private_ip}:5044"]
' | sudo tee /etc/filebeat/filebeat.yml
sudo systemctl enable filebeat && sudo systemctl start filebeat

# Install Sample Node.js App

sudo apt install -y nodejs npm
mkdir /home/ubuntu/app && cd /home/ubuntu/app
echo 'console.log("Sample App Running"); setInterval(() => console.log("Log Entry"), 5000);' > app.js
node app.js > /var/log/sample_app.log 2>&1 &
EOF
}

output "elk_ip" {
value = aws_instance.elk_node.public_ip
}

output "app_ip" {
value = aws_instance.app_node.public_ip
}

output "elk_ssh" {
value = "ssh -i my-key.pem ubuntu@${aws_instance.elk_node.public_ip}"
}

output "app_ssh" {
value = "ssh -i my-key.pem ubuntu@${aws_instance.app_node.public_ip}"
}

````

---

## 3. Deploy the Infrastructure

Run these commands in your Terraform directory:

```sh
terraform init
terraform plan
terraform validate
terraform apply -auto-approve
````

---

## 4. Get Instance IPs and Connect

After apply, get public IPs and SSH commands:

```sh
terraform output
```

SSH into nodes:

```sh
ssh -i my-key.pem ubuntu@<ELK_NODE_PUBLIC_IP>
ssh -i my-key.pem ubuntu@<APP_NODE_PUBLIC_IP>
```

---

## 5. Verify the Setup

- Access Kibana UI: Open `http://<ELK_NODE_PUBLIC_IP>:5601` in your browser
- Wait 1-2 minutes for services to start
- In Kibana, go to **Discover** and select `logs-*` index to view logs

---

## 6. Troubleshooting

- Check service status:
  ```sh
  sudo systemctl status elasticsearch
  sudo systemctl status logstash
  sudo systemctl status kibana
  sudo systemctl status filebeat
  ```
- If Java is missing:
  ```sh
  sudo apt update && sudo apt install -y openjdk-11-jdk
  ```
- Restart services if needed:
  ```sh
  sudo systemctl restart elasticsearch
  sudo systemctl restart kibana
  sudo systemctl restart logstash
  sudo systemctl restart filebeat
  ```
- Check logs for errors:
  ```sh
  sudo journalctl -u elasticsearch --no-pager | tail -50
  sudo journalctl -u logstash --no-pager | tail -50
  sudo journalctl -u kibana --no-pager | tail -50
  sudo journalctl -u filebeat --no-pager | tail -50
  ```
- Check Elasticsearch health:
  ```sh
  curl -X GET "http://localhost:9200/_cluster/health?pretty"
  ```
- Edit config files if needed:
  - Elasticsearch: `/etc/elasticsearch/elasticsearch.yml`
  - Kibana: `/etc/kibana/kibana.yml`
  - Logstash: `/etc/logstash/conf.d/logstash.conf`
  - Filebeat: `/etc/filebeat/filebeat.yml`

---

## Summary

- ELK Stack is fully configured and logs are indexed in Elasticsearch
- View logs in Kibana UI
- Correlate application logs with transaction traces
- Fully automated ELK Stack deployment using Terraform
