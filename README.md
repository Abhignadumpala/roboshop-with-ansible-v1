# RoboShop with Ansible v1

RoboShop e-commerce app (10 servers) set up with Ansible playbooks.

## The 3 tiers: 10 servers in 3 groups

**Definition:** RoboShop is a **3-tier app**: a **frontend** the user sees, **backend** services that run the shop's logic, and **data** servers that store everything. Each tier only talks to the tier next to it, so each part can be changed, scaled or fixed on its own.

| Tier | Servers | Job |
|---|---|---|
| **Frontend** (1) | frontend | Shows the website and passes API calls to the backend |
| **Backend** (5) | catalogue · user · cart · shipping · payment | Business logic: products, login, cart, shipping cost, checkout |
| **Data** (4) | MongoDB · Redis · MySQL · RabbitMQ | Store the data or queue messages |

```
 Browser ──► frontend (nginx) ──► backend services (port 8080) ──► data servers
   user         tier 1               tier 2                          tier 3
```

## What the frontend server is for

**Definition:** the **frontend** is an **nginx** web server, the only server users open in the browser. It does two jobs: it serves the website files (HTML, JavaScript, images), and it works as a **reverse proxy**: every `/api/...` request is forwarded to the right backend service.

| Request from the browser | nginx sends it to |
|---|---|
| `/` (pages, images, JS) | Its own website files in `/usr/share/nginx/html` |
| `/api/catalogue/...` | catalogue:8080 |
| `/api/user/...` | user:8080 |
| `/api/cart/...` | cart:8080 |
| `/api/shipping/...` | shipping:8080 |
| `/api/payment/...` | payment:8080 |

**Why a reverse proxy?** The user only ever talks to **one address** (`abhignadevops.store`). The backend servers stay private (no public access), and we can move or replace a service by changing nginx's config, without the user noticing.

> **Interview one-liner:** the frontend is nginx: it serves the static website and reverse-proxies each `/api/<service>/` path to that backend on port 8080, so users see one domain and the backends stay private.

## What the 5 backend servers are for

**Definition:** the **backend** servers run the shop's **business logic**. Each one is a small **microservice** with one job, its own code and its own server, listening on port **8080**. The frontend calls them, and they read and write the data servers.

| Server | Language | What it does | Talks to |
|---|---|---|---|
| **catalogue** | Node.js | Lists products and product details | MongoDB |
| **user** | Node.js | Registration and login | MongoDB (users), Redis (sessions) |
| **cart** | Node.js | Add/remove items in the cart | Redis, catalogue (product details) |
| **shipping** | Java (built with Maven) | Works out the shipping cost by city | MySQL, cart |
| **payment** | Python | Checkout: checks the cart and user, then places the order | cart, user, RabbitMQ |

**Why 5 small services, not one big app?** Each **microservice** can be deployed, restarted or scaled on its own, and a bug in one (say shipping) doesn't take down the others. Each team can also pick the best language for the job: that's why RoboShop mixes Node.js, Java and Python.

**Same setup pattern for all 5:** install the language runtime → create a user to run the app → download the code into `/app` → install dependencies → create a systemd service → start it. Only the runtime and a few extra steps differ.

> **Interview one-liner:** RoboShop has 5 backend microservices on port 8080: catalogue, user and cart in Node.js, shipping in Java and payment in Python. Each has one job and its own server, so it can be deployed and scaled on its own.

## What the 4 data servers are for

**Definition:** RoboShop has **4 data servers** that hold the shop's data or pass messages between services. The app servers (catalogue, user, cart, shipping, payment) run the code; these 4 store what the code reads and writes. Each kind of data goes in the store that suits it best.

| Server | What it is | What RoboShop keeps there | Used by |
|---|---|---|---|
| **MongoDB** | NoSQL database: stores data as JSON-like documents | **Product list**, **user accounts** and order history | catalogue, user |
| **Redis** | In-memory cache: key-value data kept in RAM, very fast | **Cart items** and **login sessions** | cart, user |
| **MySQL** | Relational (SQL) database: data in tables with rows and columns | **Cities and shipping data**, used to work out the shipping cost | shipping |
| **RabbitMQ** | Message queue: holds messages until another service picks them up | **Orders** sent after payment | payment sends, dispatch reads |

**Why 4 different ones?** Products have flexible fields → a document database (MongoDB). The cart and sessions change all the time and must be fast → memory (Redis). Cities are fixed, table-shaped data → SQL (MySQL). Payment shouldn't wait for the order to be processed → it drops a message in a queue and moves on (RabbitMQ).

```
 frontend (nginx)
    ├── catalogue ──► MongoDB   (products)
    ├── user ───────► MongoDB   (users, orders) + Redis (sessions)
    ├── cart ───────► Redis     (cart items)
    ├── shipping ───► MySQL     (cities)
    └── payment ────► RabbitMQ  (orders) ──► dispatch
```

> **Interview one-liner:** RoboShop has 4 data servers: MongoDB holds products and users, Redis caches carts and sessions in memory, MySQL holds the shipping cities in tables, and RabbitMQ queues orders from payment to dispatch, so each kind of data uses the store that suits it best.

---

## How to run this lab

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

### Part 1: AWS setup (on your laptop)

#### Step 1: Create an IAM role for the Ansible server

The Ansible server calls AWS to create EC2 servers and DNS records, so it needs permission. An **IAM role** gives it temporary keys automatically, so there are no access keys on disk.

```bash
aws iam create-role --role-name ansible-server-role --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
```

**What it does:** creates the role and says **EC2 servers** are allowed to use it.

```bash
aws iam attach-role-policy --role-name ansible-server-role --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
```

```bash
aws iam attach-role-policy --role-name ansible-server-role --policy-arn arn:aws:iam::aws:policy/AmazonRoute53FullAccess
```

**What it does:** gives the role only what it needs: EC2 and Route 53. (Not `AdministratorAccess`: least privilege.)

```bash
aws iam create-instance-profile --instance-profile-name ansible-server-role
```

```bash
aws iam add-role-to-instance-profile --instance-profile-name ansible-server-role --role-name ansible-server-role
```

**What it does:** an EC2 server can't use a role directly; it uses an **instance profile**, a box that holds the role.

#### Step 2: Let the servers SSH to each other

`roboshop-common` allows SSH only from our home IP. The Ansible server is **inside AWS**, so it needs its own rule to reach the RoboShop servers.

```bash
aws ec2 authorize-security-group-ingress --group-name roboshop-common --protocol tcp --port 22 --source-group roboshop-common
```

**What it does:** any server in `roboshop-common` can now SSH to any other server in `roboshop-common`. The Ansible server and all 10 RoboShop servers are in it.

**Expected:** `"Return": true` ✅. (`InvalidPermission.Duplicate` means the rule is already there, which is fine.)

#### Step 3: Launch the Ansible server

```bash
aws ec2 run-instances --image-id ami-0055977a6e828a4f4 --instance-type t3.micro --security-groups roboshop-common --iam-instance-profile Name=ansible-server-role --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ansible-server}]'
```

**What it does:** creates the RHEL 9 Ansible server with the role from Step 1 and the security group from Step 2.

| Error | Fix |
|---|---|
| `Invalid IAM Instance Profile name` | A new profile takes a few seconds to be ready. Wait 10 seconds and run it again |

```bash
aws ec2 describe-instances --filters "Name=tag:Name,Values=ansible-server" "Name=instance-state-name,Values=running" --query 'Reservations[].Instances[].PublicIpAddress' --output text
```

**Expected:** the server's public IP ✅.

---

### Part 2: Prepare the Ansible server

#### Step 4: Log in

```bash
ssh ec2-user@<ansible-server-public-ip>
```

Type `yes` the first time, then the SSH password (nothing shows while you type).

#### Step 5: Install Ansible and the collections

```bash
sudo dnf install ansible-core git -y
```

```bash
ansible-galaxy collection install amazon.aws community.general community.rabbitmq
```

**What it does:** `ansible-core` has only the built-in modules. The **collections** add the ones we need: `amazon.aws` (EC2, Route 53), `community.general` (npm) and `community.rabbitmq` (RabbitMQ user).

#### Step 6: Install boto3 and check AWS access

```bash
sudo pip3.9 install botocore boto3
```

**What it does:** the `amazon.aws` modules call AWS through **boto3**, so it must be installed for Python 3.9, the Python Ansible uses on RHEL 9.

```bash
aws sts get-caller-identity
```

**Expected:** `Arn` shows `assumed-role/ansible-server-role/…` ✅. The server has AWS access with no keys.

#### Step 7: Get the code

```bash
git clone https://github.com/Abhignadumpala/roboshop-with-ansible-v1.git
```

```bash
cd roboshop-with-ansible-v1
```

```bash
vim group_vars/all.yaml
```

Set `ansible_password:` to the SSH password of the RoboShop servers.

> ⚠️ Never push the real password to GitHub. Keep `<ssh-password>` in the repo and fill it in only on the server.

---

### Part 3: Run RoboShop

#### Step 8: Create the 10 servers and DNS records

```bash
ansible-playbook roboshop.yaml
```

**Expected:** 10 servers `roboshop-mongodb` … `roboshop-frontend` in the EC2 console, and a Route 53 record for each one ✅.

#### Step 9: Check Ansible can reach every server

```bash
ansible all -m ping
```

**Expected:** every server shows `"ping": "pong"` ✅.

| Error | Cause | Fix |
|---|---|---|
| `UNREACHABLE … timed out` | No SSH rule from the Ansible server | Step 2 |
| `UNREACHABLE … Permission denied` | Wrong password | Fix `group_vars/all.yaml` |
| `Could not resolve hostname` | Records not created yet | Run `roboshop.yaml` again |

#### Step 10: Run the playbooks in order

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

#### Step 11: Open the shop

Open `http://abhignadevops.store` in the browser.

**Expected:** the RoboShop home page with products. Add an item to the cart, then check out ✅.

---

### Part 4: Clean up

#### Step 12: Delete the RoboShop servers and records (on the Ansible server)

```bash
ansible-playbook roboshop.yaml -e "action=delete"
```

**Expected:** all 10 servers terminating and their records gone ✅.

#### Step 13: Delete the Ansible server (on your laptop)

```bash
exit
```

```bash
aws ec2 terminate-instances --instance-ids $(aws ec2 describe-instances --filters "Name=tag:Name,Values=ansible-server" "Name=instance-state-name,Values=running,stopped" --query 'Reservations[].Instances[].InstanceId' --output text)
```

**Expected:** `"Name": "shutting-down"` ✅. The IAM role and the SG rule cost nothing, so we keep them for the next lab.

**Full explanation of every file:** [RoboShop v1 with Ansible: every file explained](https://github.com/Abhignadumpala/devops-notes/blob/main/day-22-ansible-shell-vs-ansible-aws-ec2-route53/roboshop-v1/README.md)
