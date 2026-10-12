# FALCON-APP01 Hardening Log

Baseline before hardening: 40.78% on the Ubuntu 22.04 STIG V2R3. Before changing anything, I took snapshot 02-pre-hardening so I could roll back.

## October 11, 2026

| Change | Why | NIST 800-53 |
|---|---|---|
| Applied all pending updates and confirmed 0 remaining | Close known vulnerabilities | SI-2 |
| Confirmed unattended-upgrades is enabled | Security patches keep landing daily without waiting on me | SI-2 |
| Removed 16 unneeded packages, including modemmanager and USB modem tools | A server doesn't need modem software, and every extra package is attack surface | CM-7 |
| Installed auditd and loaded 31 audit rules | The server had no audit trail | AU-2, AU-3, AU-12 |
| Switched SSH to key-only login and blocked root login | Removes password brute forcing and makes every session traceable to a named user | IA-2, IA-5, AC-17 |

## How I verified each change

**Patching.** `apt list --upgradable` returned 0. The unattended-upgrades log showed it had installed 9 security updates on its own at 20:53, including sudo.

**Audit logging.** I ran `sudo whoami` as a test event, then searched for it with `ausearch -k priv_cmd`. The record showed `auid=falconadmin` alongside `euid=root`, so even commands run as root are tied back to the person who ran them. The full loaded rule set is in [auditctl-loaded-rules.txt](app01/auditctl-loaded-rules.txt).

**SSH.** I made the change in a specific order so I couldn't lock myself out: create the key on my host, install it on the server, test key login, and only then turn off passwords. I tested the config with `sshd -t` before reloading. From my host, a login attempt without the key now gets `Permission denied (publickey)` with no password prompt. The settings SSH actually enforces are in [sshd-effective-settings.txt](app01/sshd-effective-settings.txt).

## Things I ran into

**Corrupted log.** `/var/log/dpkg.log` contained binary data, and grep stopped reading partway through. The timing lines up with the hard reset I did on October 4, when the VM froze mid-upgrade. I used `grep -a` to read past it. It's a reminder that unclean shutdowns can damage audit-relevant logs (AU-9).

**Installer setting.** The Ubuntu installer had written its own SSH setting that allowed passwords. SSH uses the first value it reads, and config files load alphabetically, so I named my file 00-falcon-hardening.conf to make sure it wins. I confirmed this with `sshd -T`.

## Config files

- [50-falcon-stig.rules](app01/50-falcon-stig.rules): audit rules
- [00-falcon-hardening.conf](app01/00-falcon-hardening.conf): SSH hardening
