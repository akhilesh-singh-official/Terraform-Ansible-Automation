# Terraform & Ansible Infrastructure Automation

An Infrastructure as Code project that automates AWS infrastructure provisioning and server configuration using **Terraform, Ansible, Python, and AWS EC2**.

This project demonstrates how to provision multiple EC2 instances across Development, Staging, and Production environments using Terraform, automatically generate Ansible inventory from Terraform outputs, and configure web servers using Ansible.

The objective is to create a repeatable infrastructure deployment workflow that minimizes manual server provisioning and configuration.

## Architecture

The project follows a two-stage automation workflow:

1. Terraform provisions the required AWS infrastructure.
2. Python generates a dynamic Ansible inventory using Terraform outputs.
3. Ansible connects to the provisioned EC2 instances.
4. Ansible installs and configures Nginx.
5. Environment-specific HTML pages are deployed to the corresponding servers.

### Infrastructure Architecture

```text
                  Developer
                      |
                      v
                deploy.sh
                      |
                      v
                  Terraform
                      |
                      v
               AWS Default VPC
                      |
              Shared Security Group
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
        DEV         STAGE        PROD
       2 EC2        2 EC2        2 EC2
          |           |           |
          +-----------+-----------+
                      |
                      v
              Terraform Outputs
                      |
                      v
             Python Inventory
                Generator
                      |
                      v
              Ansible Inventory
                      |
                      v
               Ansible Playbook
                      |
                      v
               Webserver Role
                      |
                      v
                 Nginx Setup
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
       DEV Page   STAGE Page   PROD Page
```

## Technology Stack

| Technology | Purpose |
|---|---|
| Terraform | AWS infrastructure provisioning |
| Ansible | Configuration management |
| AWS EC2 | Virtual machine provisioning |
| AWS VPC | Networking |
| AWS Security Groups | Network access control |
| Nginx | Web server |
| Python | Dynamic inventory generation |
| AWS CLI | AWS resource management |
| Linux | Server operating system |
| SSH | Remote server connectivity |
| Git & GitHub | Version control |

## Infrastructure Overview

Terraform provisions six EC2 instances across three logical environments.

| Environment | Instances | Purpose |
|---|---:|---|
| Development | 2 | Development website deployment |
| Staging | 2 | Pre-production website deployment |
| Production | 2 | Production demonstration website |
| **Total** | **6** | **Multi-environment infrastructure** |

### AWS Resources

The current Terraform configuration provisions:

- Six EC2 instances.
- A shared security group.
- An AWS EC2 key pair.
- Ubuntu 22.04 AMI selection.
- EC2 instances using the `t2.micro` instance type by default.
- Environment-specific resource tags.
- HTTP and SSH security group rules.

The instances are provisioned in the AWS default VPC, with the AWS region configurable through Terraform variables.

Development, Staging, and Production are represented as logical environments using instance tags and Ansible inventory groups. They are not separate AWS accounts or isolated VPCs.

## Project Structure

```text
Terraform-Ansible-Infra/
│
├── terra-config/
│   ├── main.tf
│   ├── provider.tf
│   ├── variables.tf
│   ├── output.tf
│   └── output.json
│
├── ansible/
│   ├── ansible.cfg
│   ├── inventory/
│   │   └── hosts.ini
│   ├── playbook.yml
│   └── roles/
│       └── webserver/
│           ├── tasks/
│           │   └── main.yml
│           ├── files/
│           │   ├── index-dev.html
│           │   ├── index-stage.html
│           │   └── index-prod.html
│           └── handlers/
│               └── main.yml
│
├── scripts/
│   └── generate_inv.py
│
├── deploy.sh
│
└── README.md
```

*The handlers directory is a recommended improvement. Add it when implementing the Nginx handler. Exclude generated output files and Terraform state from version control.*

## Prerequisites

- AWS account
- AWS CLI
- Terraform
- Ansible
- Python 3
- SSH client
- Git

## AWS Authentication

Configure AWS CLI credentials using an appropriate authentication method.

```bash
aws configure
```

Verify AWS authentication:

```bash
aws sts get-caller-identity
```

## SSH Key Configuration

Generate an SSH key pair for EC2 access:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/appKey
```

The public key is registered with AWS through Terraform, while Ansible uses the corresponding private key to connect to EC2 instances.

Ensure the private key is protected and never commit it to GitHub.

## Infrastructure Provisioning Using Terraform

### 1. Clone the repository

```bash
git clone https://github.com/akhilesh-singh-official/Terraform-Ansible-Infra.git

cd Terraform-Ansible-Infra
```

### 2. Navigate to Terraform configuration

```bash
cd terra-config
```

### 3. Initialize Terraform

```bash
terraform init
```

### 4. Validate configuration

```bash
terraform validate
```

### 5. Review infrastructure changes

```bash
terraform plan
```

### 6. Provision AWS infrastructure

```bash
terraform apply
```

Terraform creates the EC2 instances and supporting resources defined in the configuration.

### 7. Export Terraform outputs

```bash
terraform output -json instance_public_ips > output.json
```

The generated JSON contains the public IP addresses grouped by environment.

## Dynamic Ansible Inventory Generation

A Python script reads the Terraform output JSON and generates an Ansible inventory automatically.

### Generate inventory

From the project root:

```bash
python3 scripts/generate_inv.py
```

The script:

1. Reads Terraform output JSON.
2. Extracts EC2 public IP addresses.
3. Creates Development, Staging and Production inventory groups.
4. Configures the remote SSH user and private key.
5. Assigns environment variables to inventory groups.
6. Writes the generated inventory to `ansible/inventory/hosts.ini`.

This removes the need to manually update EC2 public IP addresses whenever infrastructure is recreated.

## Server Configuration Using Ansible

After generating the inventory, Ansible configures all provisioned instances.

### 1. Navigate to Ansible directory

```bash
cd ansible
```

### 2. Verify connectivity

```bash
ansible all -m ping
```

### 3. Execute the playbook

```bash
ansible-playbook playbook.yml
```

### Ansible Configuration Workflow

The Ansible playbook executes the `webserver` role on all inventory hosts.

The role performs the following tasks:

- Installs Nginx using the Ubuntu package manager.
- Copies the appropriate HTML file based on the environment variable.
- Places the website content in `/var/www/html/index.html`.
- Restarts Nginx to apply the configuration.

### Environment-Based Deployment

| Environment | HTML File |
|---|---|
| Development | `index-dev.html` |
| Staging | `index-stage.html` |
| Production | `index-prod.html` |

This allows the same Ansible playbook to configure different environments without maintaining separate playbooks.

## One-Command Deployment

The project includes a shell script that automates the complete infrastructure provisioning and configuration workflow.

From the project root:

```bash
bash deploy.sh
```

The script performs the following operations:

1. Initializes Terraform.
2. Provisions AWS infrastructure.
3. Exports EC2 public IP addresses.
4. Generates Ansible inventory using Python.
5. Waits for the EC2 instances to initialize.
6. Executes the Ansible playbook.
7. Completes the web server configuration.

This connects Terraform and Ansible into a single deployment workflow.

*The current script uses a fixed 15-second delay. A readiness check with SSH retries is a planned improvement.*

## Infrastructure Cleanup

To avoid unnecessary AWS charges, destroy the provisioned infrastructure when it is no longer required.

Navigate to the Terraform directory:

```bash
cd terra-config
```

Review the destruction plan:

```bash
terraform plan -destroy
```

Destroy the infrastructure:

```bash
terraform destroy
```

Confirm the operation after reviewing the resources.

## Security Considerations

- Restrict SSH access to trusted IP addresses.
- Avoid exposing SSH access to the entire internet.
- Protect private SSH keys.
- Use appropriate AWS IAM permissions.
- Avoid committing Terraform state and generated sensitive files.
- Restrict unnecessary inbound traffic.
- Use secure authentication for AWS and remote server access.

## Future Improvements

- Implement reusable Terraform modules.
- Introduce remote Terraform state using Amazon S3.
- Implement state locking.
- Make EC2 instance counts configurable.
- Improve dynamic inventory error handling.
- Replace fixed deployment delays with SSH readiness checks.
- Organize Ansible tasks into reusable roles.
- Implement Ansible handlers for service restarts.
- Enable SSH host key verification.
- Introduce automated Terraform validation using GitHub Actions.
- Add infrastructure monitoring using Prometheus and Grafana.
- Implement separate network environments using dedicated VPCs or other isolation mechanisms.

## What I Learned

Through this project, I gained practical experience with:

- Infrastructure as Code using Terraform.
- Provisioning multiple EC2 instances through Terraform `for_each`.
- Managing infrastructure using variables, locals and outputs.
- AWS EC2 and security group configuration.
- Automating Ansible inventory generation using Python.
- Managing multiple environments through Ansible inventory groups.
- Writing Ansible playbooks and reusable roles.
- Automating Nginx installation and configuration.
- Connecting Terraform provisioning and Ansible configuration.
- Building an end-to-end infrastructure deployment workflow using Bash.

## Author

**Akhilesh Singh**

DevOps | AWS | Terraform | Ansible | Infrastructure Automation

GitHub: [akhilesh-singh-official](https://github.com/akhilesh-singh-official)

---

*This project is part of my hands-on DevOps portfolio, focused on automating cloud infrastructure provisioning and configuration management using Terraform and Ansible.*
