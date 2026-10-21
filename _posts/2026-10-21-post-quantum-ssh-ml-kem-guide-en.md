---
title: "Post-Quantum SSH: Securing Servers for the Quantum Era & Eliminating OpenSSH Warnings"
date: 2026-10-21 16:30:00 +0300
categories: [vps, security]
tags: [ssh, post-quantum, openssh, security, linux, cryptography, vps, devops]
image:
  path: /assets/img/posts/post-quantum-ssh-cover.jpg
  alt: Post-Quantum OpenSSH Cryptography and ML-KEM Algorithms
lang: en
hidden: true
alt_lang_url: /posts/post-quantum-ssh-ml-kem-guide/
---

[🇺🇦 Читайте цю статтю українською](/posts/post-quantum-ssh-ml-kem-guide/)

---

If you have recently updated your developer tooling or executed a `git push` to a modern repository, you may have encountered an unfamiliar warning in your terminal:

```text
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
```

It is natural to wonder: *“Wait, are quantum computers capable of breaking encryption even available? Why is OpenSSH sounding the alarm right now when my standard Ed25519 or RSA keys work flawlessly?”*

The answer lies in an adversarial espionage strategy known as **SNDL (Store Now, Decrypt Later)** and the global migration toward standardized quantum-resistant cryptography (NIST FIPS 203 / ML-KEM).

Today, we explore why legacy key exchanges are aging out faster than expected, how hybrid post-quantum cryptography works inside OpenSSH, and how to configure your VPS and local workstation to eliminate these warnings while protecting your infrastructure data for decades to come.

![Post-Quantum SSH Cryptography Infographic](/assets/img/posts/post-quantum-ssh-infographic.jpg)
_Fig. 1. Architectural comparison: Classical SSH (vulnerable to Store Now, Decrypt Later attacks) vs. Hybrid Post-Quantum Key Exchange (ML-KEM-768 + X25519)_

---

## The "Store Now, Decrypt Later" Threat: Why Act Today?

All conventional asymmetric cryptographic systems (RSA, classical Diffie-Hellman, and elliptic curves like ECDH / X25519) rely on two fundamental mathematical problems:
1. The computational hardness of factoring large prime numbers (RSA).
2. The discrete logarithm problem over finite fields or elliptic curve groups (ECDH / Ed25519).

A classical supercomputer would require billions of years to brute-force a modern 256-bit elliptic curve key. However, in 1994, mathematician Peter Shor formulated **Shor’s Algorithm**, which proves that a Cryptographically Relevant Quantum Computer (CRQC) can solve these mathematical problems in **mere minutes**.

**The immediate danger:**  
Nation-state intelligence agencies and sophisticated threat actors are passively intercepting and stockpiling petabytes of encrypted public internet traffic (SSH, TLS, VPN sessions) in bulk storage facilities **right now**. While they cannot read this data today, they are playing the long game: once scalable quantum hardware emerges in 5 to 10 years, their vast historical archives will be decrypted retroactively.

If your infrastructure credentials, internal tunnels, and system configurations need to remain secret beyond the next few years, you must adopt post-quantum key agreement today.

---

## The Solution: NIST Standardization & ML-KEM (Kyber)

In August 2024, the U.S. National Institute of Standards and Technology (NIST) officially released its first final post-quantum cryptography standards. The primary standard for key encapsulation is:

* **FIPS 203 (ML-KEM):** Module-Lattice-Based Key-Encapsulation Mechanism (derived from **CRYSTALS-Kyber**).

Unlike classical cryptography, ML-KEM is founded on the geometry of high-dimensional Euclidean lattices (*Lattice-based cryptography*), specifically the hardness of the Module Learning With Errors (M-LWE) problem. There is no known quantum algorithm capable of solving these lattice problems in polynomial time.

---

## Why OpenSSH Implements a Hybrid Approach

The OpenSSH development team took a pragmatic, highly resilient architectural route: they did not simply discard classical X25519 in favor of pure lattice cryptography. Instead, they standardized **Hybrid Key Exchange**:

1. **`sntrup761x25519-sha512@openssh.com`** (combining Streamlined NTRU Prime with X25519).
2. **`mlkem768x25519-sha256`** (combining the NIST FIPS 203 standard ML-KEM-768 with X25519).

### How the Hybrid Shield Operates
During the initial handshake, the client and server negotiate **two independent shared secrets simultaneously**:
* One generated via classical X25519.
* One generated via post-quantum ML-KEM.

The final symmetric session key is derived by hashing both shared secrets together.
* If a quantum computer breaks X25519 in the future, ML-KEM preserves the session’s confidentiality.
* If a hidden mathematical vulnerability is ever uncovered in the newer lattice mathematics of ML-KEM, the battle-tested X25519 component prevents compromise.

An attacker would have to crack **both** independent mathematical problems concurrently to breach the tunnel.

---

## Step 1. Inspect Your OpenSSH Version and Supported KEX

Support for post-quantum algorithms was introduced incrementally:
* Starting with **OpenSSH 9.0**, hybrid `sntrup761x25519-sha512` was enabled by default.
* Starting with **OpenSSH 9.8 and 9.9**, the official NIST FIPS `mlkem768x25519-sha256` was introduced, alongside active telemetry warnings for legacy negotiations.

### 1. Check your client version:
```bash
ssh -V
```

### 2. Query supported key exchange algorithms:
```bash
ssh -Q kex | grep -E "sntrup|mlkem"
```

If your terminal lists:
```text
sntrup761x25519-sha512
sntrup761x25519-sha512@openssh.com
mlkem768x25519-sha256
```
Your OpenSSH binaries support modern post-quantum cryptography.

---

## Step 2. Verify Your Active Session Encryption

To verify which key exchange algorithm your client and server actually agree upon, initiate a connection with verbose logging (`-v`):

```bash
ssh -v -p YOUR_PORT adminuser@YOUR_SERVER_IP
```

Locate the key exchange confirmation block in the connection output:

```text
debug1: kex: algorithm: sntrup761x25519-sha512@openssh.com
debug1: kex: host key algorithm: ssh-ed25519
debug1: kex: client->server cipher: chacha20-poly1305@openssh.com
```

If the `kex: algorithm` row specifies an algorithm containing **`sntrup`** or **`mlkem`**, your connection is hardened against quantum eavesdropping.

If it falls back to `curve25519-sha256` or `ecdh-sha2-nistp256`, your server or client configuration needs an update.

---

## Step 3. Server-Side Hardening: Prioritize Post-Quantum KEX

To ensure your SSH daemon strictly prioritizes post-quantum algorithms for all incoming connections, create a dedicated drop-in configuration:

```bash
sudo nano /etc/ssh/sshd_config.d/post-quantum.conf
```

Add the following preference list:

```text
# Prioritize hybrid post-quantum key exchange algorithms
KexAlgorithms sntrup761x25519-sha512@openssh.com,mlkem768x25519-sha256,curve25519-sha256,curve25519-sha256@libssh.org,diffie-hellman-group16-sha512,diffie-hellman-group18-sha512
```

Validate the daemon configuration syntax before restarting:

```bash
sudo sshd -t
```

If the validator returns cleanly, restart the SSH service:

* **Ubuntu / Debian:** `sudo systemctl restart ssh`
* **AlmaLinux / Rocky Linux:** `sudo systemctl restart sshd`

---

## Step 4. Client-Side Tuning: Banish Warnings Forever

To ensure your local workstation (macOS, Linux, or Windows PowerShell) always requests quantum-safe key exchange, configure your local client profile:

Open `~/.ssh/config`:

```bash
nano ~/.ssh/config
```

Add the global directive:

```text
Host *
    KexAlgorithms sntrup761x25519-sha512@openssh.com,mlkem768x25519-sha256,curve25519-sha256
```

Your workstation will now prioritize `sntrup761` and `mlkem768`. As long as the remote endpoint supports either algorithm, the connection warning will never appear again.

---

## What About Authentication Keys? Do We Need to Replace Ed25519?

It is vital to distinguish between data-in-transit confidentiality and authentication:

| Connection Phase | What It Protects | Vulnerable to "Store Now, Decrypt Later"? | Action Required |
| :--- | :--- | :--- | :--- |
| **Key Exchange (KEX)** | Establishes ephemeral session encryption keys | **YES.** Intercepted traffic can be decrypted later | **Upgrade immediately to ML-KEM / sntrup761** |
| **Authentication (`id_ed25519`)** | Verifies identity at the moment of login | **NO.** Breaking an auth key in 10 years cannot decrypt past traffic | **Ed25519 remains highly secure and recommended** |

Because historical traffic capture threatens session secrecy rather than past identity proofs, OpenSSH focused first on standardizing **Key Exchange (KEX)**. You can safely keep your existing `ssh-ed25519` keypairs without disruption.

---

## Summary

Post-quantum cryptography is no longer theoretical research; it is an active operational security standard across modern Linux systems.

By checking your OpenSSH packages and registering the hybrid `KexAlgorithms` directive:
1. You completely neutralize the threat of "Store Now, Decrypt Later" surveillance.
2. You eradicate annoying terminal warnings during Git pushes and administrative logins.
3. Your server infrastructure complies with the latest NIST FIPS 203 standards.

Audit your servers today using `ssh -Q kex`—quantum-safe infrastructure starts now!
