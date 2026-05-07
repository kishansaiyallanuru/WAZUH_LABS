
# Wazuh Docker Stack — Upgrade & Fix Journal

> Upgrading from `4.10.3` → `4.14.5` and fixing the certificate mess that came with it.

---

## What Happened

I had a Wazuh agent running on my Ubuntu VM at version `4.14.5` — installed it fresh and didn't realise my Docker stack was still sitting on `4.10.3`. The moment the agent tried to connect, it threw:

```
Agent version must be lower or equal to manager version
```

Fair enough. The fix seemed simple — just update the image versions in `docker-compose.yml`, pull, restart. And it was... until the Wazuh Dashboard decided to greet me with:

```
unable to verify the first certificate
```

Turns out the old Docker volumes were still holding onto stale TLS certificates from the previous deployment. The new containers came up but couldn't talk to the indexer properly because the certs didn't match.

So what started as a one-liner version bump turned into a full clean reinstall. Here's exactly what I did, in order.

---

## Environment

| Component           | Before   | After    |
|---------------------|----------|----------|
| Wazuh Manager       | 4.10.3   | 4.14.5   |
| Wazuh Dashboard     | 4.10.3   | 4.14.5   |
| Wazuh Indexer       | 4.10.3   | 4.14.5   |
| Wazuh Agent (Ubuntu)| 4.14.5   | 4.14.5 ✅|

**Host:** Kali Linux (Docker single-node deployment)
**Agent:** Ubuntu VM

---

## Part 1 — The First Attempt (Version Bump)

### Step 1 — Confirm the agent version

First I wanted to see what version the Ubuntu agent was actually reporting.

```bash
dpkg -l | grep wazuh
```

Output:
```
wazuh-agent    4.14.5
```

Yep — `4.14.5`. And the manager was on `4.10.3`. That explained everything.

---

### Step 2 — Check what Docker was running

```bash
docker ps
```

Confirmed all three stack components on the old version:

```
wazuh-manager:4.10.3
wazuh-dashboard:4.10.3
wazuh-indexer:4.10.3
```

---

### Step 3 — Find the compose file

I didn't remember the exact path off the top of my head, so:

```bash
find ~ -name "docker-compose.yml" 2>/dev/null
```

Found it at `~/wazuh-docker/single-node/docker-compose.yml`.

---

### Step 4 — Navigate there

```bash
cd ~/wazuh-docker/single-node
```

---

### Step 5 — Edit the compose file

Opened it in nano and replaced every occurrence of `4.10.3` with `4.14.5`. There are three image lines — one each for the manager, indexer, and dashboard.

```bash
nano docker-compose.yml
```

In nano: `Ctrl + \` → find `4.10.3` → replace with `4.14.5` → replace all.

Or just do it from the terminal in one shot:

```bash
sed -i 's/4\.10\.3/4.14.5/g' docker-compose.yml
```

Verified the change landed:

```bash
grep "image:" docker-compose.yml
```

All three lines now showed `4.14.5`. Good.

---

### Step 6 — Pull the new images

```bash
docker compose pull
```

Downloaded the new layers for all three services. Took a few minutes.

---

### Step 7 — Restart the stack

```bash
docker compose down
docker compose up -d
```

Containers came up. Looked fine from `docker ps`. But then I opened the browser and hit the Dashboard URL — and ran straight into the certificate error.

---

## Part 2 — The Certificate Problem

### Step 8 — Check the Dashboard logs

```bash
docker logs -f single-node-wazuh.dashboard-1
```

Two errors repeating:

```
unable to verify the first certificate
ECONNREFUSED
```

The dashboard couldn't verify the indexer's TLS certificate. The old volumes from the `4.10.3` deployment were still mounted, and they contained certificates that didn't match what the new containers expected.

---

### Step 9 — Remove the old certificates

```bash
sudo rm -rf config/wazuh_indexer_ssl_certs
```

---

### Step 10 — Try regenerating certificates

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

---

### Step 11 — Remove old volumes completely

The cert regeneration alone wasn't enough. The volumes themselves were stale, so I brought everything down and removed them:

```bash
docker compose down -v
```

Then checked that the volumes were actually gone:

```bash
docker volume ls | grep single-node
```

---

At this point I'd gone back and forth enough times that I decided to stop fighting the old setup and just do a clean install from scratch.

---

## Part 3 — Clean Reinstall (The Right Move)

### Step 12 — Back up and move the old directory

```bash
cd ~
mv wazuh-docker wazuh-docker-old
```

Kept it as a backup rather than deleting it outright. Just in case.

---

### Step 13 — Clone a fresh copy of the repo

```bash
git clone https://github.com/wazuh/wazuh-docker.git
cd wazuh-docker/single-node
```

---

### Step 14 — Checkout the right version tag

This is the important step I'd skipped the first time. Instead of just editing image tags, checking out the correct git tag ensures the compose file, config files, and cert generation scripts are all consistent with `4.14.5`.

```bash
git checkout v4.14.5
```

---

### Step 15 — Generate fresh certificates

With a clean directory and the correct version checked out, the cert generator ran without issues:

```bash
docker compose -f generate-indexer-certs.yml run --rm generator
```

---

### Step 16 — Start the stack

```bash
docker compose up -d
```

---

### Step 17 — Verify everything came up on the right version

```bash
docker ps
```

Output:
```
wazuh-manager:4.14.5     Up
wazuh-dashboard:4.14.5   Up
wazuh-indexer:4.14.5     Up
```

All three services running, all on `4.14.5`.

---

## Part 4 — Agent Connection

### Step 18 — Restart the Wazuh agent on Ubuntu

```bash
sudo systemctl restart wazuh-agent
```

---

### Step 19 — Watch the agent logs

```bash
sudo tail -f /var/ossec/logs/ossec.log
```

The version mismatch error was gone. Instead I saw:

```
Connected to manager
```

---

### Step 20 — Open the Dashboard

```bash
https://10.181.125.37
```

Accepted the browser security warning (self-signed cert), logged in, and the Dashboard loaded cleanly. Agents showed as Active.

---

## What I'd Do Differently Next Time

The version bump approach *can* work, but only if you also handle the certificates properly. The safer path for a major version jump is always the clean reinstall:

- Clone fresh from the repo
- Check out the exact version tag (don't just edit image strings)
- Let the cert generator run clean before starting containers
- Only then bring the stack up

Editing `docker-compose.yml` image tags without checking out the matching git tag leaves the rest of the config files out of sync. That's what caused the cert issue.

---

## Lessons Learned

The actual version mismatch error has a straightforward fix. The certificate problem caught me off guard because it only showed up *after* the containers were already running — everything looked fine from `docker ps` until I opened the browser.

If you hit `unable to verify the first certificate` after a Wazuh Docker upgrade, don't keep regenerating certs on top of old volumes. Bring the whole thing down with `docker compose down -v`, clone fresh, check out the right git tag, and regenerate from scratch. That's the clean path.
