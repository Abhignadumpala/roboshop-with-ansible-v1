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

## Extra notes

| Topic | Link |
|---|---|
| Where the data comes from and where it's stored (master data vs live data, mongosh / mysql loading) | [notes/where-the-data-comes-from.md](notes/where-the-data-comes-from.md) |
| Logs vs metrics vs traces (observability), log levels, checking a service with `systemctl` / `journalctl` / `ss` / `curl` | [notes/logs-metrics-observability.md](notes/logs-metrics-observability.md) |
| DNS records: public vs private IP, the two frontend records, security groups, fixing the website after a stop/start | [notes/dns-records-public-vs-private-ip.md](notes/dns-records-public-vs-private-ip.md) |

---

## How to run this lab

The full step-by-step guide is in **[LAB.md](LAB.md)**: create the IAM role and the Ansible server, install Ansible, create the 10 servers, run the playbooks in order, open the shop, and clean up.

```
 Laptop ──► IAM role + SG rule + Ansible server ──► roboshop.yaml (10 servers + DNS) ──► 01…10 playbooks ──► http://<your-domain>
```
