---
layout: default
title: Home
---

# EC Two-Tier PKI Lab Guides

Collection of quick reference guides for building a **two-tier Microsoft ADCS PKI hierarchy**: a standalone
**offline root CA**, an enterprise **issuing CA**, an **OCSP** Online Responder, and
HTTP/LDAP **CRL + AIA** distribution — on Windows Server 2025 VMs with a Windows 11 client.

Please feel free to navigate the guides below.

## Guides

- **[PKI Lab — VM Summary & Build Guide](vm-building.html)** — the five lab VMs (DC01, CA01,
  CA02, SRV1, WIN11), the build-order dependency graph, and the full per-machine setup, with
  inline PowerShell for every step.
- **[PKI File Types & Certificate Concepts](files-and-concepts.html)** — `.crt` / `.cer` /
  `.req` / `.crl` / `.pfx`, DER vs PEM, and how a TLS cert differs from CA/OCSP/client certs.
