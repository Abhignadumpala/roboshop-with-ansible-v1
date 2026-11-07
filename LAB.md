# RoboShop with Ansible v1: Lab Step by Step

[← Back to README](README.md)

> **In short:** from your laptop we create an IAM role, one security group rule and the **Ansible server**. On the Ansible server we install Ansible, get the code, create all 10 RoboShop servers with `roboshop.yaml`, set up each one with its playbook (`01-mongodb.yaml` … `10-frontend.yaml`), open the shop in the browser, and delete everything at the end.

## Table of Contents

**Part 1: AWS setup (on your laptop)**
- [Step 1: Create the IAM role](#step-1-create-the-iam-role)
- [Step 2: Attach the EC2 policy to the role](#step-2-attach-the-ec2-policy-to-the-role)
- [Step 3: Attach the Route 53 policy to the role](#step-3-attach-the-route-53-policy-to-the-role)
- [Step 4: Create an instance profile](#step-4-create-an-instance-profile)
- [Step 5: Add the role to the instance profile](#step-5-add-the-role-to-the-instance-profile)
- [Step 6: Add an SSH rule to the roboshop-common security group](#step-6-add-an-ssh-rule-to-the-roboshop-common-security-group)
- [Step 7: Create the Ansible server (EC2)](#step-7-create-the-ansible-server-ec2)
- [Step 8: Get the Ansible server's public IP](#step-8-get-the-ansible-servers-public-ip)

**Part 2: Prepare the Ansible server**
- [Step 9: Log in to the Ansible server](#step-9-log-in-to-the-ansible-server)
- [Step 10: Install Ansible and the collections](#step-10-install-ansible-and-the-collections)
- [Step 11: Install boto3 and check AWS access](#step-11-install-boto3-and-check-aws-access)
- [Step 12: Get the code](#step-12-get-the-code)
- [Step 13: Set your domain](#step-13-set-your-domain)
- [Step 14: Set the SSH password](#step-14-set-the-ssh-password)
- [Step 15: Create the log folder](#step-15-create-the-log-folder)

**Part 3: Run RoboShop**
- [Step 16: Create the 10 servers and DNS records](#step-16-create-the-10-servers-and-dns-records)
- [Step 17: Check Ansible can reach every server](#step-17-check-ansible-can-reach-every-server)
- [Step 18: Run the playbooks in order](#step-18-run-the-playbooks-in-order)
- [Step 19: Open the shop](#step-19-open-the-shop)

**Part 4: Clean up**
- [Step 20: Delete the RoboShop servers and records (on the Ansible server)](#step-20-delete-the-roboshop-servers-and-records-on-the-ansible-server)
- [Step 21: Delete the Ansible server (on your laptop)](#step-21-delete-the-ansible-server-on-your-laptop)

**Wrap-up**
- [Troubleshooting](#troubleshooting)
- [What I learned](#what-i-learned)

---

## How this lab works

**Definition:** in this lab one **Ansible server** (the control node) creates the 10 RoboShop servers on AWS and then sets each one up with a playbook. We never log in to the RoboShop servers by hand. Ansible does all the work over SSH.

```
 Laptop ──(aws cli)──► IAM role + SG rule + Ansible server
                                  │
                                  ▼
                           Ansible server
                  ├─ roboshop.yaml ──► AWS API: 10 EC2 servers + Route 53 records
                  └─ 01-…10-*.yaml ──► SSH: install + configure each server
```

| Part | Where | What we do |
|---|---|---|
| **Part 1: AWS setup** | Laptop | IAM role, security group rule, Ansible server |
| **Part 2: Ansible server** | Ansible server | Install Ansible + boto3, get the code |
| **Part 3: Run RoboShop** | Ansible server | Create the servers, run the playbooks in order, open the site |
| **Part 4: Clean up** | Ansible server + laptop | Delete everything |

**Before you start:**

| Need | Why |
|---|---|
| AWS CLI on your laptop, logged in as an admin user | Creates the role and the Ansible server |
| Security groups `roboshop-common` + `roboshop-<name>` for every server | Each server gets both; the playbook doesn't create them |
| Route 53 hosted zone (`abhignadevops.store`) | Where the DNS records go |
| AMI `Redhat-9-DevOps-Practice` (`ami-0055977a6e828a4f4`) | RHEL 9 with SSH password login |

---

## Part 1: AWS setup (on your laptop)

**Order:** first the IAM role (Steps 1–5), then the security group rule (Step 6), then the Ansible server (Step 7). The server gets the role and the rule the moment it is created.

```
 Step 1 role ─► Steps 2–3 policies ─► Steps 4–5 instance profile ─► Step 6 SSH rule ─► Step 7 Ansible server
```

### Step 1: Create the IAM role

The Ansible server needs permission to create EC2 servers and DNS records. An **IAM role** gives it that permission, with temporary keys and no access keys on disk.

```bash
aws iam create-role \
  --role-name ansible-server-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
```

**What it does:** creates an empty role called `ansible-server-role`. The JSON says **only EC2 servers** can use it.

**Expected:** JSON output with `"RoleName": "ansible-server-role"` ✅.

### Step 2: Attach the EC2 policy to the role

```bash
aws iam attach-role-policy \
  --role-name ansible-server-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
```

**What it does:** gives the role **`AmazonEC2FullAccess`**, so the Ansible server can **create and delete EC2 servers**.

**Expected:** no output ✅.

### Step 3: Attach the Route 53 policy to the role

```bash
aws iam attach-role-policy \
  --role-name ansible-server-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess
```

**What it does:** gives the role **`AmazonRoute53FullAccess`**, so the Ansible server can **create and delete DNS records**.

**Expected:** no output ✅.

### Step 4: Create an instance profile

An EC2 server can't take a role directly. It takes an **instance profile**: a container that holds the role.

```
 IAM role  ──put inside──►  instance profile  ──attach──►  EC2 server
```

```bash
aws iam create-instance-profile --instance-profile-name ansible-server-role
```

**What it does:** creates an empty instance profile called `ansible-server-role`.

**Expected:** JSON output with `"InstanceProfileName": "ansible-server-role"` ✅.

> 💡 In the AWS console this is done for you when you create a role for EC2. With the CLI we do it ourselves.

### Step 5: Add the role to the instance profile

```bash
aws iam add-role-to-instance-profile \
  --instance-profile-name ansible-server-role \
  --role-name ansible-server-role
```

**What it does:** puts our role inside the instance profile. Now it's ready to attach to a server.

**Expected:** no output ✅.

### Step 6: Add an SSH rule to the roboshop-common security group

We are **not creating a new security group** here. We already created `roboshop-common` in the previous RoboShop lab, so we **use the same one** and just **add one inbound rule** to it.

**Why the rule?** Right now `roboshop-common` allows SSH only from our home IP. The Ansible server is inside AWS, so its SSH to the RoboShop servers would time out.

> **Don't have `roboshop-common` yet?** Create it first and allow SSH from your home IP, then add the rule below.
>
> ```bash
> aws ec2 create-security-group \
>   --group-name roboshop-common \
>   --description "SSH for all RoboShop servers"
> ```
>
> ```bash
> aws ec2 authorize-security-group-ingress \
>   --group-name roboshop-common \
>   --protocol tcp \
>   --port 22 \
>   --cidr $(curl -s checkip.amazonaws.com)/32
> ```
>
> **What it does:** creates the security group, then allows SSH (port 22) from **your laptop's public IP only**.

**Add the inbound rule to the existing security group:**

```bash
aws ec2 authorize-security-group-ingress \
  --group-name roboshop-common \
  --protocol tcp \
  --port 22 \
  --source-group roboshop-common
```

**What it does:** adds the rule **SSH, port 22, source `roboshop-common`**: any server in this group can SSH to any other server in it.

**Expected:** `"Return": true` ✅. (`InvalidPermission.Duplicate` means the rule is already there, which is fine.)

### Step 7: Create the Ansible server (EC2)

```bash
aws ec2 run-instances \
  --image-id ami-0055977a6e828a4f4 \
  --instance-type t3.micro \
  --security-groups roboshop-common \
  --iam-instance-profile Name=ansible-server-role \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ansible-server}]' \
  --query 'Instances[0].InstanceId' --output text
```

**What it does:** creates the RHEL 9 Ansible server with the instance profile (Step 5) and the security group (Step 6).

**Expected:** an instance ID like `i-0abc123def4567890` ✅.

| Error | Fix |
|---|---|
| `Invalid IAM Instance Profile name` | A new profile takes a few seconds to be ready. Wait 10 seconds and run it again |

### Step 8: Get the Ansible server's public IP

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ansible-server" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].PublicIpAddress' --output text
```

**What it does:** prints the public IP we use to log in.

**Expected:** an IP like `54.xx.xx.xx` ✅.

---

## Part 2: Prepare the Ansible server

### Step 9: Log in to the Ansible server

```bash
ssh ec2-user@<ansible-server-public-ip>
```

Type `yes` the first time, then the SSH password (nothing shows while you type).

### Step 10: Install Ansible and the collections

```bash
sudo dnf install ansible-core git -y
```

```bash
ansible-galaxy collection install amazon.aws community.general community.rabbitmq
```

**What it does:** `ansible-core` has only the built-in modules. The **collections** add the ones we need: `amazon.aws` (EC2, Route 53), `community.general` (npm) and `community.rabbitmq` (RabbitMQ user).

### Step 11: Install boto3 and check AWS access

```bash
sudo pip3.9 install botocore boto3
```

**What it does:** the `amazon.aws` modules call AWS through **boto3**, so it must be installed for Python 3.9, the Python Ansible uses on RHEL 9.

```bash
aws sts get-caller-identity
```

**Expected:** `Arn` shows `assumed-role/ansible-server-role/…` ✅. The server has AWS access with no keys.

### Step 12: Get the code

```bash
git clone https://github.com/Abhignadumpala/roboshop-with-ansible-v1.git
```

```bash
cd roboshop-with-ansible-v1
```

**What it does:** downloads the playbooks, templates and `group_vars/` to the Ansible server. We run every `ansible-playbook` command from inside this folder, so Ansible finds `ansible.cfg` and the inventory.

### Step 13: Set your domain

The server names in the inventory, `group_vars/all.yaml` and `roboshop.yaml` use the domain `abhignadevops.store`. If your hosted zone is different, change all of them in one go:

```bash
sed -i 's/abhignadevops.store/<your-domain>/g' inventory.ini group_vars/all.yaml roboshop.yaml
```

```bash
grep -rn "domain_name\|_host" roboshop.yaml group_vars/all.yaml
```

**Expected:** every name ends with your domain ✅.

### Step 14: Set the SSH password

```bash
vim group_vars/all.yaml
```

Set `ansible_password:` to the SSH password of the RoboShop servers (`ansible_user` is `ec2-user`).

> ⚠️ Never push the real password to GitHub. Keep `<ssh-password>` in the repo and fill it in only on the server.

### Step 15: Create the log folder

```bash
mkdir -p ~/roboshop-logs
```

**What it does:** `ansible.cfg` has `log_path = /home/ec2-user/roboshop-logs/ansible.log`, so Ansible saves every run's output there. The folder must exist first.

---

## Part 3: Run RoboShop

### Step 16: Create the 10 servers and DNS records

```bash
ansible-playbook roboshop.yaml
```

**Expected:** 10 servers `roboshop-mongodb` … `roboshop-frontend` in the EC2 console, and a Route 53 record for each one ✅.

### Step 17: Check Ansible can reach every server

```bash
ansible all -m ping
```

**Expected:** every server shows `"ping": "pong"` ✅.

| Error | Cause | Fix |
|---|---|---|
| `UNREACHABLE … timed out` | No SSH rule from the Ansible server | Step 6 |
| `UNREACHABLE … Permission denied` | Wrong password | Fix `group_vars/all.yaml` |
| `Could not resolve hostname` | Records not created yet | Run `roboshop.yaml` again |

### Step 18: Run the playbooks in order

```bash
ansible-playbook 01-mongodb.yaml
```

```bash
ansible-playbook 02-redis.yaml
```

```bash
ansible-playbook 03-mysql.yaml
```

```bash
ansible-playbook 04-rabbitmq.yaml
```

```bash
ansible-playbook 05-catalogue.yaml
```

```bash
ansible-playbook 06-user.yaml
```

```bash
ansible-playbook 07-cart.yaml
```

```bash
ansible-playbook 08-shipping.yaml
```

```bash
ansible-playbook 09-payment.yaml
```

```bash
ansible-playbook 10-frontend.yaml
```

| Order | Why |
|---|---|
| Databases (01–04) first | catalogue loads products into MongoDB, shipping loads cities into MySQL |
| Apps (05–09) next | They need the databases running |
| Frontend last | It forwards `/api/...` to all the services |

**Expected:** every play ends with `failed=0` ✅. Re-running a playbook is safe (idempotent).

### Step 19: Open the shop

Open `http://abhignadevops.store` in the browser.

**Expected:** the RoboShop home page with products. Add an item to the cart, then check out ✅.

---

## Part 4: Clean up

### Step 20: Delete the RoboShop servers and records (on the Ansible server)

```bash
ansible-playbook roboshop.yaml -e "action=delete"
```

**Expected:** all 10 servers terminating and their records gone ✅.

### Step 21: Delete the Ansible server (on your laptop)

```bash
exit
```

```bash
ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ansible-server" "Name=instance-state-name,Values=running,stopped" \
  --query 'Reservations[].Instances[].InstanceId' --output text)
```

**What it does:** finds the Ansible server's instance ID and saves it in the variable `ID`.

```bash
aws ec2 terminate-instances --instance-ids $ID
```

**Expected:** `"Name": "shutting-down"` ✅. The IAM role and the SG rule cost nothing, so we keep them for the next lab.

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `Invalid IAM Instance Profile name` | Profile not ready yet | Wait 10 seconds, run Step 7 again |
| `Unable to locate credentials` | No IAM role on the Ansible server | Attach `ansible-server-role` (EC2 → Actions → Security → Modify IAM role) |
| `Failed to import the required Python library (botocore and boto3)` | boto3 installed for a different Python | `sudo pip3.9 install botocore boto3` |
| `couldn't resolve module/action 'amazon.aws.ec2_instance'` | Collection not installed | Step 10 |
| `Invalid callback for stdout specified: yaml` | New `community.general` versions removed the `yaml` output | Delete the `stdout_callback=yaml` line in `ansible.cfg` |
| `InvalidGroup.NotFound` | A `roboshop-<name>` security group is missing | Create it in the same region (`us-east-1`) |
| `UNREACHABLE … timed out` | No SSH rule from the Ansible server | Step 6 |
| `UNREACHABLE … Permission denied` | Wrong SSH password | Step 14 |
| Website shows `502 Bad Gateway` | A backend service is down | `ansible <name> -b -m command -a "systemctl status <name>"` |

---

## What I learned

| Part | What I learned |
|---|---|
| **AWS setup** | An **IAM role** (via an instance profile) gives the Ansible server AWS access with no keys on disk; a **self-referencing SG rule** lets servers in the same group SSH to each other |
| **Ansible server** | `ansible-core` needs **collections** for AWS and community modules, and **boto3** for the same Python Ansible uses |
| **Run RoboShop** | One playbook creates all servers + DNS records; playbooks run in **dependency order**: data → backend → frontend |
| **Clean up** | `-e "action=delete"` reuses the same playbook to delete everything |

```
 IAM role + SG rule ──► Ansible server ──► roboshop.yaml ──► 01…04 data ──► 05…09 backend ──► 10 frontend ──► delete
```

**Full explanation of every file:** [RoboShop v1 with Ansible: every file explained](https://github.com/Abhignadumpala/devops-notes/blob/main/day-22-ansible-shell-vs-ansible-aws-ec2-route53/roboshop-v1/README.md)
