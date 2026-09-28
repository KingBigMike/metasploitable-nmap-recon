# Port 22 — SSH (OpenSSH 4.7p1 Debian 8ubuntu1, protocol 2.0)

## Objective
Enumerate the SSH service, assess authentication strength and supported cryptographic algorithms, and
identify any known vulnerabilities tied to this specific OpenSSH version.

## Recon
Nmap's SSH NSE scripts (`ssh-brute`, `ssh-auth-methods`, `ssh-publickey-acceptance`, `ssh2-enum-algos`,
`ssh-hostkey`) revealed:

- **Version:** OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0) — released 2007, long past end-of-support
- **Authentication methods supported:** `publickey`, `password`
- **Public key acceptance:** none accepted (password auth is effectively the only usable path)
- **Brute-force result:** `user:user` — valid credentials found (971 guesses over 301 seconds)
- **Weak/legacy algorithms exposed via `ssh2-enum-algos`:**
  - Key exchange includes `diffie-hellman-group1-sha1` — a weak, deprecated KEX algorithm
  - Encryption includes `arcfour`, `arcfour128`, `arcfour256` (RC4-based, cryptographically broken),
    `3des-cbc`, `blowfish-cbc`, `cast128-cbc` — all outdated ciphers no longer recommended
  - MAC algorithms include `hmac-md5`, `hmac-md5-96` — MD5-based, weak by modern standards
  - Host keys: a 1024-bit DSA key and a 2048-bit RSA key — 1024-bit DSA is considered weak

## Vulnerability Research

**1. Weak/reused credentials (`user:user`)**
Same pattern as the FTP finding — a simple, guessable username/password pair grants direct shell
access. Not a code vulnerability — a credential hygiene failure — but the impact is just as real: an
interactive shell on the box.

**2. OpenSSH 4.7p1 — known historical CVEs**
- **CVE-2008-5161** — under certain configurations (specifically CBC-mode ciphers, several of which
  are exposed here — `aes128-cbc`, `3des-cbc`, `blowfish-cbc`), an attacker positioned on the network
  could potentially recover limited plaintext from an SSH session due to a weakness in CBC-mode error
  handling. Partial information-disclosure rather than full compromise, but directly relevant given the
  CBC ciphers confirmed enabled here.
- Various other minor bugs were patched in the years following 4.7p1's release.

**3. Legacy cipher/KEX/MAC support (CWE-327)**
None individually a "the box gets popped" vulnerability, but together a real weakened security posture:
`diffie-hellman-group1-sha1` uses a 768-bit modulus too small for modern margins; RC4 has known
cryptographic biases; MD5-based MACs are collision-prone and deprecated.

**4. Password authentication enabled at all**
With `publickey` offered but no keys actually accepted, password auth is the only real path in — which
is exactly what made the brute-force attack work.

**Severity:** Medium-High — no unauthenticated RCE like the FTP backdoor, but weak credentials plus
password-only practical auth plus legacy crypto stacks up to a realistic compromise path.

## Exploitation (lab-safe verification)
```bash
ssh user@192.168.0.156
# password: user
```
Confirms the brute-forced credentials work and demonstrates a real shell foothold.

## Root Cause
- Weak, guessable password set for a valid system account
- Password authentication left enabled without rate-limiting or lockout
- Server left on default weak cipher/KEX/MAC configuration inherited from an old OpenSSH release,
  never hardened

## Remediation
- Enforce strong password policy or disable password authentication entirely in favor of key-only auth
  (`PasswordAuthentication no` in `sshd_config`)
- Upgrade OpenSSH to a current, supported release
- Explicitly disable weak algorithms in `sshd_config`: remove `diffie-hellman-group1-sha1` from
  `KexAlgorithms`, remove `arcfour*`/`cbc`-mode ciphers from `Ciphers`, remove `hmac-md5*` from `MACs`
- Rotate/regenerate host keys using stronger key types (Ed25519 preferred, or RSA ≥ 3072-bit); retire
  the 1024-bit DSA key entirely
- Implement rate-limiting/fail2ban-style lockout on repeated failed auth attempts
- Restrict SSH access via firewall to known source IPs or a VPN/bastion where feasible

## Tools Used
- Nmap (`-sV -sC`, SSH NSE script family: `ssh-brute`, `ssh-auth-methods`,
  `ssh-publickey-acceptance`, `ssh2-enum-algos`, `ssh-hostkey`)
- `ssh` client (manual login verification)

## Reflections
No two ports in this assessment failed for exactly the same reason. This one combines weak credentials
with legacy cryptography — a different failure mode than the supply-chain backdoor on FTP or the pure
misconfiguration on MySQL, which is itself worth calling out explicitly in a summary report.
