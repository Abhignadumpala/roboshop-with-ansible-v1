# RoboShop with Ansible v1

RoboShop e-commerce app (10 servers) set up with Ansible playbooks.

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
