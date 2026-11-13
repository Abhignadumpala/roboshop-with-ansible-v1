# Logs, metrics and observability

**Definition:** **observability** means being able to tell what's happening inside a running system from the data it gives out. That data comes in three parts: **logs**, **metrics** and **traces**.

| Part | What it covers |
|---|---|
| [Part 1: Concepts](#part-1-concepts) | Logs vs metrics vs traces, log levels |
| [Part 2: Lab](#part-2-lab-check-the-user-service) | Check the RoboShop `user` service with `systemctl`, `journalctl`, `ss` and `curl` |
| [Interview one-liners](#interview-one-liners) | Short answers to remember |

---

## Part 1: Concepts

### The three parts of observability

| Part | Definition | Answers | Example |
|---|---|---|---|
| **Logs** | Timestamped **text events** written by an app or system | **What happened, and why?** | `MongoDB connected`, `Error: connection refused` |
| **Metrics** | **Numbers** measured over time | **How much / how fast** right now? | CPU 45%, memory 60 MB, 120 requests/sec |
| **Traces** | The **path one request takes** across services | **Where** is a request slow or failing? | frontend → cart → catalogue |

```
            ┌────────────── Observability ──────────────┐
            │                                            │
          Logs                  Metrics                 Traces
     (what happened)          (how much)       (path of one request)
   journalctl, ELK,       top, Prometheus,       Jaeger, X-Ray,
   CloudWatch Logs        Grafana, CloudWatch    OpenTelemetry
```

### Logs vs metrics

| | **Logs** | **Metrics** |
|---|---|---|
| **What** | Text lines | Numbers |
| **Example** | `MongoNetworkError: connection refused` | Memory 43.9 MB |
| **Used to** | Find the **cause** of a problem | **Spot** a problem (alerts, dashboards) |
| **Tools** | `journalctl`, `/var/log/*`, ELK, CloudWatch Logs | `top`, `free -m`, Prometheus, Grafana, CloudWatch Metrics |

> **Common mix-up:** "logs" doesn't mean only error logs. Logs record **every** event: normal ones and errors.

### Log levels

**Definition:** a **log level** tags how serious a log line is.

| Level | Meaning | Example |
|---|---|---|
| **DEBUG** | Details for developers | `Query took 3 ms` |
| **INFO** | Normal activity | `Started on port 8080` |
| **WARN** | Something to watch | `Redis slow to respond` |
| **ERROR** | Something broke | `MongoNetworkError: connection refused` |

### Where a service's logs come from

**Definition:** **`journalctl`** reads the **systemd journal**, the central log store on RHEL-family Linux. Every service that systemd starts writes its output there automatically.

```
 node /app/server.js ──prints──► stdout/stderr ──► systemd ──► journal ──► journalctl -u user
```

`journalctl -u user` shows two kinds of lines:

| Line comes from | Example |
|---|---|
| **systemd** (start/stop events) | `Started User Service`, `Main process exited, status=1/FAILURE` |
| **The app** (whatever it prints) | `MongoDB connected`, error stack traces |

The `SyslogIdentifier=user` line in the `.service` file is the **tag** on each line (`user[5034]: ...`).

---

## Part 2: Lab: check the user service

In my case, the `user` service is already deployed by `06-user.yaml`. If you're doing this for the first time, run that playbook first, then log in to the user server.

### Step 1: Switch to root

```bash
sudo su -
```

**What it does:** gives root access. As `ec2-user` you can't read other services' logs or see which process owns a port.
**Expected:** the prompt ends in `#`.

### Step 2: Is the service running?

```bash
systemctl status user
```

**What it does:** shows the service state, plus a few **metrics** (memory, CPU).
**Expected:** `Active: active (running)` and `Memory: 43.9M`. Press `q` to exit.

### Step 3: Read its logs

```bash
journalctl -u user -n 20 --no-pager
```

**What it does:** shows the last 20 **log** lines of the user service.
**Expected:** `Started User Service` plus the app's own lines, with no errors repeating.

### Step 4: Is it listening?

```bash
ss -lntp | grep 8080
```

**What it does:** shows whether the app is listening on its port (`-l` listening, `-n` numbers, `-t` TCP, `-p` process).
**Expected:** `LISTEN ... *:8080 ... node`

### Step 5: Does it answer?

```bash
curl http://localhost:8080/health
```

**What it does:** a **health check**, a quick "is it working?" test.
**Expected:** `{"app":"OK","mongo":true,"redis":true}`

### Useful `journalctl` options

| Command | Shows |
|---|---|
| `journalctl -u user` | All logs for the service |
| `journalctl -u user -n 20` | Last 20 lines |
| `journalctl -u user -f` | Follow live, like `tail -f` |
| `journalctl -u user --since "10 min ago"` | Recent lines only |
| `journalctl -u user -p err` | Errors only |
| `journalctl -b` | Everything since the last boot |

> `-u` = **unit** (service name). Uppercase `-U` = **until** (a time), so `journalctl -U user` fails with `Failed to parse timestamp`.

### Troubleshooting order

```
 systemctl status ──► journalctl -u ──► ss -lntp ──► curl /health
  (is it running?)     (why not?)      (listening?)   (answers?)
```

| You see | Meaning | Fix |
|---|---|---|
| `journalctl` says `-- No entries --` | Not root | `sudo su -` |
| `ss` Process column is empty | Not root | `sudo su -` |
| `active` but `curl` fails | App started but can't reach its DB | Check `journalctl -u user` for the connection error |

---

## Interview one-liners

- *"Logs are timestamped text events that tell you why something happened; metrics are numbers over time that tell you how much. Use metrics to spot a problem and logs to find the cause."*
- *"Observability has three parts: logs (what happened), metrics (how much) and traces (the path one request takes across services)."*
- *"`journalctl -u <service>` shows a service's stdout/stderr plus systemd's start/stop messages. It's the first place to look when a service fails."*
- *"`ss -lntp` lists the TCP ports a server is listening on, with the owning process."*
