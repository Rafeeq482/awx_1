# Ansible Webstack Deployment

A production-ready Ansible automation project for deploying and managing web infrastructure components including Nginx, Apache, and load balancers.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Environment Setup](#environment-setup)
- [Configuration](#configuration)
- [Deployment Guide](#deployment-guide)
- [Production Deployment](#production-deployment)
- [Troubleshooting](#troubleshooting)
- [Security Best Practices](#security-best-practices)
- [Contributing](#contributing)

## Overview

This Ansible project automates the deployment of web servers (Nginx and Apache) across multiple environments. It supports:

- Individual Nginx server deployment
- Individual Apache server deployment
- Combined web server deployment on single hosts
- Load balancer configuration
- Production-grade configurations

## Prerequisites

### Local Environment Requirements

- **Operating System**: Ubuntu 20.04+ (tested), RHEL/CentOS 8+, or macOS
- **Python**: 3.8 or higher
- **Ansible**: 2.10 or higher
- **SSH Client**: OpenSSH or compatible

### Target Server Requirements

- **Operating System**: Ubuntu 18.04+, RHEL/CentOS 7+
- **Python**: 3.6 or higher
- **SSH Access**: Key-based authentication required
- **Sudo Access**: Required for package installation and service management

## Project Structure

```
ansible-webstack/
├── README.md                 # This file
├── site.yml                  # Main playbook
├── inventory.ini             # Production inventory
├── ansible.cfg              # Ansible configuration (create if needed)
├── group_vars/              # Group-specific variables
├── host_vars/               # Host-specific variables
├── roles/                   # Ansible roles
│   ├── nginx/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   ├── templates/
│   │   └── vars/main.yml
│   ├── apache/
│   │   ├── tasks/main.yml
│   │   ├── handlers/main.yml
│   │   ├── templates/
│   │   └── vars/main.yml
│   └── loadbalancer/
│       ├── tasks/main.yml
│       ├── handlers/main.yml
│       ├── templates/
│       └── vars/main.yml
└── environments/            # Environment-specific configs
    ├── staging/
    └── production/
```

## Environment Setup

### 1. Install Ansible

#### Ubuntu/Debian:
```bash
sudo apt update
sudo apt install software-properties-common
sudo add-apt-repository --yes --update ppa:ansible/ansible
sudo apt install ansible
```

#### macOS:
```bash
brew install ansible
```

#### Python pip:
```bash
pip3 install ansible
```

### 2. Verify Installation

```bash
ansible --version
ansible-playbook --version
```

### 3. Clone and Setup Project

```bash
# Navigate to your project directory
cd /home/divyansh_pathak/ansible-webstack

# Verify project structure
ls -la
```

## Configuration

### 1. Create Ansible Configuration (Optional but Recommended)

Create `ansible.cfg` in the project root:

```ini
[defaults]
inventory = inventory.ini
host_key_checking = False
retry_files_enabled = False
stdout_callback = yaml
gathering = smart
fact_caching = memory
timeout = 30

[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=60s -o StrictHostKeyChecking=no
pipelining = True
```

### 2. Configure SSH Access

Ensure your SSH private key has correct permissions:

```bash
chmod 600 ~/aws.pem
```

Test SSH connectivity to your servers:

```bash
ssh -i ~/aws.pem ubuntu@13.234.101.11
```

### 3. Inventory Configuration

Your current inventory structure:

- **nginx group**: Dedicated Nginx servers
- **apache group**: Dedicated Apache servers  
- **both group**: Servers running both web servers

## Deployment Guide

### Pre-deployment Checks

1. **Verify connectivity to all hosts**:
```bash
ansible all -i inventory.ini -m ping
```

2. **Check sudo access**:
```bash
ansible all -i inventory.ini -m shell -a "sudo whoami" --become
```

3. **Gather system facts**:
```bash
ansible all -i inventory.ini -m setup --tree /tmp/facts
```

### Basic Deployment Commands

#### Deploy to Specific Groups

**Deploy Nginx only**:
```bash
ansible-playbook -i inventory.ini site.yml --limit nginx
```

**Deploy Apache only**:
```bash
ansible-playbook -i inventory.ini site.yml --limit apache
```

**Deploy both web servers**:
```bash
ansible-playbook -i inventory.ini site.yml --limit both
```

#### Deploy to All Servers

```bash
ansible-playbook -i inventory.ini site.yml
```

#### Dry Run (Check Mode)

```bash
ansible-playbook -i inventory.ini site.yml --check --diff
```

### Advanced Deployment Options

#### Verbose Output
```bash
ansible-playbook -i inventory.ini site.yml -v   # verbose
ansible-playbook -i inventory.ini site.yml -vv  # more verbose
ansible-playbook -i inventory.ini site.yml -vvv # debug level
```

#### Step-by-step Execution
```bash
ansible-playbook -i inventory.ini site.yml --step
```

#### Start from Specific Task
```bash
ansible-playbook -i inventory.ini site.yml --start-at-task="Install Nginx"
```

#### Tag-based Execution (if tags are defined)
```bash
ansible-playbook -i inventory.ini site.yml --tags "nginx,config"
ansible-playbook -i inventory.ini site.yml --skip-tags "restart"
```

## Production Deployment

### 1. Pre-Production Checklist

- [ ] All roles tested in staging environment
- [ ] Backup existing configurations
- [ ] Verify rollback procedures
- [ ] Confirm maintenance window
- [ ] Notify stakeholders

### 2. Production Deployment Process

#### Step 1: Backup Current State
```bash
# Create backup playbook or manual backup
ansible all -i inventory.ini -m shell -a "sudo tar -czf /tmp/webserver-backup-$(date +%Y%m%d).tar.gz /etc/nginx /etc/apache2 /var/www" --become
```

#### Step 2: Deploy with Rolling Updates (if applicable)
```bash
# Deploy to one server at a time
ansible-playbook -i inventory.ini site.yml --limit nginx[0]
ansible-playbook -i inventory.ini site.yml --limit nginx[1]
```

#### Step 3: Verify Deployment
```bash
# Check service status
ansible all -i inventory.ini -m service -a "name=nginx state=started" --become
ansible all -i inventory.ini -m service -a "name=apache2 state=started" --become

# Verify web server response
ansible all -i inventory.ini -m uri -a "url=http://localhost/ method=GET"
```

### 3. Post-Deployment Validation

```bash
# Check logs for errors
ansible all -i inventory.ini -m shell -a "sudo tail -n 50 /var/log/nginx/error.log"
ansible all -i inventory.ini -m shell -a "sudo tail -n 50 /var/log/apache2/error.log"

# Verify port listening
ansible all -i inventory.ini -m shell -a "sudo netstat -tlnp | grep -E ':(80|443)'"
```

## Troubleshooting

### Common Issues and Solutions

#### 1. SSH Connection Issues
```bash
# Test SSH connectivity
ssh -i ~/aws.pem ubuntu@<target-ip> -v

# Check SSH agent
ssh-add ~/aws.pem
```

#### 2. Permission Denied Errors
```bash
# Verify sudo access
ansible all -i inventory.ini -m shell -a "groups" --become
```

#### 3. Service Start Failures
```bash
# Check service status
ansible all -i inventory.ini -m service -a "name=nginx" --become
ansible all -i inventory.ini -m systemd -a "name=nginx state=started" --become
```

#### 4. Playbook Syntax Errors
```bash
# Validate playbook syntax
ansible-playbook site.yml --syntax-check

# Validate inventory
ansible-inventory -i inventory.ini --list
```

### Debug Commands

```bash
# Check Ansible configuration
ansible-config dump

# List all tasks that would be executed
ansible-playbook -i inventory.ini site.yml --list-tasks

# List all hosts
ansible-playbook -i inventory.ini site.yml --list-hosts
```

## Security Best Practices

### 1. SSH Key Management
- Use dedicated SSH keys for production
- Implement key rotation policies
- Store keys securely (avoid committing to version control)

### 2. Inventory Security
- Use Ansible Vault for sensitive data:
```bash
ansible-vault create group_vars/all/vault.yml
ansible-vault edit group_vars/all/vault.yml
```

### 3. Privilege Management
- Use `become` only when necessary
- Implement least privilege principle
- Regular security audits

### 4. Network Security
- Configure firewall rules
- Use VPN for production access
- Implement jump hosts/bastion servers

## Environment Variables

Create a `.env` file (don't commit to git):

```bash
export ANSIBLE_HOST_KEY_CHECKING=False
export ANSIBLE_STDOUT_CALLBACK=yaml
export ANSIBLE_PRIVATE_KEY_FILE=~/aws.pem
```

Source it before running playbooks:
```bash
source .env
```

## Monitoring and Logging

### Enable Ansible Logging
Add to `ansible.cfg`:

```ini
[defaults]
log_path = ./ansible.log
```

### Monitoring Deployments
```bash
# Monitor deployment progress
tail -f ansible.log

# Check system resources during deployment
ansible all -i inventory.ini -m shell -a "top -bn1 | head -n 20"
```

## Contributing

1. Test all changes in staging environment first
2. Follow Ansible best practices and conventions
3. Document any new variables or requirements
4. Update this README for any new procedures

## Useful Commands Reference

```bash
# Quick health check
ansible all -i inventory.ini -m ping

# Gather facts
ansible all -i inventory.ini -m setup

# Check disk space
ansible all -i inventory.ini -m shell -a "df -h"

# Check memory usage
ansible all -i inventory.ini -m shell -a "free -h"

# Restart services
ansible all -i inventory.ini -m service -a "name=nginx state=restarted" --become

# Execute arbitrary commands
ansible all -i inventory.ini -m shell -a "uptime"
```

---

**Note**: Always test deployments in a staging environment before applying to production. Keep backups of critical configurations and have rollback procedures ready.

For questions or issues, please refer to the [Ansible Documentation](https://docs.ansible.com/) or contact the infrastructure team.
