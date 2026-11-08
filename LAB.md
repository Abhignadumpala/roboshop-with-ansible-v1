# RoboShop with Ansible v1: Lab Step by Step

[← Back to README](README.md)

> **In short:** from the laptop we create an IAM role, one security group rule and the **Ansible server**. We write the project files in this repo and push them. On the Ansible server we clone the repo and then build RoboShop **one server at a time**: write its playbook → push → pull → create the server → run the playbook → check it. At the end we open the shop and delete everything.

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

**Part 2: Project setup files (on your laptop)**
- [Step 9: Go to the repo](#step-9-go-to-the-repo)
- [Step 10: Create ansible.cfg](#step-10-create-ansiblecfg)
- [Step 11: Create inventory.ini](#step-11-create-inventoryini)
- [Step 12: Create group_vars/all.yaml](#step-12-create-group_varsallyaml)
- [Step 13: Create the group-specific variable files](#step-13-create-the-group-specific-variable-files)
- [Step 14: Create roboshop.yaml](#step-14-create-roboshopyaml)
- [Step 15: Push the setup files](#step-15-push-the-setup-files)

**Part 3: Prepare the Ansible server**
- [Step 16: Log in to the Ansible server](#step-16-log-in-to-the-ansible-server)
- [Step 17: Install Ansible and the collections](#step-17-install-ansible-and-the-collections)
- [Step 18: Install boto3 and check AWS access](#step-18-install-boto3-and-check-aws-access)
- [Step 19: Clone the repo](#step-19-clone-the-repo)
- [Step 20: Set the SSH password](#step-20-set-the-ssh-password)
- [Step 21: Create the log folder](#step-21-create-the-log-folder)

**Part 4: Build RoboShop one server at a time**
- [Server 1: MongoDB](#server-1-mongodb)
- [Server 2: Redis](#server-2-redis)
- [Server 3: MySQL](#server-3-mysql)
- [Server 4: RabbitMQ](#server-4-rabbitmq)
- [Server 5: Catalogue](#server-5-catalogue)
- [Server 6: User](#server-6-user)
- [Server 7: Cart](#server-7-cart)
- [Server 8: Shipping](#server-8-shipping)
- [Server 9: Payment](#server-9-payment)
- [Server 10: Frontend](#server-10-frontend)

**Part 5: Open the shop**
- [Step 22: Open the website](#step-22-open-the-website)

**Part 6: Clean up**
- [Step 23: Delete the RoboShop servers and records (Ansible server)](#step-23-delete-the-roboshop-servers-and-records-ansible-server)
- [Step 24: Delete the Ansible server (laptop)](#step-24-delete-the-ansible-server-laptop)
- [Troubleshooting](#troubleshooting)
- [What I learned](#what-i-learned)

---

## How this lab works

**Definition:** in this lab one **Ansible server** (the control node) creates the 10 RoboShop servers on AWS and then sets each one up with a playbook. We never log in to the RoboShop servers by hand. Ansible does all the work over SSH.

```
 Laptop ──(aws cli)──► IAM role + SG rule + Ansible server
 Laptop ──(git push)─► this repo ──(git pull)──► Ansible server
                                                     ├─ roboshop.yaml ──► AWS: EC2 server + Route 53 record
                                                     └─ 0X-<name>.yaml ─► SSH: install + configure that server
```

| Part | Where | What we do |
|---|---|---|
| **Part 1: AWS setup** | Laptop | IAM role, security group rule, Ansible server |
| **Part 2: Project setup files** | Laptop | `ansible.cfg`, inventory, `group_vars/`, `roboshop.yaml` → push |
| **Part 3: Prepare the Ansible server** | Ansible server | Install Ansible + boto3, clone the repo |
| **Part 4: Build RoboShop one server at a time** | Laptop + Ansible server | For each of the 10 servers: write code → push → pull → create → configure → check |
| **Part 5: Open the shop** | Browser | Test the website |
| **Part 6: Clean up** | Ansible server + laptop | Delete everything |

**Before you start:**

| Need | Why |
|---|---|
| AWS CLI on your laptop, logged in as an admin user | Creates the role and the Ansible server |
| Security groups `roboshop-common` + `roboshop-<name>` for every server | Each server gets both; the playbook doesn't create them |
| Route 53 hosted zone (`abhignadevops.store`) | Where the DNS records go. Using another domain? Replace `abhignadevops.store` with yours in every file |
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

In my case, I already created the `roboshop-common` security group in my previous RoboShop lab, so I'm **reusing the same one** and only **adding an inbound rule** to it.

**If you're doing this lab for the first time**, or you don't have `roboshop-common` yet, first create the security group (6a), then add the inbound rule (6b). If you already have it, skip to 6b.

**Why do we need this rule?** `roboshop-common` allows SSH only from your home IP. The Ansible server runs inside AWS with a different IP, so without this rule it can't SSH to the RoboShop servers (the connection times out).

#### 6a: Create the security group (first time only)

```bash
aws ec2 create-security-group \
  --group-name roboshop-common \
  --description "SSH for all RoboShop servers"
```

**What it does:** creates an empty security group called `roboshop-common`.

```bash
aws ec2 authorize-security-group-ingress \
  --group-name roboshop-common \
  --protocol tcp \
  --port 22 \
  --cidr $(curl -s checkip.amazonaws.com)/32
```

**What it does:** allows SSH (port 22) from **your laptop's public IP only**, so you can log in to the servers.

#### 6b: Add the inbound rule for the Ansible server

```bash
aws ec2 authorize-security-group-ingress \
  --group-name roboshop-common \
  --protocol tcp \
  --port 22 \
  --source-group roboshop-common
```

**What it does:** adds the rule **SSH, port 22, source `roboshop-common`**. Any server in this group can now SSH to any other server in it. The Ansible server and all 10 RoboShop servers are in it.

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

## Part 2: Project setup files (on your laptop)

These files are used by every playbook. We write them once and push them.

```
 roboshop-with-ansible-v1/
 ├── ansible.cfg          ← Ansible settings
 ├── inventory.ini        ← list of servers (by DNS name)
 ├── group_vars/
 │   ├── all.yaml         ← variables for every server
 │   ├── cart.yaml        ← variables only for cart
 │   ├── mongodb.yaml
 │   └── mysql.yaml
 ├── roboshop.yaml        ← creates/deletes servers + DNS records
 ├── 01-mongodb.yaml … 10-frontend.yaml   ← Part 4, one by one
 └── *.repo, *.j2         ← files the playbooks copy to the servers
```

### Step 9: Go to the repo

```bash
cd ~/roboshop-with-ansible-v1
```

```bash
git pull
```

**What it does:** opens the local copy of the repo and gets the latest changes.

### Step 10: Create ansible.cfg

```bash
vim ansible.cfg
```

```ini
[defaults]
inventory=inventory.ini
host_key_checking=False
log_path=/home/ec2-user/roboshop-logs/ansible.log
```

| Line | Why |
|---|---|
| `inventory=inventory.ini` | No need to type `-i inventory.ini` on every command |
| `host_key_checking=False` | Skips the `yes/no` host key question on the first SSH. Without it, password login to new servers fails |
| `log_path=...` | Saves the output of every run to a log file |

### Step 11: Create inventory.ini

```bash
vim inventory.ini
```

```ini
[local]
localhost

[mongodb]
mongodb.abhignadevops.store

[redis]
redis.abhignadevops.store

[mysql]
mysql.abhignadevops.store

[rabbitmq]
rabbitmq.abhignadevops.store

[catalogue]
catalogue.abhignadevops.store

[user]
user.abhignadevops.store

[cart]
cart.abhignadevops.store

[shipping]
shipping.abhignadevops.store

[payment]
payment.abhignadevops.store

[frontend]
frontend.abhignadevops.store
```

**What it does:** one group per server, listed by its **DNS name** (created by `roboshop.yaml`), so the inventory never changes when IPs change. `[local]` is the Ansible server itself, used by `roboshop.yaml`.

### Step 12: Create group_vars/all.yaml

```bash
mkdir group_vars
```

```bash
vim group_vars/all.yaml
```

```yaml
ansible_user: ec2-user
ansible_password: <ssh-password>
mongodb_host: mongodb.abhignadevops.store
redis_host: redis.abhignadevops.store
mysql_host: mysql.abhignadevops.store
rabbitmq_host: rabbitmq.abhignadevops.store
catalogue_host: catalogue.abhignadevops.store
user_host: user.abhignadevops.store
cart_host: cart.abhignadevops.store
shipping_host: shipping.abhignadevops.store
payment_host: payment.abhignadevops.store
mysql_root_password: RoboShop@1
shipping_db_user: shipping
shipping_db_password: RoboShop@1
rabbitmq_user: roboshop
rabbitmq_password: roboshop123
```

**What it does:** variables for **every** server: the SSH login, the DNS name of each server, and the database passwords. Playbooks and templates read them as `{{ mongodb_host }}`, etc.

> ⚠️ Keep `<ssh-password>` in the repo (it's public). We fill in the real password only on the Ansible server (Step 20).

### Step 13: Create the group-specific variable files

```bash
vim group_vars/cart.yaml
```

```yaml
app_version: v4
```

```bash
vim group_vars/mongodb.yaml
```

```yaml
app_name: mongodb
```

```bash
vim group_vars/mysql.yaml
```

```yaml
mysql_root_password: RoboShop@1
```

**What it does:** a file named after a group applies **only** to that group: `cart.yaml` sets the cart app version, `mysql.yaml` the MySQL root password.

### Step 14: Create roboshop.yaml

```bash
vim roboshop.yaml
```

```yaml
- name: create ec2 instances and r53 records
  hosts: local
  connection: local
  vars:
    # ansible-playbook roboshop.yaml -e "instances=['mongodb', 'redis']"
    # ansible-playbook roboshop.yaml -e "action=delete"
    instances: ["mongodb", "redis", "mysql", "rabbitmq", "catalogue", "user", "cart", "shipping", "payment", "frontend"]
    action: create
    domain_name: abhignadevops.store
    ami_id: ami-0055977a6e828a4f4
  tasks:
  - name: create ec2 instance
    amazon.aws.ec2_instance:
      name: "roboshop-{{ item }}"
      image_id: "{{ ami_id }}"
      instance_type: t3.micro
      security_groups:
      - roboshop-common
      - "roboshop-{{ item }}"
      region: us-east-1
    loop: "{{ instances }}"
    when: action == "create"
    register: ec2_output

  # - name: print ec2 creation output
  #   debug:
  #     msg: "{{ ec2_output }}"

  - name: create R53 record
    amazon.aws.route53:
      state: present
      zone: "{{ domain_name }}"
      record: "{{ item.item }}.{{ domain_name }}" # mongodb.abhignadevops.store
      type: A
      ttl: 1
      value: "{{ item.instances[0].private_ip_address }}"
      overwrite: true
    loop: "{{ ec2_output.results }}"
    when: action == "create"

  - name: create R53 record for frontend
    amazon.aws.route53:
      state: present
      zone: "{{ domain_name }}"
      record: "{{ domain_name }}" # abhignadevops.store points to frontend public IP
      type: A
      ttl: 1
      value: "{{ item.instances[0].public_ip_address }}"
      overwrite: true
    loop: "{{ ec2_output.results }}"
    when: action == "create" and item.item == "frontend"

  - name: delete R53 records
    amazon.aws.route53:
      state: absent
      zone: "{{ domain_name }}"
      record: "{{ item }}.{{ domain_name }}"
      ttl: 1
      type: A
    loop: "{{ instances }}"
    when: action == "delete"

  - name: delete R53 record for frontend
    amazon.aws.route53:
      state: absent
      zone: "{{ domain_name }}"
      record: "{{ domain_name }}"
      ttl: 1
      type: A
    when: action == "delete"

  - name: delete ec2 instances
    amazon.aws.ec2_instance:
      state: absent
      name: "roboshop-{{ item }}"
      region: us-east-1
      wait: false
    loop: "{{ instances }}"
    when: action == "delete"
```

| Part | What it does |
|---|---|
| `vars:` | Defaults: all 10 servers, `action: create`, our domain, the AMI |
| `create ec2 instance` | Creates `roboshop-<name>` with 2 security groups: `roboshop-common` + `roboshop-<name>` |
| `create R53 record` | `<name>.abhignadevops.store` → that server's **private IP** |
| `create R53 record for frontend` | `abhignadevops.store` → frontend's **public IP** |
| `delete …` tasks | Run only with `-e "action=delete"` |

> 💡 `-e "instances=['mongodb']"` replaces the list of 10 with one server. That's how we create the servers **one at a time** in Part 4.

### Step 15: Push the setup files

```bash
git add ansible.cfg inventory.ini group_vars roboshop.yaml
```

```bash
git commit -m "Add project setup files"
```

```bash
git push
```

**Expected:** the files show up on GitHub ✅.

---

## Part 3: Prepare the Ansible server

### Step 16: Log in to the Ansible server

```bash
ssh ec2-user@<ansible-server-public-ip>
```

Type `yes` the first time, then the SSH password (nothing shows while you type).

### Step 17: Install Ansible and the collections

```bash
sudo dnf install ansible-core git -y
```

```bash
ansible-galaxy collection install amazon.aws community.general community.rabbitmq
```

**What it does:** `ansible-core` has only the built-in modules. The **collections** add the ones we need: `amazon.aws` (EC2, Route 53), `community.general` (npm) and `community.rabbitmq` (RabbitMQ user).

### Step 18: Install boto3 and check AWS access

```bash
sudo pip3.9 install botocore boto3
```

**What it does:** the `amazon.aws` modules call AWS through **boto3**, so it must be installed for Python 3.9, the Python Ansible uses on RHEL 9.

```bash
aws sts get-caller-identity
```

**Expected:** `Arn` shows `assumed-role/ansible-server-role/…` ✅. The server has AWS access with no keys.

### Step 19: Clone the repo

```bash
git clone https://github.com/Abhignadumpala/roboshop-with-ansible-v1.git
```

```bash
cd roboshop-with-ansible-v1
```

**What it does:** downloads the setup files from Part 2. We run every `ansible-playbook` command from inside this folder, so Ansible finds `ansible.cfg` and the inventory.

### Step 20: Set the SSH password

```bash
vim group_vars/all.yaml
```

Replace `<ssh-password>` with the SSH password of the RoboShop servers. **Don't commit or push this change.**

### Step 21: Create the log folder

```bash
mkdir -p ~/roboshop-logs
```

**What it does:** `ansible.cfg` writes its log to `/home/ec2-user/roboshop-logs/ansible.log`. The folder must exist first.

---

## Part 4: Build RoboShop one server at a time

For **every** server we repeat the same loop:

```
 Laptop: write the code ─► git push ─► Ansible server: git pull ─► create the server ─► ping ─► run the playbook ─► check
```

| Order | Server | Why this order |
|---|---|---|
| 1–4 | mongodb, redis, mysql, rabbitmq | Databases first: the apps connect to them and load data into them |
| 5–9 | catalogue, user, cart, shipping, payment | Backend services |
| 10 | frontend | Last: it forwards requests to all the backend services |

> 💡 While building, `ansible all -m ping` shows the servers that don't exist yet as `UNREACHABLE`. That's expected. Ping only the group you just created.

### Server 1: MongoDB

**MongoDB** is the NoSQL database for products and users.

#### 1a. Write the code (laptop)

```bash
vim mongo.repo
```

```ini
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/9/mongodb-org/7.0/x86_64/
enabled=1
gpgcheck=0
```

`mongo.repo` tells `dnf` where to download MongoDB 7.0 from. The playbook copies it to `/etc/yum.repos.d/`.

```bash
vim 01-mongodb.yaml
```

```yaml
- name: configure mongodb server
  hosts: mongodb
  become: yes
  tasks:
  - name: copy mongo repo
    copy:
      src: mongo.repo
      dest: /etc/yum.repos.d/mongo.repo

  # - name: test the inventory variable
  #   debug:
  #     msg: "{{ app_name }}"

  - name: install mongodb server
    package:
      name: mongodb-org
      state: present

  - name: start and enable mongodb
    service:
      name: mongod
      state: started
      enabled: yes

  - name: allow remote connections
    replace:
      path: /etc/mongod.conf
      regexp: '127.0.0.1'
      replace: '0.0.0.0'

  - name: restart mongod
    service:
      name: mongod
      state: restarted
```

| Task | What it does |
|---|---|
| `copy mongo repo` | Adds the MongoDB package repo to the server |
| `install mongodb server` | Installs `mongodb-org` |
| `start and enable mongodb` | Starts it now and on every boot |
| `allow remote connections` | Changes `127.0.0.1` to `0.0.0.0` in `/etc/mongod.conf`, so other servers can connect |
| `restart mongod` | Restarts so the new config is used |

#### 1b. Push it (laptop)

```bash
git add mongo.repo 01-mongodb.yaml
```

```bash
git commit -m "Add mongodb playbook"
```

```bash
git push
```

#### 1c. Pull the code (Ansible server)

```bash
git pull
```

#### 1d. Create the mongodb server and DNS record

```bash
ansible-playbook roboshop.yaml -e "instances=['mongodb']"
```

**Expected:** `roboshop-mongodb` in the EC2 console and the record `mongodb.abhignadevops.store` ✅.

#### 1e. Check Ansible can reach it

```bash
ansible mongodb -m ping
```

**Expected:** `"ping": "pong"` ✅. (Just created? Wait a minute for the server to boot, then try again.)

#### 1f. Configure it

```bash
ansible-playbook 01-mongodb.yaml
```

**Expected:** the recap shows `failed=0` ✅.

#### 1g. Check the service is running

```bash
ansible mongodb -b -m command -a "systemctl is-active mongod"
```

**Expected:** `active` ✅. If not: `ansible mongodb -b -m command -a "journalctl -u mongod -n 20"` shows why.

### Server 2: Redis

**Redis** is the in-memory database for carts and login sessions.

#### 2a. Write the code (laptop)

```bash
vim 02-redis.yaml
```

```yaml
- name: configure redis server
  hosts: redis
  become: yes
  tasks:
  - name: enable redis 7
    command: dnf module enable redis:7 -y

  - name: install redis
    package:
      name: redis
      state: present

  - name: allow remote connections
    replace:
      path: /etc/redis/redis.conf
      regexp: '127.0.0.1'
      replace: '0.0.0.0'

  - name: disable protected mode
    lineinfile:
      path: /etc/redis/redis.conf
      regexp: '^#?protected-mode yes'
      line: 'protected-mode no'

  - name: enable and start redis
    service:
      name: redis
      state: restarted
      enabled: yes
```

| Task | What it does |
|---|---|
| `enable redis 7` | Picks Redis version 7 |
| `install redis` | Installs Redis |
| `allow remote connections` | `127.0.0.1` → `0.0.0.0` so other servers can connect |
| `disable protected mode` | Lets other servers connect without a password |
| `enable and start redis` | Restarts with the new config and starts on boot |

#### 2b. Push it (laptop)

```bash
git add 02-redis.yaml
```

```bash
git commit -m "Add redis playbook"
```

```bash
git push
```

#### 2c. Pull the code (Ansible server)

```bash
git pull
```

#### 2d. Create the redis server and DNS record

```bash
ansible-playbook roboshop.yaml -e "instances=['redis']"
```

**Expected:** `roboshop-redis` in the EC2 console and the record `redis.abhignadevops.store` ✅.

#### 2e. Check Ansible can reach it

```bash
ansible redis -m ping
```

**Expected:** `"ping": "pong"` ✅. (Just created? Wait a minute for the server to boot, then try again.)

#### 2f. Configure it

```bash
ansible-playbook 02-redis.yaml
```

**Expected:** the recap shows `failed=0` ✅.

#### 2g. Check the service is running

```bash
ansible redis -b -m command -a "systemctl is-active redis"
```

**Expected:** `active` ✅. If not: `ansible redis -b -m command -a "journalctl -u redis -n 20"` shows why.

### Server 3: MySQL

**MySQL** is the SQL database for the shipping cities.

#### 3a. Write the code (laptop)

```bash
vim 03-mysql.yaml
```

```yaml
- name: configure mysql server
  hosts: mysql
  become: yes
  tasks:
  - name: enable mysql 8.4
    command: dnf module enable mysql:8.4 -y

  - name: install mysql server
    package:
      name: mysql-server
      state: present

  - name: enable and start mysql
    service:
      name: mysqld
      state: started
      enabled: yes

  # - name: setup mysql root password
  #   ansible.mysql.mysql_user:
  #     name: root
  #     host: localhost
  #     password: "{{ mysql_root_password }}"
  #     check_implicit_admin: true
  #     state: present

  - name: setup mysql root password
    command: mysql_secure_installation --set-root-pass "{{ mysql_root_password }}"
```

| Task | What it does |
|---|---|
| `enable mysql 8.4` | Picks MySQL version 8.4 |
| `install mysql server` | Installs MySQL |
| `enable and start mysql` | Starts it now and on boot |
| `setup mysql root password` | Sets the root password from `group_vars/mysql.yaml` |

#### 3b. Push it (laptop)

```bash
git add 03-mysql.yaml
```

```bash
git commit -m "Add mysql playbook"
```

```bash
git push
```

#### 3c. Pull the code (Ansible server)

```bash
git pull
```

#### 3d. Create the mysql server and DNS record

```bash
ansible-playbook roboshop.yaml -e "instances=['mysql']"
```

**Expected:** `roboshop-mysql` in the EC2 console and the record `mysql.abhignadevops.store` ✅.

#### 3e. Check Ansible can reach it

```bash
ansible mysql -m ping
```

**Expected:** `"ping": "pong"` ✅. (Just created? Wait a minute for the server to boot, then try again.)

#### 3f. Configure it

```bash
ansible-playbook 03-mysql.yaml
```

**Expected:** the recap shows `failed=0` ✅.

#### 3g. Check the service is running

```bash
ansible mysql -b -m command -a "systemctl is-active mysqld"
```

**Expected:** `active` ✅. If not: `ansible mysql -b -m command -a "journalctl -u mysqld -n 20"` shows why.

### Server 4: RabbitMQ

**RabbitMQ** is the message queue: payment drops orders in it.

#### 4a. Write the code (laptop)

```bash
vim rabbitmq.repo
```

```ini
[modern-erlang]
name=modern-erlang-el9
baseurl=https://yum1.rabbitmq.com/erlang/el/9/$basearch
        https://yum2.rabbitmq.com/erlang/el/9/$basearch
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[modern-erlang-noarch]
name=modern-erlang-el9-noarch
baseurl=https://yum1.rabbitmq.com/erlang/el/9/noarch
        https://yum2.rabbitmq.com/erlang/el/9/noarch
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key
       https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[rabbitmq-el9]
name=rabbitmq-el9
baseurl=https://yum2.rabbitmq.com/rabbitmq/el/9/$basearch
        https://yum1.rabbitmq.com/rabbitmq/el/9/$basearch
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key
       https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md

[rabbitmq-el9-noarch]
name=rabbitmq-el9-noarch
baseurl=https://yum2.rabbitmq.com/rabbitmq/el/9/noarch
        https://yum1.rabbitmq.com/rabbitmq/el/9/noarch
repo_gpgcheck=1
enabled=1
gpgkey=https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key
       https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc
gpgcheck=1
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
metadata_expire=300
pkg_gpgcheck=1
autorefresh=1
type=rpm-md
```

`rabbitmq.repo` tells `dnf` where to download RabbitMQ and Erlang from.

```bash
vim 04-rabbitmq.yaml
```

```yaml
- name: configure rabbitmq server
  hosts: rabbitmq
  become: yes
  tasks:
  - name: copy rabbitmq repo
    copy:
      src: rabbitmq.repo
      dest: /etc/yum.repos.d/rabbitmq.repo

  - name: install rabbitmq server
    dnf:
      name:
      - erlang
      - rabbitmq-server
      state: present

  - name: start and enable rabbitmq server
    service:
      name: rabbitmq-server
      state: started
      enabled: yes

  - name: create rabbitmq user
    community.rabbitmq.rabbitmq_user:
      user: roboshop
      password: roboshop123
      vhost: /
      configure_priv: .*
      read_priv: .*
      write_priv: .*
      state: present
```

| Task | What it does |
|---|---|
| `copy rabbitmq repo` | Adds the RabbitMQ and Erlang package repos |
| `install rabbitmq server` | Installs Erlang (the language RabbitMQ runs on) + RabbitMQ |
| `start and enable rabbitmq server` | Starts it now and on boot |
| `create rabbitmq user` | Creates the `roboshop` user that payment uses |

#### 4b. Push it (laptop)

```bash
git add rabbitmq.repo 04-rabbitmq.yaml
```

```bash
git commit -m "Add rabbitmq playbook"
```

```bash
git push
```

#### 4c. Pull the code (Ansible server)

```bash
git pull
```

#### 4d. Create the rabbitmq server and DNS record

```bash
ansible-playbook roboshop.yaml -e "instances=['rabbitmq']"
```

**Expected:** `roboshop-rabbitmq` in the EC2 console and the record `rabbitmq.abhignadevops.store` ✅.

#### 4e. Check Ansible can reach it

```bash
ansible rabbitmq -m ping
```

**Expected:** `"ping": "pong"` ✅. (Just created? Wait a minute for the server to boot, then try again.)

#### 4f. Configure it

```bash
ansible-playbook 04-rabbitmq.yaml
```

**Expected:** the recap shows `failed=0` ✅.

#### 4g. Check the service is running

```bash
ansible rabbitmq -b -m command -a "systemctl is-active rabbitmq-server"
```

**Expected:** `active` ✅. If not: `ansible rabbitmq -b -m command -a "journalctl -u rabbitmq-server -n 20"` shows why.

### Server 5: Catalogue

**catalogue** is the Node.js service for the product list. It reads from MongoDB. It also needs `mongo.repo` (already written for MongoDB) to install the MongoDB client.

#### 5a. Write the code (laptop)

```bash
vim catalogue.service.j2
```

```ini
[Unit]
Description=Catalogue Service
After=network-online.target

[Service]
User=roboshop
Environment=MONGO_URL="mongodb://{{ mongodb_host }}:27017/catalogue"
ExecStart=/bin/node /app/server.js
SyslogIdentifier=catalogue
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`catalogue.service.j2` is the systemd service file. `{{ mongodb_host }}` is replaced with `mongodb.abhignadevops.store` when the playbook copies it.

```bash
vim 05-catalogue.yaml
```

```yaml
- name: configure catalogue server
  hosts: catalogue
  become: yes
  tasks:
  - name: disable default nodejs
    command: dnf module disable nodejs -y

  - name: enable nodejs24
    command: dnf module enable nodejs:24 -y

  - name: install nodejs
    dnf:
      name: nodejs
      state: present

  - name: create roboshop system user
    user:
      name: roboshop
      create_home: false
      home: /app
      system: true
      shell: /sbin/nologin
      comment: "roboshop system user"

  - name: remove existing app directory
    file:
      path: /app
      state: absent

  - name: remove catalogue.zip
    file:
      path: /tmp/catalogue.zip
      state: absent

  - name: create app directory
    file:
      path: /app
      state: directory

  - name: download catalogue application
    get_url:
      url: https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/catalogue-v4.zip
      dest: /tmp/catalogue.zip

  - name: unzip catalogue
    unarchive:
      src: /tmp/catalogue.zip
      dest: /app
      remote_src: yes

  - name: install dependencies
    community.general.npm:
      path: /app

  - name: copy catalogue service
    template:
      src: catalogue.service.j2
      dest: /etc/systemd/system/catalogue.service

  - name: copy mongo repo
    copy:
      src: mongo.repo
      dest: /etc/yum.repos.d/mongo.repo

  - name: install mongodb client
    dnf:
      name: mongodb-mongosh
      state: present

  - name: check products loaded or not
    command: mongosh --host "{{ mongodb_host }}" --quiet --eval 'db.getSiblingDB("catalogue").products.countDocuments()'
    register: product_count

  - name: print product_count
    debug:
      msg: "{{ product_count }}"

  - name: load the products
    shell: mongosh --host "{{ mongodb_host }}" < /app/db/master-data.js
    when: product_count.stdout | int == 0

  - name: daemon reload
    systemd_service:
      daemon_reload: true

  - name: start and enable catalogue
    service:
      name: catalogue
      state: restarted
      enabled: yes
```

| Task | What it does |
|---|---|
| `disable/enable nodejs, install nodejs` | Installs Node.js 24 |
| `create roboshop system user` | A user with no login that runs the app |
| `remove … / create app directory` | Deletes the old code and makes an empty `/app` |
| `download + unzip catalogue` | Gets the code into `/app` (`remote_src: yes`: the zip is on the server) |
| `install dependencies` | `npm install` in `/app` |
| `copy catalogue service` | Fills the template and creates the systemd service |
| `copy mongo repo + install mongodb client` | Installs `mongosh` |
| `check products loaded / load the products` | Loads the products into MongoDB **only if** there are none |
| `daemon reload + start and enable` | Tells systemd about the new service and starts it |

#### 5b. Push it (laptop)

```bash
git add catalogue.service.j2 05-catalogue.yaml
```

```bash
git commit -m "Add catalogue playbook"
```

```bash
git push
```

#### 5c. Pull the code (Ansible server)

```bash
git pull
```

#### 5d. Create the catalogue server and DNS record

```bash
ansible-playbook roboshop.yaml -e "instances=['catalogue']"
```

**Expected:** `roboshop-catalogue` in the EC2 console and the record `catalogue.abhignadevops.store` ✅.

#### 5e. Check Ansible can reach it

```bash
ansible catalogue -m ping
```

**Expected:** `"ping": "pong"` ✅. (Just created? Wait a minute for the server to boot, then try again.)

#### 5f. Configure it

```bash
ansible-playbook 05-catalogue.yaml
```

**Expected:** the recap shows `failed=0` ✅.

#### 5g. Check the service is running

```bash
ansible catalogue -b -m command -a "systemctl is-active catalogue"
```

**Expected:** `active` ✅. If not: `ansible catalogue -b -m command -a "journalctl -u catalogue -n 20"` shows why.

### Server 6: User

**user** is the Node.js service for login and registration. It uses MongoDB and Redis.

#### 6a. Write the code (laptop)

```bash
vim user.service.j2
```

```ini
[Unit]
Description=User Service
After=network-online.target

[Service]
User=roboshop
Environment=REDIS_URL="redis://{{ redis_host }}:6379"
Environment=MONGO_URL="mongodb://{{ mongodb_host }}:27017/users"
ExecStart=/bin/node /app/server.js
SyslogIdentifier=user
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`user.service.j2` points the app to Redis (`REDIS_URL`) and MongoDB (`MONGO_URL`).

```bash
vim 06-user.yaml
```

```yaml
- name: configure user server
  hosts: user
  become: yes
  tasks:
  - name: disable default nodejs
    command: dnf module disable nodejs -y

  - name: enable nodejs24
    command: dnf module enable nodejs:24 -y

  - name: install nodejs
    dnf:
      name: nodejs
      state: present

  - name: create roboshop system user
    user:
      name: roboshop
      create_home: false
      home: /app
      system: true
      shell: /sbin/nologin
      comment: "roboshop system user"

  - name: remove existing app directory
    file:
      path: /app
      state: absent

  - name: remove user.zip
    file:
      path: /tmp/user.zip
      state: absent

  - name: create app directory
    file:
      path: /app
      state: directory

  - name: download user application
    get_url:
      url: https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/user-v4.zip
      dest: /tmp/user.zip

  - name: unzip user
    unarchive:
      src: /tmp/user.zip
      dest: /app
      remote_src: yes

  - name: install dependencies
    community.general.npm:
      path: /app

  - name: copy user service
    template:
      src: user.service.j2
      dest: /etc/systemd/system/user.service

  - name: daemon reload
    systemd_service:
      daemon_reload: true

  - name: start and enable user
    service:
      name: user
      state: restarted
      enabled: yes
```

| Task | What it does |
|---|---|
| `install nodejs → install dependencies` | Same as catalogue: Node.js, app user, code in `/app`, `npm install` |
| `copy user service` | Creates the systemd service from the template |
| `daemon reload + start and enable` | Starts the service |

#### 6b. Push it (laptop)

```bash
git add user.service.j2 06-user.yaml
```

```bash
git commit -m "Add user playbook"
```

```bash
git push
```

#### 6c. Pull the code (Ansible server)

```bash
git pull
```

#### 6d. Create the user server and DNS record

```bash
ansible-playbook roboshop.yaml -e "instances=['user']"
```

**Expected:** `roboshop-user` in the EC2 console and the record `user.abhignadevops.store` ✅.

#### 6e. Check Ansible can reach it

```bash
ansible user -m ping
```

**Expected:** `"ping": "pong"` ✅. (Just created? Wait a minute for the server to boot, then try again.)

#### 6f. Configure it

```bash
ansible-playbook 06-user.yaml
```

**Expected:** the recap shows `failed=0` ✅.

#### 6g. Check the service is running

```bash
ansible user -b -m command -a "systemctl is-active user"
```

**Expected:** `active` ✅. If not: `ansible user -b -m command -a "journalctl -u user -n 20"` shows why.

### Server 7: Cart

**cart** is the Node.js service for the shopping cart. It uses Redis and asks catalogue for product details.

#### 7a. Write the code (laptop)

```bash
vim cart.service.j2
```

```ini
[Unit]
Description=Cart Service
After=network-online.target

[Service]
User=roboshop
Environment=REDIS_HOST={{ redis_host }}
Environment=CATALOGUE_HOST={{ catalogue_host }}
Environment=CATALOGUE_PORT=8080
ExecStart=/bin/node /app/server.js
SyslogIdentifier=cart
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`cart.service.j2` points the app to Redis and catalogue.

```bash
vim 07-cart.yaml
```

```yaml
- name: configure cart server
  hosts: cart
  become: yes
  tasks:
  - name: disable default nodejs
    command: dnf module disable nodejs -y

  - name: enable nodejs24
    command: dnf module enable nodejs:24 -y

  - name: install nodejs
    dnf:
      name: nodejs
      state: present

  - name: create roboshop system user
    user:
      name: roboshop
      create_home: false
      home: /app
      system: true
      shell: /sbin/nologin
      comment: "roboshop system user"

  - name: remove existing app directory
    file:
      path: /app
      state: absent

  - name: remove cart.zip
    file:
      path: /tmp/cart.zip
      state: absent

  - name: create app directory
    file:
      path: /app
      state: directory

  - name: download cart application
    get_url:
      url: https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/cart-{{ app_version }}.zip
      dest: /tmp/cart.zip

  - name: unzip cart
    unarchive:
      src: /tmp/cart.zip
      dest: /app
      remote_src: yes

  - name: install dependencies
    community.general.npm:
      path: /app

  - name: copy cart service
    template:
      src: cart.service.j2
      dest: /etc/systemd/system/cart.service

  - name: daemon reload
    systemd_service:
      daemon_reload: true

  - name: start and enable cart
    service:
      name: cart
      state: restarted
      enabled: yes
```

| Task | What it does |
|---|---|
| `install nodejs → install dependencies` | Same as user; the download URL uses `{{ app_version }}` from `group_vars/cart.yaml` |
| `copy cart service` | Creates the systemd service |
| `daemon reload + start and enable` | Starts the service |

#### 7b. Push it (laptop)

```bash
git add cart.service.j2 07-cart.yaml
```

```bash
git commit -m "Add cart playbook"
```

```bash
git push
```

#### 7c. Pull the code (Ansible server)

```bash
git pull
```

#### 7d. Create the cart server and DNS record

```bash
ansible-playbook roboshop.yaml -e "instances=['cart']"
```

**Expected:** `roboshop-cart` in the EC2 console and the record `cart.abhignadevops.store` ✅.

#### 7e. Check Ansible can reach it

```bash
ansible cart -m ping
```

**Expected:** `"ping": "pong"` ✅. (Just created? Wait a minute for the server to boot, then try again.)

#### 7f. Configure it

```bash
ansible-playbook 07-cart.yaml
```

**Expected:** the recap shows `failed=0` ✅.

#### 7g. Check the service is running

```bash
ansible cart -b -m command -a "systemctl is-active cart"
```

**Expected:** `active` ✅. If not: `ansible cart -b -m command -a "journalctl -u cart -n 20"` shows why.

### Server 8: Shipping

**shipping** is the Java service that works out the shipping cost. It reads cities from MySQL.

#### 8a. Write the code (laptop)

```bash
vim shipping.service.j2
```

```ini
[Unit]
Description=Shipping Service
After=network-online.target

[Service]
User=roboshop
Environment=CART_ENDPOINT={{ cart_host }}:8080
Environment=DB_HOST={{ mysql_host }}
Environment=DB_USER={{ shipping_db_user }}
Environment=DB_PASSWORD={{ shipping_db_password }}
ExecStart=/bin/java -jar /app/shipping.jar
SyslogIdentifier=shipping
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`shipping.service.j2` gives the app the cart address and the MySQL login.

```bash
vim 08-shipping.yaml
```

```yaml
- name: configure shipping server
  hosts: shipping
  become: yes
  tasks:
  - name: enable maven 3.9
    command: dnf module enable maven:3.9 -y

  - name: install maven and java25
    dnf:
      name:
      - maven
      - java-25-openjdk-devel
      - maven-openjdk25 # installing this so that we no need to set JAVA_HOME again
      - mysql
      state: present

  # - name: find java25 path
  #   shell: readlink -f $(ls /usr/lib/jvm/java-25-openjdk*/bin/java | head -1)
  #   register: java25

  # - name: print the output
  #   debug:
  #     msg: "{{ java25 }}"

  # - name: make java25 as default
  #   command: alternatives --set java {{ java25.stdout }}

  # - name: set JAVA_HOME
  #   copy:
  #     content: "export JAVA_HOME={{ java25.stdout | dirname | dirname  }}"
  #     dest: /etc/profile.d/java.sh

  - name: create roboshop system user
    user:
      name: roboshop
      create_home: false
      home: /app
      system: true
      shell: /sbin/nologin
      comment: "roboshop system user"

  - name: remove existing app directory
    file:
      path: /app
      state: absent

  - name: remove shipping.zip
    file:
      path: /tmp/shipping.zip
      state: absent

  - name: create app directory
    file:
      path: /app
      state: directory

  - name: download shipping application
    get_url:
      url: https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/shipping-v4.zip
      dest: /tmp/shipping.zip

  - name: unzip shipping
    unarchive:
      src: /tmp/shipping.zip
      dest: /app
      remote_src: yes

  - name: mvn version without JAVA_HOME
    command: mvn --version
    args:
      chdir: /app
    register: mvn_default

  - name: print mvn version without JAVA_HOME
    debug:
      msg: "{{ mvn_default.stdout_lines }}"

  - name: build shipping
    command: mvn clean package
    args:
     chdir: /app
    # environment:
    #   JAVA_HOME: "{{ java25.stdout | dirname | dirname  }}"

  - name: move shipping jar file
    command: mv target/shipping.jar shipping.jar
    args:
     chdir: /app

  - name: check cities data exist or not
    command: mysql -h "{{ mysql_host }}" -u root -p{{ mysql_root_password }} -N -B -e "SELECT COUNT(*) FROM cities.cities;"
    register: cities_count
    ignore_errors: true
    tags:
    - mysql

  - name: print cities_count
    debug:
      msg: "{{ cities_count }}"
    tags:
    - mysql

  - name: load cities data into mysql
    shell: mysql -h "{{ mysql_host }}" -u root -p{{ mysql_root_password }} < /app/db/master-data.sql
    when: cities_count.rc != 0 or cities_count.stdout | int == 0
    tags:
    - mysql

  - name: create app user in mysql
    shell: mysql -h "{{ mysql_host }}" -u root -p{{ mysql_root_password }} < /app/db/app-user.sql
    tags:
    - mysql

  - name: copy shipping service
    template:
      src: shipping.service.j2
      dest: /etc/systemd/system/shipping.service

  - name: daemon reload
    systemd_service:
      daemon_reload: true

  - name: start and enable shipping
    service:
      name: shipping
      state: restarted
      enabled: yes
```

| Task | What it does |
|---|---|
| `enable maven 3.9 + install` | Installs Maven, Java 25, `maven-openjdk25` (makes Maven use Java 25) and the MySQL client |
| `create user → unzip shipping` | App user and code in `/app` |
| `mvn version / build shipping` | Shows the Maven/Java version, then builds the app with `mvn clean package` |
| `move shipping jar file` | `target/shipping.jar` → `/app/shipping.jar` |
| `check cities / load cities data` | Loads the cities into MySQL **only if** they are not there |
| `create app user in mysql` | Creates the `shipping` DB user |
| `copy shipping service → start` | Creates the systemd service and starts it |

#### 8b. Push it (laptop)

```bash
git add shipping.service.j2 08-shipping.yaml
```

```bash
git commit -m "Add shipping playbook"
```

```bash
git push
```

#### 8c. Pull the code (Ansible server)

```bash
git pull
```

#### 8d. Create the shipping server and DNS record

```bash
ansible-playbook roboshop.yaml -e "instances=['shipping']"
```

**Expected:** `roboshop-shipping` in the EC2 console and the record `shipping.abhignadevops.store` ✅.

#### 8e. Check Ansible can reach it

```bash
ansible shipping -m ping
```

**Expected:** `"ping": "pong"` ✅. (Just created? Wait a minute for the server to boot, then try again.)

#### 8f. Configure it

```bash
ansible-playbook 08-shipping.yaml
```

**Expected:** the recap shows `failed=0` ✅.

#### 8g. Check the service is running

```bash
ansible shipping -b -m command -a "systemctl is-active shipping"
```

**Expected:** `active` ✅. If not: `ansible shipping -b -m command -a "journalctl -u shipping -n 20"` shows why.

### Server 9: Payment

**payment** is the Python service for checkout. It talks to cart, user and RabbitMQ.

#### 9a. Write the code (laptop)

```bash
vim payment.service.j2
```

```ini
[Unit]
Description=Payment Service
After=network-online.target

[Service]
User=roboshop
WorkingDirectory=/app
Environment=CART_HOST={{ cart_host }}
Environment=CART_PORT=8080
Environment=USER_HOST={{ user_host }}
Environment=USER_PORT=8080
Environment=AMQP_HOST={{ rabbitmq_host }}
Environment=AMQP_USER={{ rabbitmq_user }}
Environment=AMQP_PASS={{ rabbitmq_password }}
ExecStart=/app/.venv/bin/gunicorn --config gunicorn.conf.py --no-control-socket payment:app
SyslogIdentifier=payment
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

`payment.service.j2` gives the app the cart, user and RabbitMQ addresses.

```bash
vim 09-payment.yaml
```

```yaml
- name: configure payment server
  hosts: payment
  become: yes
  tasks:
  - name: install python3.14
    dnf:
      name:
      - python3.14
      - python3.14-pip
      state: present

  - name: create roboshop system user
    user:
      name: roboshop
      create_home: false
      home: /app
      system: true
      shell: /sbin/nologin
      comment: "roboshop system user"

  - name: remove existing app directory
    file:
      path: /app
      state: absent

  - name: remove payment.zip
    file:
      path: /tmp/payment.zip
      state: absent

  - name: create app directory
    file:
      path: /app
      state: directory

  - name: download payment application
    get_url:
      url: https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/payment-v4.zip
      dest: /tmp/payment.zip

  - name: unzip payment
    unarchive:
      src: /tmp/payment.zip
      dest: /app
      remote_src: yes

  - name: install dependencies in virtual environment
    ansible.builtin.pip:
      requirements: /app/requirements.txt
      virtualenv: /app/.venv
      virtualenv_command: python3.14 -m venv

  - name: copy payment service
    template:
      src: payment.service.j2
      dest: /etc/systemd/system/payment.service

  - name: daemon reload
    systemd_service:
      daemon_reload: true

  - name: start and enable payment
    service:
      name: payment
      state: restarted
      enabled: yes
```

| Task | What it does |
|---|---|
| `install python3.14` | Installs Python 3.14 + pip |
| `create user → unzip payment` | App user and code in `/app` |
| `install dependencies in virtual environment` | Installs the Python libraries into `/app/.venv` |
| `copy payment service → start` | Creates the systemd service and starts it |

#### 9b. Push it (laptop)

```bash
git add payment.service.j2 09-payment.yaml
```

```bash
git commit -m "Add payment playbook"
```

```bash
git push
```

#### 9c. Pull the code (Ansible server)

```bash
git pull
```

#### 9d. Create the payment server and DNS record

```bash
ansible-playbook roboshop.yaml -e "instances=['payment']"
```

**Expected:** `roboshop-payment` in the EC2 console and the record `payment.abhignadevops.store` ✅.

#### 9e. Check Ansible can reach it

```bash
ansible payment -m ping
```

**Expected:** `"ping": "pong"` ✅. (Just created? Wait a minute for the server to boot, then try again.)

#### 9f. Configure it

```bash
ansible-playbook 09-payment.yaml
```

**Expected:** the recap shows `failed=0` ✅.

#### 9g. Check the service is running

```bash
ansible payment -b -m command -a "systemctl is-active payment"
```

**Expected:** `active` ✅. If not: `ansible payment -b -m command -a "journalctl -u payment -n 20"` shows why.

### Server 10: Frontend

**frontend** is the nginx web server users open in the browser.

#### 10a. Write the code (laptop)

```bash
vim roboshop.conf.j2
```

```ini
server_tokens off;

# nginx gives every request a unique id ($request_id). It is sent to the
# services and back to the browser, so one click can be traced on all servers
add_header X-Request-Id $request_id always;
add_header X-Content-Type-Options nosniff always;
add_header X-Frame-Options SAMEORIGIN always;
add_header Referrer-Policy strict-origin-when-cross-origin always;

gzip on;
gzip_types text/css application/javascript application/json image/svg+xml;
gzip_min_length 1024;

# common settings for all backend calls
proxy_http_version 1.1;
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Request-Id      $request_id;
# the services send the id back too, nginx already adds it, so hide theirs
proxy_hide_header X-Request-Id;
proxy_connect_timeout 5s;
proxy_read_timeout    30s;   # 504 if a service takes longer than this

# the trailing / removes the /api/<service> part:
# /api/catalogue/products -> http://<CATALOGUE-IP>:8080/products
location /api/catalogue/ { proxy_pass http://{{ catalogue_host }}:8080/; }
location /api/user/      { proxy_pass http://{{ user_host }}:8080/; }
location /api/cart/      { proxy_pass http://{{ cart_host }}:8080/; }
location /api/shipping/  { proxy_pass http://{{ shipping_host }}:8080/; }
location /api/payment/   { proxy_pass http://{{ payment_host }}:8080/; }

# when nginx itself cannot reach a service, answer in JSON like the services do.
# errors a service returns on its own (its 404, 503...) pass through untouched
error_page 502 = @bad_gateway;
error_page 504 = @gateway_timeout;
location @bad_gateway {
    default_type application/json;
    return 502 '{"message":"service not reachable (502 Bad Gateway from nginx)"}';
}
location @gateway_timeout {
    default_type application/json;
    return 504 '{"message":"service did not answer in time (504 Gateway Timeout from nginx)"}';
}

# nginx's own connection counters
location = /health {
    stub_status on;
    access_log off;
}

# product images, missing ones fall back to the placeholder
location /images/ {
    expires 1h;
    try_files $uri /images/placeholder.jpg;
}

# file names in /assets contain a hash of their content, so they can be cached forever
location /assets/ {
    expires max;
    try_files $uri =404;
}

# React handles all other paths (/product/RMC, /cart ...) in the browser,
# so every unknown path returns index.html
location / {
    expires -1;
    try_files $uri $uri/ /index.html;
}
```

`roboshop.conf.j2` is the nginx config: each `location /api/<service>/` forwards to that backend, and every other path returns the website.

```bash
vim 10-frontend.yaml
```

```yaml
- name: configure frontend server
  hosts: frontend
  become: yes
  tasks:
  - name: enable nginx 1.26
    command: dnf module enable nginx:1.26 -y

  - name: install nginx
    dnf:
      name: nginx
      state: present

  - name: start and enable nginx
    service:
      name: nginx
      state: started
      enabled: yes

  - name: Find all contents inside the directory
    ansible.builtin.find:
      paths: /usr/share/nginx/html/
      recurse: true
      hidden: true
      file_type: any
    register: directory_contents

  - name: print directory content
    debug:
      msg: "{{ directory_contents }}"

  - name: Delete the discovered contents
    ansible.builtin.file:
      path: "{{ item.path }}"
      state: absent
    loop: "{{ directory_contents.files }}"

  - name: download frontend application
    get_url:
      url: https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/frontend-v4.zip
      dest: /tmp/frontend.zip

  - name: unzip frontend
    unarchive:
      src: /tmp/frontend.zip
      dest: /usr/share/nginx/html/
      remote_src: yes

  - name: copy roboshop config
    template:
      src: roboshop.conf.j2
      dest: /etc/nginx/default.d/roboshop.conf

  - name: restart frontend
    service:
      name: nginx
      state: restarted
```

| Task | What it does |
|---|---|
| `enable nginx 1.26 + install + start` | Installs and starts nginx |
| `find / delete the discovered contents` | Removes the default nginx page |
| `download + unzip frontend` | Puts the RoboShop website in `/usr/share/nginx/html/` |
| `copy roboshop config` | The reverse proxy config: `/api/<service>/` → that service on port 8080 |
| `restart frontend` | Restarts nginx with the new config |

#### 10b. Push it (laptop)

```bash
git add roboshop.conf.j2 10-frontend.yaml
```

```bash
git commit -m "Add frontend playbook"
```

```bash
git push
```

#### 10c. Pull the code (Ansible server)

```bash
git pull
```

#### 10d. Create the frontend server and DNS record

```bash
ansible-playbook roboshop.yaml -e "instances=['frontend']"
```

**Expected:** `roboshop-frontend` in the EC2 console and the record `frontend.abhignadevops.store` ✅.

#### 10e. Check Ansible can reach it

```bash
ansible frontend -m ping
```

**Expected:** `"ping": "pong"` ✅. (Just created? Wait a minute for the server to boot, then try again.)

#### 10f. Configure it

```bash
ansible-playbook 10-frontend.yaml
```

**Expected:** the recap shows `failed=0` ✅.

#### 10g. Check the service is running

```bash
ansible frontend -b -m command -a "systemctl is-active nginx"
```

**Expected:** `active` ✅. If not: `ansible frontend -b -m command -a "journalctl -u nginx -n 20"` shows why.

---

## Part 5: Open the shop

### Step 22: Open the website

Open `http://abhignadevops.store` in the browser.

**Expected:** the RoboShop home page with products. Register, log in, add an item to the cart and check out ✅.

| Page shows | Check |
|---|---|
| Site doesn't open | `nslookup abhignadevops.store` must give frontend's public IP; `roboshop-frontend` SG must allow port 80 |
| No products | catalogue (Server 5) and MongoDB (Server 1) |
| `502` / `service not reachable` | That backend service is down: Step g of that server |

---

## Part 6: Clean up

### Step 23: Delete the RoboShop servers and records (Ansible server)

```bash
ansible-playbook roboshop.yaml -e "action=delete"
```

**Expected:** all 10 servers terminating and their records gone ✅.

> ⚠️ Deleting one server (`-e "action=delete instances=['mongodb']"`) also deletes the `abhignadevops.store` record. Run `-e "instances=['frontend']"` again to bring it back.

### Step 24: Delete the Ansible server (laptop)

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
| `couldn't resolve module/action 'amazon.aws.ec2_instance'` | Collection not installed | Step 17 |
| `couldn't resolve module/action 'systemd_service'` | Older Ansible version | Change `systemd_service:` to `systemd:` in the playbook |
| `InvalidGroup.NotFound` | A `roboshop-<name>` security group is missing | Create it in `us-east-1` |
| `UNREACHABLE … timed out` | No SSH rule from the Ansible server | Step 6 |
| `UNREACHABLE … Permission denied` | Wrong SSH password | Step 20 |
| `git pull` says `local changes … would be overwritten` | You edited a file on the Ansible server that also changed on GitHub | `git stash`, `git pull`, `git stash pop` |

---

## What I learned

| Part | What I learned |
|---|---|
| **AWS setup** | An **IAM role** (inside an instance profile) gives the Ansible server AWS access with no keys; a **self-referencing SG rule** lets servers in the same group SSH to each other |
| **Project setup** | `ansible.cfg`, `inventory.ini` and `group_vars/` are shared by every playbook; servers are listed by **DNS name** |
| **One server at a time** | `-e "instances=['<name>']"` creates one server; each playbook installs, configures and starts one server; **templates** fill in server names |
| **Order** | Databases → backend → frontend, because of dependencies |
| **Clean up** | `-e "action=delete"` reuses the same playbook to delete everything |

```
 IAM role + SG rule ─► Ansible server ─► setup files ─► (write → push → pull → create → configure → check) × 10 ─► open shop ─► delete
```

**Full explanation of every file:** [RoboShop v1 with Ansible: every file explained](https://github.com/Abhignadumpala/devops-notes/blob/main/day-22-ansible-shell-vs-ansible-aws-ec2-route53/roboshop-v1/README.md)
