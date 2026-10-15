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

## October 15, 2026

I rescanned after the October 11 work and the score rose from 40.78% to 61.91%. The fail count also went up, from 70 to 106. Installing auditd made about 80 audit rules applicable that couldn't be evaluated before, so the scan was checking more of the system, not finding it worse. I then triaged the remaining 106 failures by severity: 4 CAT I, 96 CAT II, and 6 CAT III.

| Change | Why | NIST 800-53 |
|---|---|---|
| Masked ctrl-alt-del.target (CAT I) | Anyone at the console could reboot the server | CM-6 |
| Set the hardware clock to UTC (CAT I) | Unambiguous timestamps in logs | AU-8 |
| Added the DoD Notice and Consent Banner | Users must see the monitoring notice before login | AC-8 |
| Restricted SSH ciphers, MACs, and key exchange to FIPS-approved algorithms | Weak algorithms can be broken or downgraded | SC-8, SC-13 |
| Set a 10 minute SSH idle disconnect and disabled X11 forwarding and user environment | Unattended sessions are an open door; unused features are attack surface | AC-12, CM-7 |
| Enabled full ASLR, SYN cookies, and dmesg restriction | Harder exploitation, flood protection, less information leakage | SI-16, SC-5 |
| Blocked USB storage | Prevents data theft and malware through USB drives | CM-7, MP-7 |
| Added a 10 minute idle shell timeout, installed vlock, set UMASK 077 | Session lock and private-by-default files | AC-11, AC-6 |
| Enabled ufw with default deny incoming and rate-limited SSH | The server had no firewall | SC-7, AC-17 |
| Replaced systemd-timesyncd with chrony | STIG requires chrony for accurate time | AU-8 |

After these fixes the score is **71.46%** (112 pass, 87 fail).

| Scan | Score | Pass | Fail |
|---|---|---|---|
| Baseline, Oct 11 | 40.78% | 49 | 70 |
| After core fixes, Oct 15 | 61.91% | 92 | 106 |
| After quick wins, Oct 15 | 71.46% | 112 | 87 |

## Things I ran into

**Banner typo turned into an information leak.** I mistyped the end marker for the banner file as `EOFz`, so Bash kept writing everything that followed into `/etc/issue.net`, including all of my SSH cipher settings. The banner is shown to anyone who connects, before login, so it was advertising my exact crypto configuration. I trimmed the file back to the single banner line and confirmed it with `wc -l` and a fresh connection.

**Frozen scan.** A rescan hung with a lock error. `top` showed the CPU idle with 20 stopped tasks, and `ps` showed the scan processes in state T (stopped), most likely from stray text I pasted into the terminal. I killed them with `pkill`, using the pattern `[u]sg audit` so the command wouldn't match and kill my own SSH session, removed the stale lock, and reran the scan in the foreground.

**Lockout prevention.** Before enabling the firewall I added the SSH rule first, and after changing SSH crypto I tested a fresh login from my host while keeping the existing session open.

Evidence for today is in [app01/](app01/): the SSH, sysctl, USB, and timeout config files, the effective SSH settings, and the firewall status.

### Audit expansion (Oct 15, 4:30 PM)
Added a second rules file (51-falcon-stig-extra.rules) covering extended attributes, ACL tools, rmdir, more privileged commands, root execution by group, and failed file access attempts (EACCES and EPERM). Loaded rules went from 31 to 61. Set auditd to warn at 25% free space and restricted audit log ownership to root. Tightened permissions on audit logs, rules files, journalctl, and the journal folder.

Score: **75.57%** (147 pass, 52 fail).

Lesson: sudo chmod 600 /var/log/audit/*.log failed because my own shell expanded the wildcard before sudo ran, and it couldn't read the root-only folder. Running the command with sudo sh -c fixed it.

The remaining 52 failures are in [app01-fails-1636.txt](app01/app01-fails-1636.txt). A few rules I'd already fixed still fail because the scanner reads config files that disagree with the effective settings, for example X11Forwarding yes in the main sshd_config while my drop-in sets no.

### Persistence and benchmark-guided fixes (Oct 15, 5:00 PM)

A rescan showed several rules I had already fixed were still failing, so I investigated each one:

- **X11 forwarding.** The main sshd_config still said `X11Forwarding yes`. My drop-in file was what SSH actually enforced, but the scanner reads the config files too, so I changed the main file to match.
- **Journal permissions reverted after a reboot.** systemd-tmpfiles resets /var/log/journal at every boot, and my chmod had been undone. I created an override in /etc/tmpfiles.d/systemd.conf, which package updates won't overwrite, removed the ACL entries, and confirmed the fix survived a second reboot.
- **Read the benchmark instead of guessing.** For the last three failures I pulled the official fix scripts out of the USG benchmark XML. That showed `ensure_rtc_utc_configuration` actually checks the system time zone, not the hardware clock, and that the banner needs line breaks between sections.

Changes:

| Change | NIST 800-53 |
|---|---|
| Set the system time zone to UTC (CAT I). FALCON-DC01 stays on Eastern, documented for the SSP | AU-8 |
| Reformatted the DoD banner with section breaks, wrapped at 80 characters | AC-8 |
| Made journal permissions permanent with a tmpfiles.d override and removed the ACLs | AU-9, CM-2 |
| Set chrony maxpoll 16 on all pools and makestep 1 -1 | AU-8 |
| Set audit log folder to 700 and disk_full_action to HALT | AU-5, AU-9 |

**Why HALT is safe here:** auditd rotates its logs (`max_log_file_action = ROTATE`, `num_logs = 5`), so audit logs alone can't fill the disk, and the 25% warning fires well before that. If the disk does fill for another reason, the system stops rather than run without an audit trail.

**Session timeout confirmed in practice.** My SSH session was automatically logged out after 10 minutes of inactivity, which shows the TMOUT control is enforced.

Score: **81.23%** (154 pass, 45 fail).

### Password policy, AIDE, and boot hardening (Oct 15, 5:15 to 6:00 PM)

I made the PAM changes from a root shell so a mistake couldn't lock me out of admin rights, and I tested sudo with `su - falconadmin` after each change.

| Change | NIST 800-53 |
|---|---|
| Password complexity: 15 character minimum, all four character types, 8 changed characters, dictionary check, enforced | IA-5 |
| SHA-512 password hashing, 60 day maximum and 1 day minimum age, account disabled 35 days after expiry | IA-5, AC-2 |
| Lockout after 3 failures in 15 minutes, 4 second delay after a failed login, at most 10 sessions per user | AC-7, AC-10 |
| AIDE baseline of 131,373 files, daily check, audit tools protected, every check reported | SI-7, AU-9 |
| GRUB password for editing boot entries, normal boots unrestricted | AC-3 |
| Auditing from first boot (audit=1) | AU-14 |
| Smart card packages installed but not enabled, ready for future CAC integration | IA-2(12) |

**Lockout decision.** The STIG wants `unlock_time = 0`, meaning an admin has to unlock a locked account. I chose 900 seconds (automatic unlock after 15 minutes), because this server has a single administrator and no second account that could do the unlock. With the 15 character password policy, 3 guesses every 15 minutes isn't a practical attack. This is tracked as an accepted deviation in the POA&M.

**Boot test.** After adding the GRUB password, I rebooted and confirmed the server came back without anyone at the console. `/proc/cmdline` shows `audit=1`. The GRUB password hash is deliberately not in this repo, because a published hash can be cracked offline.

**A remediation I decided not to apply.** The benchmark's fix for `root_permissions_syslibrary_files` would change three library files to group root: the dbus launch helper, utempter, and a vlock module. Those groups are set on purpose to limit who can run privileged helpers. Changing them would break those helpers or widen access, so I documented it as an accepted deviation instead.

**Final score: 95.43%** (181 pass, 18 fail), up from 40.78%.

### Remaining 18 failures

| Category | Rules | Status |
|---|---|---|
| Needs infrastructure this lab doesn't have | FIPS mode (CAT I), smart card / CAC (5), remote log server (3) | POA&M |
| Accepted deviations | faillock unlock_time, library group ownership | POA&M, risk accepted |
| Configured, scanner still disagrees | 4 faillock rules, fdisk and journal audit rules, journal folder permissions | POA&M, open for investigation |

Full list: [app01-fails-final.txt](app01/app01-fails-final.txt)
