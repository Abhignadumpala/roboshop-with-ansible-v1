# Where the data comes from and where it's stored

**Definition:** RoboShop's databases start **empty**. Some data is loaded **once at deployment** (starter data the app needs from day 1). The rest is written **by the apps at run time** when people use the shop.

## Two kinds of data

| Kind | Who puts it in | When | Example |
|---|---|---|---|
| **Master data** (starter data) | Ansible runs the developer's script | Once, during deployment | 11 products, the city list, a demo user |
| **Live data** | The apps | While people use the shop | New sign-ups, carts, orders |

## Where master data comes from

It's inside the **app zip** (the **artifact**). Ansible downloads and unzips it to `/app` on the app server:

```
 Developer writes the app + db scripts ─► zip (artifact) ─► GitHub artifacts/ ─► curl + unzip to /app
                                                                                  └── /app/db/*.js / *.sql
```

> In this lab, the trainer plays the developer. In companies, CI builds the artifact and stores it in Nexus / JFrog / S3, and the business team manages the real products.

## Which service loads what

| Service | Data file (in `/app/db/`) | Client installed | Sent to | Stored in | Loaded at deploy? |
|---|---|---|---|---|---|
| catalogue | `master-data.js` | `mongosh` | MongoDB :27017 | `catalogue.products` (11) + demo user in `users.users` | ✅ Yes |
| user | none | none | MongoDB + Redis | `users.users` (sign-ups), sessions in Redis | ❌ No (writes at run time) |
| cart | none | none | Redis | carts in memory | ❌ No |
| shipping | `master-data.sql`, `app-user.sql` | `mysql` | MySQL :3306 | `cities.cities` + MySQL user `shipping` | ✅ Yes |
| payment | none | none | RabbitMQ | order messages in a queue | ❌ No |

## How loading works (catalogue example)

The **app server** has the file and the client. The **DB server** stores the data on its disk.

```
 ┌──────── catalogue server ────────┐            ┌──────── mongodb server ────────┐
 │ /app/db/master-data.js           │   27017    │ catalogue.products (11)         │
 │ mongosh --host mongodb... < file │ ─────────► │ users.users (demo user)         │
 └──────────────────────────────────┘            │ disk: /var/lib/mongo            │
                                                 └─────────────────────────────────┘
 ┌──────── shipping server ─────────┐            ┌──────── mysql server ──────────┐
 │ /app/db/master-data.sql          │   3306     │ cities.cities                   │
 │ mysql -h mysql... < file         │ ─────────► │ user: shipping                  │
 └──────────────────────────────────┘            │ disk: /var/lib/mysql            │
                                                 └─────────────────────────────────┘
```

| Server | Repo / package | Why |
|---|---|---|
| mongodb | `mongo.repo` → `mongodb-org` (the **database server**) | Stores the data |
| catalogue | `mongo.repo` → `mongodb-mongosh` (only the **client**) | Sends the data to the mongodb server |
| shipping | `mysql` (only the **client**) | Sends the data to the mysql server |

## Load only once

Loading again **adds the data twice**. The playbook counts first and loads only if the count is 0:

| Service | Check command | Loaded means |
|---|---|---|
| catalogue | `mongosh --host mongodb.<domain> --quiet --eval 'db.getSiblingDB("catalogue").products.countDocuments()'` | `11` |
| shipping | `mysql -h mysql.<domain> -u root -p<password> -N -B -e "SELECT COUNT(*) FROM cities.cities;"` | more than `0` |

In the playbook: `register: product_count` → `when: product_count.stdout | int == 0`.

## How the apps find the databases (run time)

From the systemd service files (Jinja2 templates):

| Service | Variables |
|---|---|
| catalogue | `MONGO_URL=mongodb://{{ mongodb_host }}:27017/catalogue` |
| user | `MONGO_URL=…/users`, `REDIS_URL=redis://{{ redis_host }}:6379` |
| cart | `REDIS_HOST={{ redis_host }}`, `CATALOGUE_HOST={{ catalogue_host }}` |
| shipping | `DB_HOST={{ mysql_host }}`, `DB_USER`, `DB_PASSWORD` |
| payment | `AMQP_HOST={{ rabbitmq_host }}`, `AMQP_USER`, `AMQP_PASS` |

## Interview one-liner

*"The app comes as a versioned artifact with DB seed scripts inside. Ansible deploys it and runs the seed script once from the app server using the DB client (mongosh / mysql), with a count check first so it's idempotent. The data is stored on the DB server's disk. Everything else, like users, carts and orders, is written by the apps at run time."*
