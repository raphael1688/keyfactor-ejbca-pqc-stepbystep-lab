# Module 00: Getting Started with EJBCA® Community Edition

*This lab is part of the [Post-Quantum Cryptography Step-by-Step Lab](https://github.com/f5devcentral/openssl-pqc-stepbystep-lab) series. For educational and internal testing purposes. Production deployments should use HSMs, air-gapped Root CAs, and follow organizational security policies.*

## Introduction — From OpenSSL Knowledge to Enterprise PKI Management

This is our second journy of the [Post-Quantum Cryptography Step-by-Step Lab](https://github.com/f5devcentral/openssl-pqc-stepbystep-lab) series. In Part 1 we built a PQC CA hierarchy from scratch using OpenSSL and learned what every flag, every file, and every config directive actually does. Here, you take that knowledge and build the same CAs inside an modern PKI platform — because knowing `openssl ca` flags by heart is great at parties but doesn't scale.

## Why Native Creation?

You might be wondering: "We already built PQC CAs in OpenSSL — why not just import those keys and certificates into EJBCA?"

Great question. We tried. Here's what happens:

The current `importca` command packages external CA keys and certificates into PKCS#12 keystores and imports them as soft crypto tokens. The problem is that the code path that creates the signer during import **assumes RSA keys**. When you hand it an ML-DSA key, it throws:

```
OperatorCreationException: cannot create signer: Supplied key (BCMLDSAPrivateKey) is not a RSAPrivateKey instance
```

This is an current limitation in EJBCA® Community Edition — the `importca` signer code path casts to `RSAPrivateKey` internally. It's not a bug in the PQC algorithm support itself; EJBCA's Bouncy Castle provider handles ML-DSA just fine for **native** crypto token operations. The import path just hasn't been updated for non-RSA key types yet. We'll let you know when this changes.

All of Keyfactor's own PQC examples (including their [PQC Hybrid CA Tutorial](https://docs.keyfactor.com/ejbca/9.0/tutorial-create-pqc-hybrid-ca-chain)) use **native crypto token creation** inside EJBCA, never external import. So that's what we'll do too.

The good news: everything you learned in the OpenSSL lab directly informs how we configure these CAs. Same algorithm choices (ML-DSA-87 for Root, ML-DSA-65 for Intermediate), same SassyCorp organizational identity, same chain-of-trust architecture. We're just building them inside EJBCA's crypto token framework instead of importing them from outside.

> **💡 Your OpenSSL CAs aren't wasted.** They taught you how PQC certificate authorities work at the deepest level — key generation, certificate signing, chain validation, revocation. That knowledge is exactly what makes this lab possible. You'll know *why* every configuration choice matters because you've already done it by hand.

<br>

## What is EJBCA®?

EJBCA® (Enterprise Java Beans Certificate Authority) is an open-source PKI mangement platform developed by Keyfactor. It's one of the more widely deployed PKI platforms and solves for:

- **Certificate Lifecycle Management** — issuance, renewal, revocation, and expiration tracking
- **Protocol Support** — SCEP, CMP, EST, ACME, OCSP, and REST APIs
- **Role-Based Access Control** — granular permissions for administrators and operators
- **Audit Logging** — comprehensive, tamper-evident logging of all PKI operations
- **Crypto Token Management** — soft tokens, PKCS#11, and HSM integration

The Community Edition (CE) provides the core CA functionality we need for this lab. The Enterprise Edition adds features like HA, cloud HSM support, and advanced compliance tools — but CE is more than sufficient for learning and many environments.

<br>

## What We're Building

This lab follows the official [Keyfactor EJBCA installation documentation](https://docs.keyfactor.com/ejbca-software/latest/installation) and extends it with native PQC CA creation. Here's the full stack:

```
┌──────────────────────────────────────────────┐
│       EJBCA Admin Web UI (Browser)           │
│  Port 8080 — HTTP (public pages)             │
│  Port 8442 — HTTPS (server auth only)        │
│  Port 8443 — HTTPS (mTLS, client cert REQ)   │
├──────────────────────────────────────────────┤
│        EJBCA Community Edition v9.3          │
│     (git clone from Keyfactor/ejbca-ce)      │
├──────────────────────────────────────────────┤
│         WildFly 35.0.1.Final App Server      │
│         (Jakarta EE Platform)                │
├──────────────────────────────────────────────┤
│      OpenJDK 21 (ARM64 or AMD64)             │
├──────────────────────────────────────────────┤
│         MariaDB Database                     │
└──────────────────────────────────────────────┘
```

### The 3-Port Architecture

EJBCA uses three distinct ports, each with a different security model. Understanding this upfront saves a lot of confusion later:

| Port | Protocol | TLS Mode | Purpose |
|------|----------|----------|---------|
| **8080** | HTTP | None | Public enrollment pages, health checks, unencrypted access |
| **8442** | HTTPS | Server auth only | Public HTTPS access — server presents its certificate, no client cert required |
| **8443** | HTTPS | Mutual TLS (mTLS) | Admin web interface — server presents its cert AND requires a client certificate from your browser |

Port 8443 is where all the admin magic happens. When you navigate to `https://localhost:8443/ejbca/adminweb/`, WildFly demands your browser present a client certificate (the SuperAdmin cert we'll generate). If your browser doesn't have one, you get rejected. This is mutual TLS authentication — both sides prove their identity.

## Prerequisites

### Completed Prior Lab

You **really should** complete the [NIST FIPS PQC Lab](https://github.com/f5devcentral/openssl-pqc-stepbystep-lab/tree/main/fipsqs) before starting this guide. It provides the first steps to understanding how PQC certificate authorities operate at a granular level. Specifically we would end up with:

- 🌈 An ML-DSA-87 Root CA with key and certificate at `/opt/sassycorp-pqc/root-ca/`
- 🌈 An ML-DSA-65 Intermediate CA with key and certificate at `/opt/sassycorp-pqc/intermediate-ca/`
- 🌈 A working certificate chain (Root → Intermediate → End Entity)
- 🌈 OpenSSL 3.5.x with native PQC algorithm support

You don't **HAVE** to do complete the first learning path but we do recommend it.  Grab some coffee, we'll wait. ☕ 

*Note: I'm tired of AI ruining green checks so I'm using rainbows and unicorns now. Bring it.*

### System Requirements

| Component | Requirement |
|-----------|-------------|
| **Operating System** | Ubuntu 25.10 (same VM/system from prior lab) |
| **Architecture** | AMD64 (x86_64) or ARM64 (aarch64) — both supported |
| **RAM** | 4 GB minimum (8 GB recommended — WildFly alone wants 2 GB heap) |
| **Permissions** | Root or sudo access |
| **Network** | Internet access for package downloads, git clone, and for taking breaks on slashdot |

### Software Stack

| Software | Version | Purpose |
|----------|---------|---------|
| **OpenJDK** | 21 (21.0.5+) | Java runtime for EJBCA® and WildFly |
| **WildFly** | 35.0.1.Final | Jakarta EE application server |
| **MariaDB** | Latest from Ubuntu repos | Database backend for EJBCA |
| **Apache Ant** | 1.9.8+ | Build tool for EJBCA compilation and deployment |
| **Git** | Latest from Ubuntu repos | Clone EJBCA CE source code |
| **EJBCA® CE** | 9.3 | The PKI platform itself |

### Required Knowledge

- Everything from the prior lab (Linux CLI, PKI concepts, X.509 structure)
- Basic understanding of Java application servers (helpful but not required — we'll explain as we go)
- Familiarity with database concepts (tables, users, privileges)

<br>

## Architecture Support — ARM64 and AMD64

This lab supports both ARM64 (aarch64) and AMD64 (x86_64) architectures. OpenJDK 21 provides native builds for both. Throughout this guide, when a step differs between architectures, we'll call it out explicitly. In most cases, the installation is identical.

To check your architecture:

```bash
uname -m
```

**Expected Output:**
- `x86_64` — you're on AMD64
- `aarch64` — you're on ARM64

### Where Our OpenSSL PQC CAs Currently Resides (Reference)

From the prior lab, your SassyCorp CA infrastructure should still be at:

```
/opt/sassycorp-pqc/
├── root-ca/
│   ├── certs/
│   │   └── root-ca.crt          ← ML-DSA-87 Root CA certificate
│   ├── private/
│   │   └── root-ca.key          ← ML-DSA-87 Root CA private key
│   ├── crl/
│   ├── newcerts/
│   ├── index.txt
│   └── serial
└── intermediate-ca/
    ├── certs/
    │   ├── intermediate-ca.crt  ← ML-DSA-65 Intermediate CA certificate
    │   └── ca-chain.crt         ← Full certificate chain
    ├── private/
    │   └── intermediate-ca.key  ← ML-DSA-65 Intermediate CA private key
    ├── crl/
    ├── newcerts/
    ├── index.txt
    └── serial
```

> **💡 Note:** We won't be importing these files into EJBCA (see "Why Native Creation" above), but they serve as our reference for the organizational identity and algorithm choices we'll replicate. Your OpenSSL CAs remain intact as a working reference and backup/secondary CA.

<br>

## Manual Entry — Not Scripts

Just like the OpenSSL PQC lab, this guide uses **manual command entry only — no scripts**. No Docker Compose, no signing up for prebuilt configurations. Every command you type, every configuration file you edit, every CLI interaction is done by hand. Blame our love for "Learning Python the Hard Way" for this method.

Why? Because:

1. **You learn the tools.** You'll understand WildFly's JBoss CLI, EJBCA's Ant commands, MariaDB's SQL interface, and systemd service management.
2. **You learn the "why."** Every command has context explaining what it does and why it matters.
3. **You can troubleshoot.** When something goes wrong (and in PKI, something always goes wrong), you'll know where to look because you built it yourself.
4. **You remember it.** Muscle memory from manual entry beats copy-paste amnesia every time.

You COULD copy/paste from this guide but you're only cheating yourself.

<br>

## What's Different from Keyfactor's Official Docs?

[Keyfactor's official documentation is plenty great](https://docs.keyfactor.com/ejbca-software/latest/installation), they have a lot of options — it's our reference throughout this lab. But we've adapted it for our specific use case:

| Official Docs | Our Lab |
|--------------|---------|
| Generic CA setup | PQC CAs created natively using OpenSSL lab knowledge |
| Multiple database options | MariaDB only (focused and clear) |
| Multiple app server versions | WildFly 35 only |
| Production-oriented | Learning-oriented with explanations |
| RSA/ECDSA examples | Post-quantum (ML-DSA) throughout |

We follow the same order and structure as the official docs, but every step includes the educational context and `SassyCorp`-specific configuration that makes this a learning experience rather than a deployment checklist.

<br>

## A Note on Post-Quantum Compatibility

Here's something important: EJBCA® uses Bouncy Castle as its cryptographic provider, and Bouncy Castle has been adding PQC algorithm support. EJBCA CE v9.3 includes Bouncy Castle libraries that support ML-DSA and other post-quantum algorithms.

When creating PQC CAs natively in EJBCA®, there are a few things to be aware of:

- **Bouncy Castle version conflicts** — WildFly ships its own Bouncy Castle, which can conflict with EJBCA's bundled version (Keyfactor's documentation and our lab solve for this in Module 04)
- **Signing vs. encryption key pairs** — PQC signing keys (ML-DSA) work great, but EJBCA requires the encryption key pair to still be RSA (per Keyfactor's documentation)
- **OCSP signing** — doesn't support PQC at this time; use RSA/EC for OCSP signing keys

We'll address each of these in detail as we encounter them. For now, just be aware that PQC + EJBCA is cutting edge territory and we'll be working through any compatibility considerations together. As the software revs, we'll keep updating this guide.

<br>

## Enough already, let's Go

Ready to build a modern, quantum-resistant Certificate Authority?

**Next Module:** [01 - Installation Prerequisites](01_ejbca_prerequisites.md)

<br>

## References

| Resource | URL |
|----------|-----|
| EJBCA® Community Edition Source | https://github.com/Keyfactor/ejbca-ce |
| EJBCA® Software Stack Documentation | https://docs.keyfactor.com/ejbca-software/latest/installation |
| OpenSSL PQC Lab (Prior Lab) | https://github.com/f5devcentral/openssl-pqc-stepbystep-lab/tree/main/fipsqs |
| WildFly 35 | https://www.wildfly.org/ |
| OpenJDK 21 | https://openjdk.org/projects/jdk/21/ |
| MariaDB | https://mariadb.org/ |
| NIST FIPS 204 (ML-DSA) | https://csrc.nist.gov/pubs/fips/204/final |
| Keyfactor EJBCA PQC Hybrid Tutorial | https://docs.keyfactor.com/ejbca/9.0/tutorial-create-pqc-hybrid-ca-chain |

---
