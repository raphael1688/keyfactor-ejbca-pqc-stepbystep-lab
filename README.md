# EJBCA® Post-Quantum Cryptography Lab

> **🎯 tl;dr - Our Goal:** Build a managed PQC Certificate Authorities in EJBCA®, informed by everything we learned in the OpenSSL lab — same algorithms, same organizational identity, now with a modern PKI platform underneath.

Let's do more post-quantum cryptography! In our [NIST FIPS PQC Lab](https://github.com/f5devcentral/openssl-pqc-stepbystep-lab/tree/main/fipsqs), we built a fully functional quantum-resistant Certificate Authority using OpenSSL 3.5.x — complete with an ML-DSA-87 Root CA, an ML-DSA-65 Intermediate CA, end-entity certificates, and revocation infrastructure. Fun stuff.

Now comes a real-world question: **how do you manage that CA infrastructure at scale?**

OpenSSL is an incredible tool for building and understanding PKI, but operating a production CA requires lifecycle management, role-based access control, audit logging, OCSP responders, CRL distribution, and management that doesn't involve memorizing `openssl ca` flags. That's where PKI management solutions like [EJBCA® Community Edition](https://www.ejbca.org/) enter our learning path.

This lab guide walks you through building **new, enterprise style PQC Certificate Authorities** natively inside **Keyfactor's EJBCA® Community Edition v9.3** — an open-source, PKI management platform. Apply knowledge from the prior learnings in the OpenSSL lab (algorithm choices, key sizes, subject DNs, chain-of-trust design) to create a certificate authority with the same SassyCorp identity, now backed by a proper PQC PKI solution. 

By the end, you'll have quantum-resistant Root and Intermediate CAs managed through EJBCA® with certificate lifecycle management, audit logging, and a web-based admin interface — and you'll be more aligned to where we need to end up in order to start practicing cryptographic agility at scale; crypto agility for the marketing people. It's going to be a hotter topic as we ebb and flow with the quantum resistant crypto-secure future being mandated by pretty much everyone at this point. So suck up the marketing terms and let's dive in.

## What You'll Build

An entire PKI stack from the ground up:

- 🌈 **MariaDB** database backend with proper character sets, binary log format, and both socket and TCP -connectivity (because WildFly uses TCP and this will bite you exactly once before you learn why)
- 🌈 **WildFly 35.0.1.Final** application server with 3-port TLS separation, Elytron security configuration, credential stores, and a JDBC datasource — all configured through JBoss CLI commands
- 🌈 **EJBCA® Community Edition v9.3** compiled from source with Apache Ant, deployed as an EAR, initialized with a Management CA, and serving a mutual TLS admin interface
- 🌈 **ML-DSA-87 Root CA** created natively inside EJBCA's crypto token framework — quantum-resistant, NIST FIPS 204 compliant, same SassyCorp identity from the OpenSSL lab
- 🌈 **ML-DSA-65 Intermediate CA** signed by the Root CA, completing a post-quantum chain of trust that you can verify with OpenSSL on the command line

When you're done, you'll have a web-based admin UI, certificate lifecycle management, audit logging, OCSP, CRL distribution, and a REST API — all backed by post-quantum cryptography. Disco.

## Why This Matters (The Short and Sassy Version)

RSA and ECC have an "expiration" date. We don't know exactly when, but NIST has already finalized the replacements and everyone from the NSA, CCCS, to the BSI is publishing migration timelines. The transition is happening.

The problem is that reading about PQC migration and actually doing it are two completely different things. There's a gap between "I understand lattice-based cryptography conceptually" and "I can stand up a CA that signs certificates with ML-DSA-87 and I know which Bouncy Castle library is going to ruin my afternoon." This lab closes that gap.

By the end, you'll have hands-on experience with the exact workflow organizations will need to follow: create crypto tokens with PQC key pairs, initialize certificate authorities with quantum-resistant signing algorithms, build a chain of trust, and manage it all through an enterprise platform. When someone in your organization asks "can we actually do this?" you'll be the one who already has.

## What is ML-DSA?

ML-DSA (Module-Lattice-Based Digital Signature Algorithm) is the NIST-standardized post-quantum signature scheme from [FIPS 204](https://csrc.nist.gov/pubs/fips/204/final). It replaces RSA and ECDSA for digital signatures in a post-quantum world. It comes in three flavors:

| Parameter Set | Security Level | What We Use It For |
|---------------|---------------|---------------------|
| **ML-DSA-65** | NIST Level 3 | Intermediate CA signing key |
| **ML-DSA-87** | NIST Level 5 | Root CA signing key |

ML-DSA-87 for the root (maximum security for the trust anchor), ML-DSA-65 for the intermediate (strong security, better performance for day-to-day signing). Same choices we made in the OpenSSL lab, now running inside PKI platform infrastructure.

## Creating PQC Certificates for Your Environment

Once the lab is built, you have a fully operational PQC certificate authority. The Intermediate CA can sign end-entity certificates through EJBCA's enrollment interface, REST API, or CLI. Want to issue a quantum secure server certificate? Create a certificate profile, set the signature algorithm, and enroll. Want to test your application's PQC certificate handling? Issue certificates and export them. The infrastructure is there — the busy part is building it.



## Lab Modules

| Module | Description |
|--------|-------------|
| [00 - Introduction](/ejbca-pqc/00_ejbca_migration_intro.md.md) | Architecture overview, objectives, and why native creation |
| [01 - Prerequisites](/ejbca-pqc/01_ejbca_prerequisites.md.md) | OpenJDK 21, Apache Ant, system dependencies |
| [02 - Configurations](/ejbca-pqc/02_ejbca_configurations.md.md) | EJBCA configuration files and properties |
| [03 - Database](/ejbca-pqc/03_ejbca_database.md.md) | MariaDB setup, database creation, user privileges |
| [04 - WildFly](/ejbca-pqc/04_ejbca_wildfly.md.md) | WildFly 35 setup, datasource, 3-port TLS configuration |
| [05 - Deploy](/ejbca-pqc/05_ejbca_deploy.md.md) | Clone EJBCA CE v9.3, build with Ant, deploy |
| [06 - Install](/ejbca-pqc/06_ejbca_install.md.md) | Management CA, keystores, TLS initialization |
| [07 - Create PQC CAs](/ejbca-pqc/07_ejbca_create_pqc_ca.md.md) | Native ML-DSA Root and Intermediate CA creation |
| [08 - Finalize](/ejbca-pqc/08_ejbca_finalize.md.md) | Browser setup, verification, and testing |
| [09 - Reference](/ejbca-pqc/09_ejbca_reference.md.md) | Deployment reference, CLI cheat sheets, and troubleshooting |

## Prerequisites

You should seriously complete [Part 1 — the OpenSSL PQC Lab](https://github.com/f5devcentral/openssl-pqc-stepbystep-lab/tree/main/fipsqs) first. This lab builds on that knowledge and references the CA infrastructure you created there. If you skipped it, go back. We'll wait.

## Get Started

**[Start with Module 00 -->](/ejbca-pqc/00_ejbca_migration_intro.md)**

<br>

## References

| Resource | URL |
|----------|-----|
| NIST FIPS 204 (ML-DSA) | https://csrc.nist.gov/pubs/fips/204/final |
| EJBCA Community Edition | https://github.com/Keyfactor/ejbca-ce |
| EJBCA Installation Docs | https://docs.keyfactor.com/ejbca-software/latest/installation |
| Keyfactor PQC CA Tutorial | https://docs.keyfactor.com/ejbca/9.3.2/tutorial-create-pqc-hybrid-ca-chain |
| OpenSSL PQC Lab (Part 1) | https://github.com/f5devcentral/openssl-pqc-stepbystep-lab/tree/main/fipsqs |

---

*This lab is part of the [Post-Quantum Cryptography Step-by-Step Lab](https://github.com/f5devcentral/openssl-pqc-stepbystep-lab) series. For educational and internal testing purposes. Production deployments should use HSMs, air-gapped Root CAs, and follow organizational security policies.*