# Port 80 — HTTP (Apache httpd 2.2.8, Ubuntu, mod_dav, PHP 5.2.4)

## Objective
Enumerate the web service running on port 80, identify outdated software versions, exposed
applications, and misconfigurations, and research the associated security risks.

## Recon
Nmap's HTTP NSE scripts revealed a surprising amount hosted on this single port:

- **Server:** Apache/2.2.8 (Ubuntu) with **mod_dav/2** enabled
- **Backend:** PHP/5.2.4-2ubuntu5.10 (confirmed via `X-Powered-By` header)
- **Hosted applications discovered via directory enumeration (`http-enum`, `http-sitemap-generator`):**
  - `/dvwa/` — Damn Vulnerable Web Application
  - `/mutillidae/` — OWASP Mutillidae II
  - `/phpMyAdmin/` — database admin panel
  - `/twiki/`, `/tikiwiki/` — wiki software
  - `/phpinfo.php` — exposed PHP configuration info
  - `/doc/`, `/icons/`, `/index/` — directory listing enabled
- **HTTP TRACE method enabled** (`http-trace`)
- **Slowloris DoS flagged** (`http-slowloris-check`) — CVE-2007-6750, likely vulnerable
- **CSRF-vulnerable forms detected** across DVWA and Mutillidae login/user-info pages (`http-csrf`)
- **`http-auth-finder`** located login forms at `/dvwa/`, `/phpMyAdmin/`, and Mutillidae's user-info page

## Vulnerability Research

**1. Outdated Apache 2.2.8 + PHP 5.2.4 (End-of-Life software stack)**
Both are severely outdated: Apache 2.2.x was EOL in December 2017, and PHP 5.2.4 (released 2007) has
dozens of publicly known CVEs accumulated over its lifetime. Running EOL software means no vendor
patches exist for anything discovered against it going forward — a systemic risk, not a single CVE.

**2. HTTP TRACE enabled — Cross-Site Tracing (XST) risk**
The `TRACE` method reflects the exact request back to the client, including headers. Combined with XSS
on the same origin, this can be abused to steal HTTP-only cookies that would otherwise be inaccessible
to JavaScript — the core idea behind Cross-Site Tracing (XST).

**3. Directory listing / information disclosure**
`/doc/`, `/icons/`, and `/index/` returning directory listings, plus an exposed `phpinfo.php`, hand an
attacker a low-effort map of the filesystem and full PHP build configuration.

**4. Intentionally vulnerable applications hosted directly**
DVWA and Mutillidae II are training platforms deliberately built with vulnerabilities (SQLi, XSS,
command injection, file inclusion, CSRF, broken auth, etc.) — expected on Metasploitable by design, but
each represents an enormous, separate attack surface deserving its own dedicated assessment.

**5. phpMyAdmin exposed with no apparent access restriction**
Older phpMyAdmin versions have had real critical CVEs historically — authentication bypass and remote
code execution flaws in various point releases. Exposing an unrestricted admin panel to a database
front-end is a high-value target regardless of exact version.

**6. WebDAV (mod_dav) enabled**
`DAV/2` in the server header confirms WebDAV is active. If write access via `PUT`/`MOVE` methods is
misconfigured, this is a classic path to uploading a webshell directly onto the server.

## Root Cause
A combination of:
- Deliberately outdated/EOL software (by design, since this is Metasploitable's purpose)
- Insecure defaults never hardened (TRACE enabled, directory listing on, phpinfo.php left in place)
- Sensitive admin interfaces (phpMyAdmin) exposed without network-level restriction or additional auth

## Remediation
- Upgrade Apache and PHP to actively supported versions
- Disable the `TRACE` method (`TraceEnable Off` in Apache config)
- Disable directory listing (`Options -Indexes`)
- Remove `phpinfo.php` and any other diagnostic files from production-reachable paths
- Restrict access to admin panels like phpMyAdmin via IP allowlisting, VPN, or an additional auth layer
- Review WebDAV configuration; disable `PUT`/`DELETE` methods unless explicitly required
- Implement CSRF tokens on all state-changing forms

## Tools Used
- Nmap (`-sV -sC`, HTTP NSE script family: `http-enum`, `http-trace`, `http-csrf`,
  `http-slowloris-check`, `http-auth-finder`, `http-sitemap-generator`)

## Reflections
Port 80 here is really a "meta-finding" rather than one CVE — the real lesson is that a single
misconfigured web server can end up hosting a whole stack of independent risks at once (EOL software +
insecure defaults + exposed admin tools + deliberately vulnerable apps layered on top). Differentiating
between "known CVE" and "systemic misconfiguration risk" is what separates a scanner-output copy-paste
from an actual analyst's report.

*(Note: port 8180 running Apache Tomcat is a separate HTTP service with its own findings, including
exposed default `tomcat:tomcat` credentials on the manager interface — not covered in this six-port
set.)*
