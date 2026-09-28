# Port 21 — FTP (vsftpd 2.3.4)

## Objective
Enumerate and assess the FTP service running on the target Metasploitable VM, identify any known
vulnerabilities tied to the detected version, and validate exploitability where safe to do so in an
isolated lab.

## Recon
Nmap intensive scan (`-sV -sC` with NSE scripts `ftp-anon`, `ftp-syst`, `ftp-brute`,
`ftp-vsftpd-backdoor`) revealed:

- **Service:** vsftpd 2.3.4
- **Anonymous login allowed** (FTP code 230) — no credentials required to authenticate
- **Weak credentials found via brute force:** `user:user` (from 1,226 guesses over ~5 minutes)
- **Critical finding:** the `ftp-vsftpd-backdoor` NSE script confirmed the service is running the
  infamous backdoored version of vsftpd 2.3.4, and even executed a proof-of-concept command (`id`)
  returning `uid=0(root) gid=0(root)`

## Vulnerability Research — CVE-2011-2523

**Background:** In June 2011, the official vsftpd source archive hosted on the project's master site
was compromised by an attacker who inserted a backdoor into the source code before it was discovered
and pulled down (within roughly a 2-day window). Anyone who downloaded and compiled that specific
tainted archive got a backdoored binary — this wasn't a bug in vsftpd's actual code logic, it was a
**supply-chain compromise**.

**How the backdoor works:**
- The malicious code triggers when a client sends a username containing a smiley face — literally the
  string `:)` as part of the USER command
- When triggered, the backdoor opens a **listener on TCP port 6200** and binds `/bin/sh` to it — an
  unauthenticated root shell, no valid credentials needed at all
- Any attacker who can reach port 21 can trigger it and then connect to port 6200 to get a root shell
  directly

**Severity:** Critical (root-level unauthenticated remote code execution)
**CVSS:** 10.0 (v2)
**Disclosure date:** 2011-07-03

**References:**
- CVE-2011-2523
- Metasploit module: `exploit/unix/ftp/vsftpd_234_backdoor`
- Original disclosure: scarybeastsecurity.blogspot.com's writeup on the compromised archive

## Exploitation (lab-safe verification)
Since Nmap's NSE script already triggered and confirmed it, two paths work for verification:

1. **Cite the NSE result directly** — show the `id` output returning root as proof
2. **Manually reproduce with Metasploit:**
   ```
   msfconsole
   use exploit/unix/ftp/vsftpd_234_backdoor
   set RHOSTS 192.168.0.156
   run
   ```
   This should drop into a root shell on port 6200.

Separately worth noting: **anonymous login** and the **weak `user:user` credential** are each
independently exploitable findings even without the backdoor — anonymous access alone could allow
file read/write depending on FTP root permissions, and weak creds could be reused elsewhere
(credential stuffing angle) in a real assessment.

## Root Cause
Three stacked issues, from least to most severe:
1. **Anonymous FTP enabled** — misconfiguration, unnecessary attack surface
2. **Weak/default credentials** (`user:user`) — poor password policy
3. **Backdoored binary (CVE-2011-2523)** — supply-chain compromise baked into the software itself; no
   configuration would have prevented this, only patching/upgrading away from the compromised version

## Remediation
- Upgrade vsftpd to a version after 2.3.5 (only the one tainted 2.3.4 archive was ever backdoored)
- Disable anonymous FTP login unless there's a specific, controlled business need
  (`anonymous_enable=NO` in `vsftpd.conf`)
- Enforce strong password policy, or migrate to SFTP over SSH entirely
- Since control and data connections are plaintext, plain FTP should generally be replaced with FTPS
  or SFTP in any real environment
- Network-level: restrict port 21 access via firewall/ACL to only trusted hosts if FTP must remain in
  use

## Tools Used
- Nmap (`-sV -sC`, `ftp-anon`, `ftp-syst`, `ftp-brute`, `ftp-vsftpd-backdoor` NSE scripts)
- Metasploit Framework (`vsftpd_234_backdoor` module, for manual reproduction)

## Reflections
This is a real historical supply-chain attack that made it into a widely-used package for two days in
2011, and it's still used today as the textbook example of why software provenance and checksum
verification matter. Not just "a bug" — unauthenticated root access via a trojanized upstream package.
