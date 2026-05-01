# Kubernetes Secrets — PicoCTF Writeup

> **Difficulty:** Medium  
> **Category:** Kubernetes, Web Security  
> **Objective:** Access a remote Kubernetes cluster, enumerate its Secrets, and retrieve the flag.

---

## 📋 Overview

The challenge provides a `kubeconfig` configuration file for a remote Kubernetes cluster. The goal is to explore the cluster's resources to find a **Secret** containing the flag. During the process, it is necessary to handle TLS configuration issues and navigate through different namespaces within the cluster.

---

## 🔍 Phase 1 — Configuration and TLS Resolution

### The Kubeconfig File

The provided `kubeconfig` file contains the necessary credentials (client certificates and keys) and the Kubernetes API server endpoint. Initially, the endpoint points to `localhost`, but it must be updated to reflect the remote challenge address: `https://green-hill.picoctf.net:63046`.

### Certificate Verification Issue

When attempting to connect to the new endpoint, a TLS verification error is encountered:
`tls: failed to verify certificate`.

This happens because the server's certificate was issued for `localhost` and not for the external PicoCTF domain. To proceed in a challenge environment, it is necessary to bypass TLS verification by modifying the configuration file:

```yaml
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: https://green-hill.picoctf.net:63046
```

> ⚠️ **Security Note:** The `insecure-skip-tls-verify: true` option disables protection against Man-in-the-Middle attacks. While it is common practice in CTFs, it is extremely dangerous in production environments.

---

## 🌐 Phase 2 — Cluster Enumeration

### Initial Connection

Once access is configured, verify the connection by listing the pods in the default namespace:

```sh
└─$ kubectl get pods
No resources found in default namespace.
```

The connection is successful, but the default namespace is empty. It is necessary to explore other namespaces.

### Identifying Namespaces

Listing all available namespaces:

```sh
└─$ kubectl get namespaces

NAME              STATUS   AGE
default           Active   24h
kube-system       Active   24h
kube-public       Active   24h
kube-node-lease   Active   24h
picoctf           Active   24h
```

The `picoctf` namespace is clearly the primary target for investigation.

---

## 🔐 Phase 3 — Analyzing Kubernetes Secrets

### What are Secrets?

In Kubernetes, a **Secret** is an object designed to store sensitive data (passwords, tokens, keys). However, by default, the data inside a Secret is simply **Base64** encoded, not encrypted. Anyone with the appropriate RBAC (Role-Based Access Control) permissions to read Secrets can view their content in plain text.

### Finding the Target Secret

Search for Secrets within the `picoctf` namespace:

```sh
└─$ kubectl get secrets -n picoctf

NAME                  TYPE                                  DATA   AGE
ctf-secret            Opaque                                1      24h
default-token-xxxxx   kubernetes.io/service-account-token   3      24h
```

We have identified `ctf-secret` as the likely container for the flag.

---

## 📄 Phase 4 — Extraction and Decoding

### Reading the Secret

Use the `-o yaml` flag to view the full details of the Secret, including the encoded data:

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

### Base64 Decoding

The value of the `flag` key is Base64 encoded. Use the command line to decode it:

```sh
└─$ echo "cGljb0NURntrczM**************************" | base64 -d

picoCTF{ks3**************************}
```

---
## 📊 Attack Summary

```
[Kubeconfig Modification: API server update]
        ↓
[TLS Bypass: insecure-skip-tls-verify]
        ↓
[Namespace Enumeration: identify 'picoctf']
        ↓
[Secret Enumeration: discover 'ctf-secret']
        ↓
[Data Extraction: read 'flag' field in YAML]
        ↓
[Base64 Decoding → Flag obtained]
```

---

## 🛠️ Tools Used

| Tool      | Purpose                                    |
|-----------|--------------------------------------------|
| `kubectl` | Interaction with the Kubernetes API server |
| `yaml`    | Format for kubeconfig manipulation         |
| `base64`  | Decoding data extracted from the Secret    |

---

## 📚 Lessons Learned

- **Base64 is not encryption**: Kubernetes Secrets are vulnerable if API server access is not protected by strict RBAC policies.
- **RBAC Security**: It is crucial to limit who can list and get Secrets in sensitive namespaces.
- **TLS Trust**: Bypassing certificate verification is a risk that should be avoided in real-world scenarios by using trusted CAs (Certificate Authorities).
