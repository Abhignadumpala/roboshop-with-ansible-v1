# RoboShop with Ansible v1: Lab Step by Step

[← Back to README](README.md)

> **In short:** from your laptop we create an IAM role, one security group rule and the **Ansible server**. On the Ansible server we install Ansible, get the code, create all 10 RoboShop servers with `roboshop.yaml`, set up each one with its playbook (`01-mongodb.yaml` … `10-frontend.yaml`), open the shop in the browser, and delete everything at the end.

## Table of Contents

**Part 1: AWS setup (on your laptop)**
- [Step 1: Create an IAM role for the Ansible server](#step-1-create-an-iam-role-for-the-ansible-server)
- [Step 2: Add an SSH rule to the roboshop-common security group](#step-2-add-an-ssh-rule-to-the-roboshop-common-security-group)
- [Step 3: Create the Ansible server (EC2)](#step-3-create-the-ansible-server-ec2)

**Part 2: Prepare the Ansible server**
- [Step 4: Log in to the Ansible server](#step-4-log-in-to-the-ansible-server)
- [Step 5: Install Ansible and the collections](#step-5-install-ansible-and-the-collections)
- [Step 6: Install boto3 and check AWS access](#step-6-install-boto3-and-check-aws-access)
- [Step 7: Get the code](#step-7-get-the-code)
- [Step 8: Set your domain](#step-8-set-your-domain)
- [Step 9: Set the SSH password](#step-9-set-the-ssh-password)
- [Step 10: Create the log folder](#step-10-create-the-log-folder)

**Part 3: Run RoboShop**
- [Step 11: Create the 10 servers and DNS records](#step-11-create-the-10-servers-and-dns-records)
- [Step 12: Check Ansible can reach every server](#step-12-check-ansible-can-reach-every-server)
- [Step 13: Run the playbooks in order](#step-13-run-the-playbooks-in-order)
- [Step 14: Open the shop](#step-14-open-the-shop)

**Part 4: Clean up**
- [Step 15: Delete the RoboShop servers and records](#step-15-delete-the-roboshop-servers-and-records-on-the-ansible-server)
- [Step 16: Delete the Ansible server](#step-16-delete-the-ansible-server-on-your-laptop)

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

### Step 1: Create an IAM role for the Ansible server

We **attach an IAM role to the Ansible server so that it can create the RoboShop servers**. `roboshop.yaml` runs on the Ansible server and asks AWS to create 10 EC2 servers and their Route 53 records. AWS only does that if the caller has permission, and the IAM role is that permission: EC2 + Route 53, nothing else.

The role also gives the server **temporary keys automatically** (they rotate every few hours), so there are no access keys on disk.

```
 Ansible server ──(IAM role: EC2 + Route 53 permission)──► AWS API ──► 10 EC2 servers + DNS records
```

**Why create the role before the server?** An IAM role is not inside any server. It's an **identity in your AWS account**, like a user, that exists on its own. We create it once, then **attach** it when we launch the server (Step 3, `--iam-instance-profile`), so the server has AWS access from the first second.

```
 Step 1: IAM role (in the AWS account) ──attach at launch──► Step 3: Ansible server (EC2)
```

> 💡 You can also do it the other way: launch the server first, then attach the role later in **EC2 → select the server → Actions → Security → Modify IAM role**. Both work; creating the role first saves that extra step.

```bash
aws iam create-role \
  --role-name ansible-server-role \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
```

**What it does:** creates an **empty role** (no permissions yet). The JSON is the **trust policy**: it says *who* may use the role.

| JSON part | Means |
|---|---|
| `"Principal":{"Service":"ec2.amazonaws.com"}` | Only **EC2 servers** can use this role (not users, not Lambda) |
| `"Action":"sts:AssumeRole"` | They may "put on" the role and get its temporary keys |

Now we give the role **2 permissions**:

1. **`AmazonEC2FullAccess`**: so the Ansible server can **create and delete EC2 servers**
2. **`AmazonRoute53FullAccess`**: so the Ansible server can **create and delete DNS records**

```bash
aws iam attach-role-policy \
  --role-name ansible-server-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
```

```bash
aws iam attach-role-policy \
  --role-name ansible-server-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess
```

**What it does:** adds the **permissions**: *what* the role may do. Both are AWS-managed policies (AWS writes and updates them).

| Policy | Allows | What `roboshop.yaml` uses it for |
|---|---|---|
| `AmazonEC2FullAccess` | Everything in EC2: create, start, stop and terminate servers, read security groups, tags, volumes (plus load balancers and auto scaling) | **Create** the 10 servers (`ec2_instance`), find them by Name tag, attach the 2 security groups, **terminate** them on delete |
| `AmazonRoute53FullAccess` | Everything in Route 53: read hosted zones, create, change and delete DNS records | **Create** `mongodb.abhignadevops.store` … records and **delete** them on cleanup |

**What the role can NOT do:** anything outside EC2 and Route 53: no S3 buckets, no IAM users or roles, no RDS, no billing. If the server tries, AWS answers `AccessDenied`.

| Option | Access | Use when |
|---|---|---|
| `AdministratorAccess` | Everything in the account | ❌ Never on a server: if it's hacked, the whole account is open |
| **EC2 + Route 53 full access** (ours) | Only the 2 services we need | ✅ Labs and learning |
| Custom policy (only `ec2:RunInstances`, `route53:ChangeResourceRecordSets` …) | Only the exact actions | ✅ Real projects: **least privilege** |

> **Least privilege** = give only the permissions the job needs, nothing more.

```bash
aws iam create-instance-profile --instance-profile-name ansible-server-role
```

```bash
aws iam add-role-to-instance-profile \
  --instance-profile-name ansible-server-role \
  --role-name ansible-server-role
```

**What it does:** an EC2 server can't use a role directly; it uses an **instance profile**, a box that holds the role.

### Step 2: Add an SSH rule to the roboshop-common security group

We **don't create a new security group** here. `roboshop-common` already exists, and every RoboShop server and the Ansible server use it. Right now it allows SSH (port 22) **only from our home IP**. The Ansible server is **inside AWS** with a different IP, so its SSH to the RoboShop servers would time out. We add **one inbound rule** to fix that.

| Inbound rule | Value |
|---|---|
| Type | SSH |
| Port | 22 |
| Source | `roboshop-common` itself (any server in this group) |

```bash
aws ec2 authorize-security-group-ingress \
  --group-name roboshop-common \
  --protocol tcp \
  --port 22 \
  --source-group roboshop-common
```

**What it does:** any server in `roboshop-common` can now SSH to any other server in `roboshop-common`. The Ansible server and all 10 RoboShop servers are in it.

**Expected:** `"Return": true` ✅. (`InvalidPermission.Duplicate` means the rule is already there, which is fine.)

### Step 3: Create the Ansible server (EC2)

```bash
aws ec2 run-instances \
  --image-id ami-0055977a6e828a4f4 \
  --instance-type t3.micro \
  --security-groups roboshop-common \
  --iam-instance-profile Name=ansible-server-role \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ansible-server}]' \
  --query 'Instances[0].InstanceId' --output text
```

**What it does:** creates the RHEL 9 Ansible server with the role from Step 1 and the security group from Step 2.

**Expected:** an instance ID like `i-0abc123def4567890` ✅.

| Error | Fix |
|---|---|
| `Invalid IAM Instance Profile name` | A new profile takes a few seconds to be ready. Wait 10 seconds and run it again |

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ansible-server" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].PublicIpAddress' --output text
```

**Expected:** the server's public IP ✅.

---

## Part 2: Prepare the Ansible server

### Step 4: Log in to the Ansible server

```bash
ssh ec2-user@<ansible-server-public-ip>
```

Type `yes` the first time, then the SSH password (nothing shows while you type).

### Step 5: Install Ansible and the collections

```bash
sudo dnf install ansible-core git -y
```

```bash
ansible-galaxy collection install amazon.aws community.general community.rabbitmq
```

**What it does:** `ansible-core` has only the built-in modules. The **collections** add the ones we need: `amazon.aws` (EC2, Route 53), `community.general` (npm) and `community.rabbitmq` (RabbitMQ user).

### Step 6: Install boto3 and check AWS access

```bash
sudo pip3.9 install botocore boto3
```

**What it does:** the `amazon.aws` modules call AWS through **boto3**, so it must be installed for Python 3.9, the Python Ansible uses on RHEL 9.

```bash
aws sts get-caller-identity
```

**Expected:** `Arn` shows `assumed-role/ansible-server-role/…` ✅. The server has AWS access with no keys.

### Step 7: Get the code

```bash
git clone https://github.com/Abhignadumpala/roboshop-with-ansible-v1.git
```

```bash
cd roboshop-with-ansible-v1
```

**What it does:** downloads the playbooks, templates and `group_vars/` to the Ansible server. We run every `ansible-playbook` command from inside this folder, so Ansible finds `ansible.cfg` and the inventory.

### Step 8: Set your domain

The server names in the inventory, `group_vars/all.yaml` and `roboshop.yaml` use the domain `abhignadevops.store`. If your hosted zone is different, change all of them in one go:

```bash
sed -i 's/abhignadevops.store/<your-domain>/g' inventory.ini group_vars/all.yaml roboshop.yaml
```

```bash
grep -rn "domain_name\|_host" roboshop.yaml group_vars/all.yaml
```

**Expected:** every name ends with your domain ✅.

### Step 9: Set the SSH password

```bash
vim group_vars/all.yaml
```

Set `ansible_password:` to the SSH password of the RoboShop servers (`ansible_user` is `ec2-user`).

> ⚠️ Never push the real password to GitHub. Keep `<ssh-password>` in the repo and fill it in only on the server.

### Step 10: Create the log folder

```bash
mkdir -p ~/roboshop-logs
```

**What it does:** `ansible.cfg` has `log_path = /home/ec2-user/roboshop-logs/ansible.log`, so Ansible saves every run's output there. The folder must exist first.

---

## Part 3: Run RoboShop

### Step 11: Create the 10 servers and DNS records

```bash
ansible-playbook roboshop.yaml
```

**Expected:** 10 servers `roboshop-mongodb` … `roboshop-frontend` in the EC2 console, and a Route 53 record for each one ✅.

### Step 12: Check Ansible can reach every server

```bash
ansible all -m ping
```

**Expected:** every server shows `"ping": "pong"` ✅.

| Error | Cause | Fix |
|---|---|---|
| `UNREACHABLE … timed out` | No SSH rule from the Ansible server | Step 2 |
| `UNREACHABLE … Permission denied` | Wrong password | Fix `group_vars/all.yaml` |
| `Could not resolve hostname` | Records not created yet | Run `roboshop.yaml` again |

### Step 13: Run the playbooks in order

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

### Step 14: Open the shop

Open `http://abhignadevops.store` in the browser.

**Expected:** the RoboShop home page with products. Add an item to the cart, then check out ✅.

---

## Part 4: Clean up

### Step 15: Delete the RoboShop servers and records (on the Ansible server)

```bash
ansible-playbook roboshop.yaml -e "action=delete"
```

**Expected:** all 10 servers terminating and their records gone ✅.

### Step 16: Delete the Ansible server (on your laptop)

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
| `Invalid IAM Instance Profile name` | Profile not ready yet | Wait 10 seconds, run Step 3 again |
| `Unable to locate credentials` | No IAM role on the Ansible server | Attach `ansible-server-role` (EC2 → Actions → Security → Modify IAM role) |
| `Failed to import the required Python library (botocore and boto3)` | boto3 installed for a different Python | `sudo pip3.9 install botocore boto3` |
| `couldn't resolve module/action 'amazon.aws.ec2_instance'` | Collection not installed | Step 5 |
| `Invalid callback for stdout specified: yaml` | New `community.general` versions removed the `yaml` output | Delete the `stdout_callback=yaml` line in `ansible.cfg` |
| `InvalidGroup.NotFound` | A `roboshop-<name>` security group is missing | Create it in the same region (`us-east-1`) |
| `UNREACHABLE … timed out` | No SSH rule from the Ansible server | Step 2 |
| `UNREACHABLE … Permission denied` | Wrong SSH password | Step 9 |
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
