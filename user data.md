---
# EC2 Instance Setup & Local Installation Guide
---

## 1. EC2 User Data Script

Use this script in the "User data" field when launching an EC2 instance (Ubuntu or Amazon Linux). It installs Python, pip, AWS CLI, and Terraform:

```sh
#!/bin/bash
# Update the package index and upgrade packages
sudo apt update && sudo apt upgrade -y
sudo apt install pipx
sudo apt install unzip

# Install required packages
sudo apt install -y python3 python3-pip curl wget stress 

# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version

# Install Terraform
wget https://releases.hashicorp.com/terraform/1.7.0/terraform_1.7.0_linux_amd64.zip
unzip terraform_1.7.0_linux_amd64.zip
sudo mv terraform /usr/local/bin/
terraform -v

pipx install locust
sudo apt install zip
sudo apt install python3-locust

# Verify installations
python3 --version
pip3 --version
```

---

## 2. Local Installation Instructions (macOS)

### Install AWS CLI

```sh
brew install awscli
aws --version
```

### Install Terraform

```sh
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
terraform -v
```

### Install Python & pip

```sh
brew install python
python3 --version
pip3 --version
```
