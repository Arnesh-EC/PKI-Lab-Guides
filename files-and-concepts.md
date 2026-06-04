---
---

# PKI File Types & Certificate Concepts

A reference for the file extensions and certificate kinds that appear throughout this
two-tier ADCS lab — what each one *is*, how it's *encoded*, and which machine produces or
consumes it.

---

## 1. The two things every file is made of

Before the extensions, two orthogonal ideas explain most of the confusion:

**(a) What's inside the file** — pick one or more:
- a **public certificate** (X.509: subject, public key, validity, issuer signature),
- a **private key** (the secret half — must never be shared),
- a **request** (a not-yet-signed cert proposal),
- a **revocation list** (which certs the CA has revoked),
- a **chain/bundle** (several of the above together).

**(b) How it's encoded** — pick one:
- **DER** — binary.
- **PEM** — Base64 text wrapped in `-----BEGIN ... -----` / `-----END ...` headers.

The *extension does not reliably tell you the encoding.* A `.cer` or `.crt` can be either
DER or PEM. This is why a cert sometimes "won't open" on another system — same content,
wrong encoding for that tool.

---

## 2. Extension-by-extension

| Ext | Contains | Has private key? | Typical encoding | One-liner |
|-----|----------|------------------|------------------|-----------|
| `.crt` | One public certificate | No | DER or PEM | A signed certificate. Unix/Linux convention. |
| `.cer` | One public certificate | No | DER or PEM | Same as `.crt`; Windows convention. |
| `.req` / `.csr` | A certificate **request** (PKCS#10) | No (holds the *public* key + subject) | PEM (usually) | "Please sign this." Sent to a CA. |
| `.rsp` | A CA's **response** to a request | No | DER/PEM | The issued cert (or chain) returned by the CA, used to complete a pending request. |
| `.p7b` / `.p7c` | A **chain** of certs (PKCS#7) | No | DER/PEM | Bundle of certs (e.g. issued cert + CA chain), no key. |
| `.crl` | A **Certificate Revocation List** | No | DER | The CA's signed list of revoked serial numbers. |
| `.pfx` / `.p12` | Cert **+ private key** (+ chain), PKCS#12 | **Yes** | Binary, password-protected | Portable bundle for *moving* an identity, key included. |
| `.pem` | Anything (cert, key, chain) | Maybe | PEM | Generic Base64 container; common on Linux/nginx/Apache. |
| `.key` | A **private key** alone | **Yes** | PEM (usually) | The bare secret key, often paired with a `.crt`. |
| `.der` | Usually one cert | No | DER | Explicitly binary-encoded cert. |

### On `.crt` vs `.cer`
Functionally identical — both are a single X.509 public certificate. The difference is
*cultural*: `.cer` is the Windows/ADCS default (what `certreq -retrieve` and the export
wizard hand you), `.crt` is the Unix default. Either can be DER or PEM internally.

### On `.req` vs `.rsp` (the "req / res" pair)
These are the two halves of an **offline signing handshake**, which is exactly how CA02 got
its cert from the offline root:
- **`.req`** = the *request* CA02 generated ("here is my public key and identity, please
  sign it"). Carried on removable media to CA01.
- **`.rsp`** / the returned `.crt` = the *response* — the signed certificate CA01 produced,
  carried back to CA02 to **complete the pending request** (*Install CA Certificate*).

A request never contains the private key; the key stays on the requesting machine the whole
time. That's the whole point of a CSR.

---

## 3. Who deals with what, per machine

| Machine | Produces | Consumes |
|---------|----------|----------|
| **CA01** (Root) | Root CA `.crt`, Root CA `.crl`; signs CA02's request → CA02 `.crt` | CA02's `.req` |
| **CA02** (Issuing) | Its own `.req`; issues end-entity `.crt`s; its own `.crl` (+ delta) | Root `.crt`/`.crl`; its signed `.crt` (`.rsp`) from CA01 |
| **SRV1** (Web/OCSP) | OCSP signing cert (auto-enrolled) | Hosts Root & Issuing `.crt` + `.crl` files for HTTP CDP/AIA |
| **WIN11** (Client) | Workstation Auth cert; exports `win11.cer` | The CA chain (to validate); CRLs/OCSP for verification |

Note the lab almost never touches private-key-bearing files (`.pfx`/`.key`) — because every
key is **generated in place** on the machine that uses it (root key on CA01, issuing key on
CA02, client key on WIN11). Only public material (`.crt`, `.req`, `.crl`)
travels between machines. That's good hygiene.

---

## 4. Is a `.pfx` strictly necessary for TLS?

**No.** A PFX is only needed when a certificate's **private key must move** — between
machines, or to/from a backup. It bundles the cert + private key (+ chain) into one
password-protected file precisely so the secret key can be transported safely.

For SRV1's HTTPS cert you can avoid it entirely:

| Approach | PFX needed? | How the key is handled |
|----------|-------------|------------------------|
| **Enroll directly on SRV1** (the lab's method) | **No** | Key is generated in SRV1's machine store and never leaves it. IIS just references it. |
| Generate/buy the cert on another box, then deploy to SRV1 | **Yes** | Export as PFX (cert+key), import on SRV1. |
| Linux/nginx/Apache style | No (but not PFX) | Two separate PEM files: a `.crt`/`.pem` cert + a `.key` private key. |
| Back up an in-place cert for disaster recovery | Yes | Export to PFX so the key is recoverable. |

So PFX is a **transport/backup format**, not a requirement of TLS itself. IIS *can* import a
PFX, and it *can* use a key that already lives in the local store — both work. Because we
enroll on SRV1, the key stays put and no PFX is involved.

> Security corollary: a PFX is as sensitive as a private key — anyone with the file and its
> password can impersonate the certificate's identity. Don't email it around or commit it.

---

## 5. How is a "TLS certificate" different from the other certs here?

They are **all X.509 certificates** — same file format, same structure. What makes one a
"TLS cert" versus a "CA cert" versus a "client cert" is two fields:

- **Basic Constraints** — `CA = true` means it can sign other certs (the Root and Issuing CA
  certs); `CA = false` means it's an end-entity (leaf) cert.
- **Extended Key Usage (EKU)** — an OID list declaring what the cert is *allowed to be used
  for*. This is the real differentiator:

| Cert in this lab | Basic Constraints | Key EKU(s) | What it's actually for |
|------------------|-------------------|------------|------------------------|
| **Root CA** (CA01) | CA = true | (none / all purposes) | Signs the Issuing CA cert; trust anchor at the top of the chain. |
| **Issuing CA** (CA02) | CA = true | (inherited) | Signs all end-entity certs; the everyday workhorse. |
| **OCSP Response Signing** (SRV1) | CA = false | OCSP Signing `1.3.6.1.5.5.7.3.9` | Signs OCSP "is this cert revoked?" responses so clients trust the answer. |
| **Workstation Authentication** (WIN11) | CA = false | **Client** Auth `1.3.6.1.5.5.7.3.2` | The *machine* proves *its* identity to a server (802.1X, IPsec, mTLS). |
| **Web Server / TLS** *(not built in this lab)* | CA = false | **Server** Auth `1.3.6.1.5.5.7.3.1` | A *server* proves *its* identity to clients and sets up the encrypted HTTPS channel. |

Key distinctions to internalize:

- **CA cert vs leaf cert:** CA certs *sign* other certificates; leaf certs (TLS, client,
  OCSP) are signed *by* a CA and cannot issue anything. Don't put a CA cert on a web server's
  binding, and don't try to sign certs with a TLS cert.
- **Server Auth vs Client Auth:** these are mirror images. A **TLS/Web Server** cert
  (Server Auth) answers *"prove you are the server I'm connecting to"* — it needs a **SAN**
  matching the hostname the browser typed. A **Workstation Authentication** cert (Client
  Auth) answers *"prove which machine you are"* when connecting *to* something. Same shape,
  opposite roles; a cert without Server Auth EKU will be rejected for HTTPS even if otherwise
  valid.
- **Why the TLS cert needs a SAN and the others may not:** TLS clients match the cert against
  the *name they connected to*. So a TLS cert for SRV1 (were you to add HTTPS) would have to
  list `srv1.EncryptionConsulting.com` **and** `pki.EncryptionConsulting.com` — valid under
  either name. CA and OCSP certs are matched by chain/role, not by a typed hostname, so they
  don't carry web SANs.

### What each is useful for, in one line
- **Root CA cert** — the anchor every client must trust; signs the Issuing CA. Kept offline.
- **Issuing CA cert** — issues and revokes all day-to-day certs.
- **CRL (`.crl`)** — lets a client check, offline, whether a cert was revoked.
- **OCSP signing cert** — lets SRV1 give a signed, real-time revocation answer instead of a CRL download.
- **Workstation Authentication cert** — machine-to-service authentication (the device proves who it is).
- **Web Server / TLS cert** — encrypts HTTPS and proves the server's hostname to browsers.
