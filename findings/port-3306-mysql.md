# Port 3306 — MySQL (5.0.51a-3ubuntu5)

## Objective
Enumerate the MySQL database service, assess authentication controls, and identify exposed databases
and credential weaknesses.

## Recon
Nmap's MySQL NSE script suite (`mysql-empty-password`, `mysql-brute`, `mysql-enum`, `mysql-databases`,
`mysql-users`, `mysql-info`, `mysql-variables`) returned:

- **Version:** MySQL 5.0.51a-3ubuntu5 (released 2008 — long EOL)
- **Critical finding:** `root` account has an **empty password**
- **Brute-force confirmed valid credentials:**
  - `root:<empty>`
  - `guest:<empty>`
  (13,685 guesses run in 300 seconds — both accounts came back valid almost immediately)
- **Enumerated user accounts:** `debian-sys-maint`, `guest`, `root`
- **Databases exposed:** `information_schema`, `dvwa`, `metasploit`, `mysql`, `owasp10`, `tikiwiki`,
  `tikiwiki195`
- **Notable config exposure via `mysql-variables`:**
  - `secure_auth: OFF` — allows older, weaker authentication handshake
  - `local_infile: ON` — permits `LOAD DATA LOCAL INFILE`, which (combined with SQL injection elsewhere)
    can be abused for local file read/information disclosure
  - `have_ssl: YES` — SSL is *supported* but nothing indicates it's being *enforced*

## Vulnerability Research

**1. Empty root password — CWE-521 / CWE-284**
Not a single CVE — a fundamental authentication failure. An empty password on the `root` MySQL account
means anyone who can reach port 3306 has full administrative control: read/write/delete on every
database, plus the ability to create new users, grant privileges, or use `LOAD DATA`/`INTO OUTFILE`
primitives (where filesystem permissions allow) to interact with the underlying host filesystem.

**2. `mysql` system database exposure**
Direct access to the internal MySQL account/privilege table means an attacker could potentially read
password hashes for all other database users directly from `mysql.user`, useful for lateral movement
or persistence planning.

**3. Legacy authentication protocol (`secure_auth: OFF`)**
Older MySQL auth handshake weaknesses remain possible with this setting off — a defense-in-depth
finding rather than an active exploit path on its own.

**4. Data exposure via visible databases**
App-specific databases (`dvwa`, `owasp10`, `tikiwiki`) confirm these applications store their data
directly in this exposed instance — a compromise of MySQL is effectively a compromise of every hosted
application's data at once.

**Severity:** Critical — root-level unauthenticated database access is functionally equivalent to full
data compromise, and depending on host configuration, can be a stepping stone to OS-level command
execution.

## Exploitation (lab-safe verification)
```bash
mysql -h 192.168.0.156 -u root
```
No password prompt needed. From there:
```sql
SHOW DATABASES;
USE dvwa;
SHOW TABLES;
SELECT * FROM users;
```
This demonstrates direct read access to application data without ever touching the web application
itself.

## Root Cause
- MySQL installed with no root password set and never hardened post-install
- No network-level restriction (firewall/bind-address) limiting who can reach port 3306
- Legacy MySQL version with weaker default security posture than modern releases

## Remediation
- Set a strong root password immediately: `mysql_secure_installation`
- Remove/rename the `guest` account or any other blank-password account
- Bind MySQL to `localhost`/internal interfaces only (`bind-address = 127.0.0.1`) unless remote DB
  access is explicitly required, restricted via firewall to known application server IPs
- Disable `local_infile` unless a specific application need exists
- Enable `secure_auth = ON`
- Enforce SSL/TLS for any remote connections; disable remote root login entirely
- Upgrade off MySQL 5.0.x entirely — EOL for well over a decade

## Tools Used
- Nmap (`-sV -sC`, MySQL NSE script family: `mysql-empty-password`, `mysql-brute`, `mysql-databases`,
  `mysql-users`, `mysql-info`, `mysql-variables`)
- `mysql` CLI client (manual verification)

## Reflections
This is the other major category of vulnerability besides "software bug with a CVE" — pure
misconfiguration. No exploit code, no backdoor, no memory corruption — just a database left wide open
by omission. Arguably more common in real assessments than flashy CVEs: attackers don't need a 0-day
when root has no password.
