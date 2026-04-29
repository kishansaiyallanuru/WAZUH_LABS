# Wazuh Lab – Restarting After Shutdown

This guide walks you through bringing your Wazuh lab back up after shutting down your system. No reinstallation needed — you just need to start the services again in the right order.

---

## Part 1: Wazuh Server (Kali Linux)

### Step 1 — Start Docker

```bash
sudo systemctl start docker
```

Docker must be running before anything else. All Wazuh components run as containers, so this is always the first step.

---

### Step 2 — Navigate to the Wazuh Directory

```bash
cd ~/wazuh-docker/single-node
```

This is where the `docker-compose.yml` file lives. You need to be in this directory to run the compose commands.

---

### Step 3 — Start the Wazuh Containers

```bash
sudo docker compose up -d
```

This starts all three Wazuh containers in the background — Manager, Indexer, and Dashboard.

---

### Step 4 — Verify Containers Are Running

```bash
sudo docker ps
```

You should see all three containers with `Up` status:

- `wazuh.manager`
- `wazuh.indexer`
- `wazuh.dashboard`

---

### Step 5 — Open the Dashboard

Open your browser and go to:

```
https://<YOUR_SERVER_IP>
```

Login credentials:

| Field | Value |
|---|---|
| Username | `admin` |
| Password | `SecretPassword` |

---

## Part 2: Wazuh Agent (Ubuntu VM)

### Step 6 — Start the Agent Service

```bash
sudo systemctl start wazuh-agent
```

This starts the agent on your Ubuntu machine and begins forwarding logs to the Wazuh Manager.

---

### Step 7 — Check Agent Status

```bash
sudo systemctl status wazuh-agent
```

You should see:

```
Active: active (running)
```

If it shows anything other than `active (running)`, check that the Manager containers are fully up before retrying.

---

## Part 3: Verify the Connection

### Step 8 — Confirm Agent Appears in Dashboard

In the Wazuh Dashboard, go to:

```
Wazuh → Agents
```

You should see `ubuntu-agent` listed with a green **Active** status. If it still shows as disconnected, wait 30 seconds and refresh — the agent sometimes takes a moment to register after startup.

---

## Part 4: Verify Logs Are Coming In

### Step 9 — Go to the Discover View

```
Menu → Explore → Discover
```

### Step 10 — Apply Agent Filter

```
agent.name: "ubuntu-agent"
```

You should start seeing log events appear in the timeline. These are real-time logs being forwarded from your Ubuntu agent.

---

## Part 5: Generate Test Events (Optional)

If you want to verify the full pipeline is working, run these on the Ubuntu VM to generate some detectable activity.

**Trigger a failed login attempt:**

```bash
su wronguser
```

**Run an nmap scan:**

```bash
nmap localhost
```

After running these, refresh the Discover view in the dashboard — new events should appear within a few seconds.

---

## Summary

| Component | What To Do |
|---|---|
| Server (Kali) | Start Docker, then start Wazuh containers |
| Agent (Ubuntu) | Start the `wazuh-agent` service |
| Dashboard | Open in browser and verify agent is Active |
| Logs | Check Discover view and filter by `ubuntu-agent` |

**Startup order:**

```
Start Docker → Start Wazuh Containers → Start Agent → Open Dashboard → Check Logs
```

No reinstallation needed. Every time you boot up, just follow these steps and your lab is ready to use.
