# Metasploitable2 — Nmap Intensive Scan: 6-Port Vulnerability Assessment

## Overview
This report documents an intensive Nmap scan (`-sV -sC`) against a Metasploitable2 VM (192.168.0.156),
followed by targeted vulnerability research on 6 open ports. The goal was to move beyond raw scan output
and understand *why* each finding matters — distinguishing coded vulnerabilities (CVEs) from
misconfigurations from protocol-level design flaws.

## Ports Assessed
| Port | Service | Severity | Core Finding |
|------|---------|----------|--------------|
| 21   | FTP (vsftpd 2.3.4) | Critical | Backdoored binary — CVE-2011-2523, unauthenticated root shell |
| 80   | HTTP (Apache 2.2.8) | Medium–High | EOL software stack, TRACE enabled, exposed admin panels |
| 3306 | MySQL 5.0.51a | Critical | Empty root password — full unauthenticated DB access |
| 25   | SMTP (Postfix) | Low | VRFY user enumeration; open relay test passed (not vulnerable) |
| 22   | SSH (OpenSSH 4.7p1) | Medium–High | Weak credentials + legacy ciphers (RC4, MD5, DH-group1) |
| 23   | Telnet | High | No encryption possible by design + weak credentials |

## The Pattern That Ties It Together
The same weak credential pair — **`user:user`** — was valid on **FTP, SSH, and Telnet**. This is the
single most important finding across the whole assessment: it wasn't six unrelated bugs, it was one
weak, reused account exposed across multiple services, several of which (Telnet especially) couldn't
have protected that credential even if it had been strong, since they transmit it in plaintext.

## Categorizing the Findings
Not every vulnerability fits the same mold — deliberately grouping them this way for the report:
- **Supply-chain compromise:** vsftpd 2.3.4 backdoor (CVE-2011-2523)
- **Pure misconfiguration:** MySQL empty root password
- **Systemic/EOL software risk:** Apache 2.2.8 + PHP 5.2.4 stack
- **Protocol-level design flaw (no patch possible):** Telnet's lack of encryption
- **Weak credential hygiene:** SSH, FTP, Telnet all accepting `user:user`
- **Negative result (tested, not vulnerable):** SMTP open relay check

## Methodology
- **Recon:** `nmap -sV -sC -A <target>` plus targeted NSE script families per service
- **Verification:** manual reproduction where safe (Metasploit module for vsftpd, direct CLI login for
  MySQL/SSH/Telnet, packet capture for Telnet cleartext demonstration)
- **Documentation:** Objective → Recon → Exploitation → Root Cause → Remediation → Tools → Reflections,
  applied consistently across all six writeups

## Full Writeups
- [Port 21 — FTP](./findings/port-21-ftp.md)
- [Port 80 — HTTP](./findings/port-80-http.md)
- [Port 3306 — MySQL](./findings/port-3306-mysql.md)
- [Port 25 — SMTP](./findings/port-25-smtp.md)
- [Port 22 — SSH](./findings/port-22-ssh.md)
- [Port 23 — Telnet](./findings/port-23-telnet.md)

## Disclaimer
All testing was performed against an intentionally vulnerable Metasploitable2 VM in an isolated lab
environment. No production systems or third parties were involved.
