# Port 25 — SMTP (Postfix smtpd)

## Objective
Enumerate the SMTP service, test for common mail server misconfigurations (open relay, user
enumeration), and check for known vulnerabilities affecting the detected service.

## Recon
Nmap's SMTP NSE scripts (`smtp-commands`, `smtp-open-relay`, `smtp-enum-users`,
`smtp-vuln-cve2010-4344`) returned:

- **Service:** Postfix smtpd (no specific version banner grabbed in this scan)
- **Supported SMTP commands/extensions:** `PIPELINING`, `SIZE 10240000`, **`VRFY`**, `ETRN`,
  **`STARTTLS`**, `ENHANCEDSTATUSCODES`, `8BITMIME`, `DSN`
- **Open relay test:** failed all tests — **server is NOT an open relay** (good result)
- **CVE-2010-4344 check (Exim-specific heap overflow):** not applicable — confirmed NOT vulnerable,
  since this is Postfix, not Exim
- **`smtp-enum-users`:** inconclusive — the `RCPT` enumeration method returned an unhandled status code

## Vulnerability Research

**1. VRFY command enabled — SMTP User Enumeration (CWE-200)**
The `VRFY` command lets a client ask the mail server to verify whether a given username/mailbox exists,
without sending a message. An attacker can enumerate valid system/mail users:
```bash
telnet 192.168.0.156 25
VRFY root
VRFY admin
VRFY msfadmin
```
The server's response differs for valid vs. invalid users, letting an attacker build a list of real
accounts for password-guessing or social engineering. A long-standing, well-documented protocol-level
information disclosure issue; disabling `VRFY` is standard hardening.

**2. STARTTLS present but plaintext by default**
`STARTTLS` is supported, meaning encrypted SMTP sessions are *possible*, but nothing confirms it's
*enforced*. Unless required before authentication/mail transfer, communications could be sent in
plaintext by default.

**3. No open relay — a genuinely good finding**
Worth including precisely because it's a negative result: an open relay would let any outside sender
bounce mail through this server (classic spam/spoofing abuse), and the scan confirms that's not the
case here.

**4. Postfix vs. legacy sendmail/Exim-class bugs**
Because this is Postfix (not Sendmail or Exim), several historically severe mail-server CVEs — like
CVE-2010-4344, explicitly checked and ruled out — simply don't apply here.

**Severity:** Low — the main actionable item (VRFY enumeration) is an information-disclosure/
reconnaissance aid rather than a direct compromise path, but it's real and worth reporting.

## Exploitation (lab-safe verification)
```bash
nc 192.168.0.156 25
VRFY msfadmin
VRFY nonexistentuser12345
```
Compare the responses — a differing reply (e.g., "250" for valid vs "550" for invalid) confirms the
enumeration is practically exploitable, not just theoretically present.

## Root Cause
- `VRFY` left enabled in the Postfix configuration (default in many installs unless explicitly
  hardened)
- No enforced encryption policy for STARTTLS

## Remediation
- Disable `VRFY`: set `disable_vrfy_command = yes` in `/etc/postfix/main.cf`
- Enforce STARTTLS for all connections where feasible (`smtpd_tls_security_level = encrypt`)
- If SMTP AUTH is used anywhere, ensure it's never permitted over an unencrypted connection
  (`smtpd_tls_auth_only = yes`)
- Monitor for and rate-limit repeated `VRFY`/`RCPT` probing attempts as a detection signal

## Tools Used
- Nmap (`-sV -sC`, SMTP NSE script family: `smtp-commands`, `smtp-open-relay`, `smtp-enum-users`,
  `smtp-vuln-cve2010-4344`)
- `nc`/`telnet` (manual VRFY verification)

## Reflections
Not every port yields a critical CVE, and that's fine to say plainly. The real skill demonstrated here
is knowing what to test for (open relay, enumeration, known CVEs for the specific mail daemon in use)
and being able to say confidently "this was checked and ruled out" alongside "this was checked and
confirmed." A portfolio full of only critical findings can look less credible than one that shows
calibrated, methodical testing.
