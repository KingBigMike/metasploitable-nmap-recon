# Port 23 — Telnet (Linux telnetd)

## Objective
Enumerate the Telnet service, assess encryption support, and test authentication strength via
brute-force.

## Recon
Nmap's Telnet NSE scripts (`telnet-encryption`, `telnet-brute`) revealed:

- **Service:** Linux telnetd
- **Encryption:** confirmed not supported — Telnet server does not support encryption
- **Brute-force result:** `user:user` — valid credentials found (2,023 guesses over 303 seconds)

## Vulnerability Research

**1. Telnet transmits everything in plaintext — CWE-319**
Not a bug or misconfiguration — the fundamental design of the Telnet protocol, dating back to 1969.
Telnet was built decades before encryption was a standard consideration for remote access protocols.
Every byte of a Telnet session — including the username and password typed at login — travels across
the network in cleartext.

This means:
- Anyone capturing traffic on the same network segment (ARP spoofing, a compromised switch, a rogue
  access point, or just being on a shared/insecure network) can trivially read login credentials and
  full session activity using a tool as simple as Wireshark
- There is no configuration fix for this — encryption isn't an option that can be toggled on. The only
  real fix is to stop using Telnet entirely

**2. Weak credentials compound the exposure**
Same `user:user` pair found on FTP and SSH — a strong signal that this is a reused weak credential
across multiple services on the same host, not an isolated one-off.

**3. No CVE ID — and that's the point**
Some of the most serious real-world findings in penetration testing aren't tied to a specific
vulnerability disclosure at all — they're protocol-level design decisions that were reasonable in 1969
and are indefensible today. Recognizing "insecure by design, no patch will ever fix this" as its own
category of finding is a mark of understanding, not a missing citation.

**Severity:** High — requires network positioning to actually intercept traffic (unlike, say, the
vsftpd backdoor which is remotely exploitable from anywhere), but the combination of zero encryption
plus confirmed-valid weak credentials makes this a realistic, low-effort compromise path in any shared
or untrusted network environment.

## Exploitation (lab-safe verification)
1. **Direct login:**
   ```bash
   telnet 192.168.0.156
   # login: user / password: user
   ```
2. **Traffic capture (the real point of this finding):** with Wireshark or `tcpdump` running on the
   attacking machine during that login, filter for `telnet` traffic and follow the TCP stream — the
   username and password will be visible in plaintext in the packet capture.

## Root Cause
- Use of an inherently insecure legacy protocol with no encryption capability
- Same weak, reused password pattern seen elsewhere on this host

## Remediation
- **Disable Telnet entirely** and replace it with SSH for all remote administrative access — a "retire
  the service" situation, not a "harden the config" one
- If Telnet must remain for legacy device compatibility, restrict it to an isolated, tightly firewalled
  management VLAN never exposed to general traffic, and never reuse credentials from any other service
- Address the underlying weak-password pattern account-wide rather than per-service, since this is
  clearly one root cause manifesting across three separate findings

## Tools Used
- Nmap (`-sV -sC`, Telnet NSE script family: `telnet-encryption`, `telnet-brute`)
- `telnet` client, Wireshark/tcpdump (cleartext credential capture demonstration)

## Reflections
Pairing this finding with the credential reuse pattern (`user:user` on FTP, SSH, and Telnet) gives the
overall assessment a strong closing insight: this VM wasn't compromised by six unrelated bugs, it was
compromised by one weak account that happened to work everywhere, made worse by services that couldn't
have protected it even if the password had been strong.
