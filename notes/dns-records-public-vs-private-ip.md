# DNS records: public vs private IP

**Definition:** a **DNS A record** maps a name (like `mongodb.abhignadevops.store`) to an IP address. In RoboShop, every server gets a record pointing to its **private IP**, and the frontend gets one extra record pointing to its **public IP**.

| Part | What it covers |
|---|---|
| [Part 1: Concepts](#part-1-concepts) | Private vs public IP, the two frontend records, who uses which |
| [Part 2: Lab](#part-2-lab-fix-the-website-after-a-stop-and-start) | Fix the website after stopping and starting the servers |
| [Interview one-liners](#interview-one-liners) | Short answers to remember |

---

## Part 1: Concepts

### Private IP vs public IP

| IP type | Definition | Reachable from | After a stop → start |
|---|---|---|---|
| **Private IP** (`172.31.x.x`) | The server's address **inside the VPC** | Only other servers in the VPC | ✅ **Stays the same** (until the server is terminated) |
| **Public IP** (auto-assigned) | An address **borrowed from AWS's pool** for internet access | The internet | ❌ **Changes**: released on stop, a new one is given on start |

### The records `roboshop.yaml` creates

| Record | Points to | Created by task | Used by |
|---|---|---|---|
| `mongodb.abhignadevops.store`, `cart...`, … (all 10 servers) | **Private IP** | `create R53 record` (loop over all servers) | **Ansible** (inventory) + **service-to-service** calls |
| `frontend.abhignadevops.store` | **Private IP** | Same loop | **Ansible** SSH to frontend |
| `abhignadevops.store` (apex / root domain) | Frontend **public IP** | `create R53 record for frontend` | **Users' browsers** |

So the frontend has **two records**, one for each kind of traffic:

```
 Internet users ──► abhignadevops.store ─────────► public IP ──► frontend :80
                                                                    ▲
 Ansible server ──► frontend.abhignadevops.store ─► private IP ─SSH 22┘
   (same VPC)
```

### Why Ansible uses the private IP

| Reason | Explanation |
|---|---|
| **Same network** | The Ansible server and all RoboShop servers are in the same VPC, so traffic stays inside AWS |
| **Doesn't change** | Private IPs survive a stop → start, so the inventory keeps working |
| **More secure** | SSH is allowed only from the `roboshop-common` security group (and my home IP), not from the whole internet |

The other 9 servers only get a private record: nobody outside needs to reach them. The frontend is the **only entry point** from the internet.

### Who is allowed to connect (security groups)

| Security group | Attached to | Inbound rule | Lets in |
|---|---|---|---|
| `roboshop-common` | All 10 servers + Ansible server | SSH 22 from `roboshop-common` (itself) | **Ansible server** → every server |
| | | SSH 22 from `<my-home-ip>/32` | **My laptop** → every server |
| `roboshop-frontend` | Frontend only | HTTP 80 from `0.0.0.0/0` | **Anyone** → the website |
| `roboshop-<server>` | That server | Its app port from the servers that call it | e.g. MongoDB 27017 from catalogue and user |

> An instance gets the **combined rules** of all its security groups. Using a security group as the source allows any server in that group, whatever its IP.

### Why the website breaks after a stop → start

A DNS record is a **static value**. `roboshop.yaml` reads the IPs **when it runs** and writes them to Route 53. Nothing updates the record later.

```
 First run:   frontend public IP = A ──► Route 53: abhignadevops.store = A
 Stop/start:  AWS gives frontend a NEW public IP = B
              Route 53 still says A ❌   → website doesn't open
              Private records unchanged ✅ → Ansible + services still work
```

---

## Part 2: Lab: fix the website after a stop and start

In my case, the servers were stopped overnight and started again the next day. If you terminated them instead, every IP is new: run `roboshop.yaml` for all servers (same command without `-e`).

### Step 1: Rerun the create playbook for frontend (Ansible server)

```bash
ansible-playbook roboshop.yaml -e "instances=['frontend']"
```

**What it does:** `ec2_instance` finds that `roboshop-frontend` already exists, so it creates nothing and just returns the current details (**idempotency**). The frontend record task writes the new public IP, and `overwrite: true` replaces the old one.
**Expected:** `failed=0`, `changed=1`.

> Running it for **all** servers (`ansible-playbook roboshop.yaml`) is also safe: still `changed=1`, because only the frontend's public record is different. No duplicate servers are created.

### Step 2: Check the record

```bash
dig +short abhignadevops.store
```

**What it does:** shows the IP the domain points to now.
**Expected:** the frontend's **current** public IP (see the EC2 console).

### Step 3: Open the website

```
http://abhignadevops.store
```

**Expected:** the RoboShop home page with products. Still the old page? Your browser cached the old IP: try an incognito window.

### How production avoids this

| Option | How it helps |
|---|---|
| **Elastic IP** | A fixed public IP attached to the server; it never changes |
| **Load balancer (ALB)** | DNS points to the ALB's name, which stays the same even when servers are replaced |

---

## Interview one-liners

- *"A stopped EC2 instance keeps its private IP but gets a new public IP when it starts, so any DNS record holding the old public IP goes stale."*
- *"Internal traffic (Ansible SSH, service to service) uses private DNS records; only the public entry point gets a public record, which in production sits behind an Elastic IP or a load balancer."*
- *"A DNS record is a static value. Rerunning idempotent automation updates only what changed."*
- *"Security groups are allow-only and stateful; an instance gets the union of all its SGs, and an SG can be used as the source of a rule."*
