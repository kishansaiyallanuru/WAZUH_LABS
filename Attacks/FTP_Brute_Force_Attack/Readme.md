# FTP Brute Force Attack Detection using Wazuh SIEM

## **Date:** 6th & 7th May, 2026

A hands-on cybersecurity task where I simulated FTP brute force attacks and detected them using Wazuh SIEM. I set this up in a controlled virtual lab across three machines. Everything here is strictly for educational purposes.

---

## Task Environment

I used three VirtualBox VMs for this — a Kali attacker, an Ubuntu victim running vsftpd, and a Wazuh server running in Docker on Kali. The Ubuntu machine has a Wazuh agent installed that forwards FTP authentication events to the Wazuh manager in near real time.

| Role         | OS              | IP Address      |
|--------------|-----------------|-----------------|
| Attacker     | Kali Linux      | 10.181.125.131  |
| Victim       | Ubuntu          | 10.181.125.208  |
| Wazuh Server | Kali (Docker)   | 10.181.125.37   |

---

## Objective

- Set up a real FTP service on Ubuntu using vsftpd
- Simulate FTP brute force attacks using Hydra and Medusa
- Monitor and detect the attacks through Wazuh
- Analyze logs, rule IDs, and event patterns in the Wazuh dashboard
- Map the attack to MITRE ATT&CK techniques

---

## Tools Used

**Wazuh SIEM** — The main detection tool. I deployed a Wazuh agent on the Ubuntu victim and configured it to read vsftpd logs. Every FTP login attempt showed up in the Wazuh dashboard in real time with decoded fields and rule alerts.

**Hydra** — A fast brute force tool that supports FTP. I used it in two ways — once targeting a single username, and once with a full username and password list.

**Medusa** — Another brute force tool. More controlled than Hydra. I ran it with `-t 2` to keep it stable inside the VM and used `-v 6` for full verbose output.

**Nmap** — I used this before the attacks to confirm FTP was running on port 21 on the victim.

**vsftpd** — The FTP server I installed on Ubuntu. It writes authentication events to `/var/log/vsftpd.log`, which Wazuh reads after I configured the agent.

---

## Hydra vs Medusa — Comparison

| Feature | Hydra | Medusa |
|---|---|---|
| Speed | Faster | Slower but steadier |
| VM Stability | Can cause timeouts at high threads | More stable with `-t 2` |
| Protocol Support | 50+ protocols | ~20+ protocols |
| Output | Compact, shows result | Verbose, shows every attempt |
| Best Use in Lab | Use `-t 2` or lower | Excellent for clean log stream |

---

## Lab Setup

Before any attacks, I set up the FTP service on the Ubuntu victim and created a user account with a weak password.

### Installing and Starting vsftpd

```bash
sudo apt update
sudo apt install vsftpd -y
sudo systemctl start vsftpd
sudo systemctl enable vsftpd
sudo systemctl status vsftpd
sudo ss -tulnp | grep :21
```

![FTP Service Status](Images/ftp_service_status.png)

I installed vsftpd on the Ubuntu victim, started and enabled the service, then checked its status. The output shows `vsftpd.service` is active and running. The last line from `ss -tulnp | grep :21` confirms `tcp LISTEN 0 32 *:21`, meaning vsftpd is listening on port 21 on all interfaces. With this confirmed, the FTP service was ready.

### Configuring vsftpd

```bash
sudo nano /etc/vsftpd.conf
```

```ini
listen=YES
listen_ipv6=NO
anonymous_enable=NO
local_enable=YES
write_enable=YES
```

I disabled anonymous access and enabled local user logins so the brute force attacks would hit real authentication.

### Creating the Victim User

```bash
sudo adduser victim
```

I created a user called `victim` with the password `test`. The weak password was intentional — I needed the brute force tools to crack it within a small wordlist so I could observe the full detection chain in Wazuh.

---

## Network Verification

Before any attacks, I confirmed both machine IPs and verified FTP was reachable from the attacker.

### Victim IP

```bash
ip a
```

![Victim IP](Images/victim_ip.png)

The `ip a` output on the Ubuntu victim shows `enp0s3` with IP `10.181.125.208`. This is the target address I used in every attack command throughout this task.

### Attacker IP

```bash
ip a
```

![Attacker IP](Images/attacker_ip.png)

The Kali attacker machine shows `eth0` with IP `10.181.125.131`. This is the source IP that appears in every Wazuh log entry, identifying where the attacks came from.

### Nmap Scan to Confirm FTP is Open

```bash
nmap -sV 10.181.125.208
```

![Nmap FTP Scan](Images/nmap_ftp_scan.png)

I ran a service version scan from the Kali attacker against the victim. Port 21 shows as open, running vsftpd 3.0.5. Port 22 is also open, which is expected. The scan completed in 1.10 seconds with no firewall interference. This confirmed FTP was reachable before I started any brute force attacks.

---

## Configuring Wazuh Agent to Monitor FTP Logs

By default, Wazuh doesn't monitor `/var/log/vsftpd.log`. I had to add a `localfile` block to the agent config on the Ubuntu victim.

```bash
sudo nano /var/ossec/etc/ossec.conf
```

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/vsftpd.log</location>
</localfile>
```

![Wazuh Agent FTP Log Config](Images/Add_ftp_logs_wazuhagent_req.png)

The screenshot shows the `ossec.conf` file with the `<localfile>` block added. Without this, none of the FTP events would reach Wazuh at all. This was the key configuration step that enabled the whole detection pipeline.

### Restarting the Wazuh Agent

```bash
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```

![Wazuh Agent Running](Images/wazuh_agent_running.png)

After saving the config, I restarted the agent. The status output confirms `wazuh-agent.service` is active and running. The process tree shows all expected components running — `wazuh-execd`, `wazuh-agentd`, `wazuh-logcollector`, `wazuh-syscheckd`, and `wazuh-modulesd`. Journal entries at the bottom confirm everything started cleanly. The agent was now forwarding vsftpd events to the Wazuh manager.

---

## Attack Method 1 — Manual FTP Authentication Testing

I started with manual FTP logins to verify Wazuh was picking up FTP events before moving to automated tools. I made both successful and failed attempts.

```bash
ftp 10.181.125.208
```

![Manual FTP Login](Images/manual_ftp_login.png)

I connected to the victim's FTP service from the Kali machine. The vsftpd banner `220 (vsFTPd 3.0.5)` appeared, I entered the `victim` credentials, and got `230 Login successful`. This confirmed end-to-end FTP connectivity and that the victim account was working before any brute force attacks.

### Wazuh Document — FTP Auth Success

![Manual FTP Login Document](Images/manual_ftp_login_doc.png)

I opened the Wazuh document for the successful login. Key fields: `rule.id: 11402` (vsftpd: FTP Authentication success), `rule.level: 3`, `data.dstuser: victim`, `data.srcip: ::ffff:10.181.125.131`, `data.status: OK LOGIN`, `decoder.name: vsftpd`. The `full_log` shows `[victim] OK LOGIN: Client "::ffff:10.181.125.131"`. MITRE T1078 was auto-tagged. This confirmed Wazuh was decoding vsftpd events correctly.

### Wazuh Filtered View — Manual Login Events

![Manual FTP Login Filter](Images/manual_ftp_login_filter.png)

Filtering by the attacker IP in Wazuh showed hits from the session. The event list shows `rule.id: 11402` (auth success), `rule.id: 11401` (FTP session opened), and `rule.id: 11403` (login failed) from the intentional bad attempts. The histogram shows the activity at the start of the session. Wazuh was confirmed working, so I moved to the automated attacks.

---

## Attack Method 2 — Hydra Single Username FTP Brute Force

I used Hydra with a single username and a custom password list I created manually.

```bash
hydra -l victim -P pass.txt ftp://10.181.125.208
```

![Hydra FTP Single User](Images/hydra_ftp_single.png)

Hydra v9.6 started at `15:33:06` and worked through 7 passwords for the `victim` username. The result shows `[21][ftp] host: 10.181.125.208 login: victim password: test`. Hydra found it quickly since `test` was in my wordlist. I verified by opening a manual FTP session right after — `230 Login successful` and the `ls` command showed the victim's home directory, confirming full access.

### Wazuh Document — Rule 40112

![Hydra FTP Single User Document](Images/hydra_ftp_single_doc.png)

The most important alert from this attack: `rule.id: 40112` at `rule.level: 12` — "Multiple authentication failures followed by a success." This fires when Wazuh sees multiple failed logins from the same source followed immediately by a success — the exact pattern of a successful brute force. The `full_log` shows `[victim] OK LOGIN: Client "::ffff:10.181.125.131"`. MITRE T1078 and T1110 are both tagged. `rule.mail: true` means this would send an email notification in production.

### Wazuh Filtered View

![Hydra FTP Single User Filter](Images/hydra_ftp_single_filter.png)

After the Hydra single-user attack, the Wazuh dashboard shows 1,252 total hits filtered by attacker IP. The top event is the severity-12 `rule.id: 40112` alert, followed by `rule.id: 11401` and `rule.id: 11403` events. The histogram shows a clear spike at the time of the attack. The attacker IP and `data.dstuser: victim` are visible in every row.

---

## Attack Method 3 — Hydra Multi-User FTP Brute Force

I escalated by targeting multiple usernames at once using both `users.txt` and `pass.txt`, simulating an attacker who doesn't know which accounts exist on the system.

```bash
hydra -L users.txt -P pass.txt -t 2 ftp://10.181.125.208
```

![Hydra FTP Multi-User](Images/hydra_ftp_double.png)

Hydra worked through `56 login tries (l:8/p:7)` — 8 usernames × 7 passwords. The result shows `login: victim password: test` found. I used `-t 2` to keep it stable in the VM. After Hydra finished, I verified by connecting manually — `230 Login successful` confirmed it. The multi-user run generated significantly more Wazuh events than the single-user run.

### Wazuh Threat Hunting — Multi-User All Logs

![Hydra Multi-User Threat Hunt](Images/Hydra_multi_all_logs_threathunt.png)

The Threat Hunting view shows the full event stream from this attack. At the top, `rule.id: 40112` at level 12 fires first. Below that, `rule.id: 11403` (login failed) and `rule.id: 5503` (PAM: User login failed) repeat across every failed attempt. `rule.id: 11451` (vsftpd: FTP brute force — multiple failed logins) also appears at level 10 — this is Wazuh's dedicated FTP brute force rule that fires even before a successful login happens. `rule.id: 5551` (PAM: Multiple failed logins in a small period) triggered too. Multiple rules fired independently from a single attack run.

### Wazuh Document — Multi-User Rule 40112

![Hydra Multi-User Document](Images/Hydra_multi_all_logs_threathunt_doc.png)

The document for the severity-12 alert shows `rule.id: 40112`, `rule.firedtimes: 3`, `data.dstuser: victim`, `data.srcip: ::ffff:10.181.125.131`, `data.status: OK LOGIN`. The `full_log` reads `[victim] OK LOGIN: Client "::ffff:10.181.125.131"` at `15:42:19`. MITRE T1078 and T1110 tagged. PCI-DSS, HIPAA, GDPR, and NIST 800-53 compliance entries are all auto-populated. The `rule.firedtimes: 3` means this composite alert had now fired three times across the session.

---

## Attack Method 4 — Medusa FTP Brute Force

The last attack used Medusa with full verbose output.

```bash
medusa -h 10.181.125.208 -U users.txt -P pass.txt -M ftp -t 2 -v 6
```

![Medusa FTP Attack](Images/medusa_ftp_attack.png)

Medusa v2.3 worked through all 56 combinations. Each line shows `ACCOUNT CHECK: [ftp] Host: 10.181.125.208 User: <username> Password: <password>` with a timestamp. When it reached `victim:test`, the output changed to `ACCOUNT FOUND: [ftp] Host: 10.181.125.208 User: victim Password: test [SUCCESS]`. I confirmed by opening a manual FTP session after — `230 Login successful`. The verbose output made it easy to watch every attempt in real time.

### Wazuh Document — Medusa Rule 40112

![Medusa FTP Document](Images/medusa_ftp_attack_doc.png)

The document for the Medusa attack shows the same `rule.id: 40112` at `rule.level: 12` as every previous successful brute force. `rule.firedtimes: 5`, `data.dstuser: victim`, `data.srcip: ::ffff:10.181.125.131`, `data.status: OK LOGIN`. The `decoder.name: vsftpd` is identical across all four methods. This confirms that switching attack tools changes nothing — Wazuh detects the behavior in the vsftpd logs regardless of what generated it.

### Wazuh Filtered View — Medusa vsftpd Events

![Medusa FTP Filter](Images/medusa_ftp_attack_filter.png)

Filtering by `vsftpd` after the Medusa attack shows 316 hits, all in a sharp spike on the right side of the histogram — exactly when Medusa ran. The top event is the severity-12 `rule.id: 40112` alert, followed by `rule.id: 11401` and a stream of `rule.id: 11403` failed login entries. One event shows a batched `previous_output` where Wazuh aggregated multiple consecutive FAIL LOGIN lines together. Every entry shows `data.srcip: ::ffff:10.181.125.131` and `decoder.parent: vsftpd`.

---

## Log Analysis Using Wazuh

The Ubuntu victim runs Wazuh Agent (agent ID: 001, named `kishansai-VirtualBox`) configured to read `/var/log/vsftpd.log`. Every vsftpd event was forwarded to the Wazuh Manager, decoded using the `vsftpd` decoder, and stored in index `wazuh-alerts-4.x-2026.05.07`.

### Key Wazuh Rule IDs Observed

| Rule ID | Description | Severity |
|---|---|---|
| 11401 | vsftpd: FTP session opened | 3 |
| 11402 | vsftpd: FTP Authentication success | 3 |
| 11403 | vsftpd: Login failed accessing the FTP server | 5 |
| 11451 | vsftpd: FTP brute force (multiple failed logins) | 10 |
| 5503  | PAM: User login failed | 5 |
| 5551  | PAM: Multiple failed logins in a small period of time | 10 |
| 40112 | Multiple authentication failures followed by a success | 12 |

### Event Volume by Attack Method

| Attack Method | Tool | Wazuh Event Count |
|---|---|---|
| Method 1 — Manual FTP | Manual | Baseline |
| Method 2 — Hydra Single User | Hydra | ~25 hits |
| Method 3 — Hydra Multi-User | Hydra | ~56+ hits |
| Method 4 — Medusa | Medusa | 316 hits (vsftpd filter) |

### MITRE ATT&CK Mapping

| MITRE ID | Technique | Tactic |
|---|---|---|
| T1110 | Brute Force | Credential Access |
| T1110.001 | Password Guessing | Credential Access |
| T1078 | Valid Accounts | Defense Evasion, Persistence, Initial Access |
| T1021 | Remote Services | Lateral Movement |

---

## What I Observed

**Weak passwords get cracked immediately.** The `victim` account used `test` as the password. Both Hydra and Medusa found it almost instantly because it was sitting in a 7-entry wordlist.

**Every failed attempt is visible in Wazuh.** Even a single Hydra run with 7 passwords generated dozens of log entries. That volume of failed authentications from one IP in seconds is very easy to spot.

**Wazuh has a dedicated FTP brute force rule.** `rule.id: 11451` fires specifically on FTP brute force behavior, even before a success happens. I saw it fire alongside the broader `rule.id: 40112` during the multi-user attacks.

**The tool doesn't matter for detection.** I switched between Hydra and Medusa across four attack sessions and the exact same rule IDs fired every time. Wazuh detects the behavior in the logs, not the tool.

**Rule 40112 is the key alert.** It fires when multiple failures are followed by a success from the same source — the direct signature of a successful brute force. It fired every time.

**PAM rules fire independently.** `rule.id: 5503` and `5551` triggered from PAM alongside the vsftpd-specific rules, giving a second detection layer on top of the FTP rules.

---

## Conclusion

This task gave me a clear end-to-end view of FTP brute force from both sides. I set up vsftpd on Ubuntu, configured Wazuh to monitor the FTP logs, and ran four attack methods — manual login, Hydra single-user, Hydra multi-user, and Medusa. Every attack was detected. Rule 40112 at severity 12 fired each time a brute force succeeded. MITRE T1078 and T1110 were auto-tagged across all events. The main takeaway: FTP with weak passwords and no lockout policy is easy to compromise, and a properly configured SIEM catches every attempt in real time.

---

## Commands Reference

### Victim Machine — FTP Setup

```bash
sudo apt update
sudo apt install vsftpd -y
sudo systemctl start vsftpd
sudo systemctl enable vsftpd
sudo systemctl status vsftpd
sudo ss -tulnp | grep :21
sudo nano /etc/vsftpd.conf
sudo adduser victim
ip a
sudo tail -f /var/log/vsftpd.log
```

### Victim Machine — Wazuh Agent

```bash
sudo nano /var/ossec/etc/ossec.conf
sudo systemctl restart wazuh-agent
sudo systemctl status wazuh-agent
```

```xml
<localfile>
  <log_format>syslog</log_format>
  <location>/var/log/vsftpd.log</location>
</localfile>
```

### Attacker Machine — Verification

```bash
ip a
nmap -sV 10.181.125.208
ftp 10.181.125.208
```

### Attack Method 2 — Hydra Single User

```bash
hydra -l victim -P pass.txt ftp://10.181.125.208
```

### Attack Method 3 — Hydra Multi-User

```bash
hydra -L users.txt -P pass.txt -t 2 ftp://10.181.125.208
```

### Attack Method 4 — Medusa

```bash
medusa -h 10.181.125.208 -U users.txt -P pass.txt -M ftp -t 2 -v 6
```
