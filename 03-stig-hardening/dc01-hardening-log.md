# FALCON-DC01 Hardening Log

Baseline before hardening: 44.12% on the Windows Server 2022 STIG, with Windows Updates at 0%. Before changing anything, I took snapshot 02-pre-hardening so I could roll back.

## October 11, 2026: Patching

The server was still on build 20348.587, the original 2022 release, so it was missing about four and a half years of security fixes. I ran Windows Update and installed:

| Update | What it is |
|---|---|
| KB5122882 | 2026-09 cumulative update for Windows Server 2022 |
| KB5122881 | Security update installed with the cumulative update |
| KB5126149 / KB5126050 | 2026-09 cumulative update for .NET Framework |
| KB2267602 | Defender security intelligence (signature) update |
| KB890830 | Malicious Software Removal Tool |

Windows cumulative updates include every earlier fix, so one package brought the OS fully current. That's also why the 0% baseline finding was so serious: missing one package meant missing every fix since 2022.

## How I verified it

`cmd /c ver` now reports build **10.0.20348.5622**, up from 20348.587. `Get-HotFix` lists the installed KBs. Windows Update reports the server is up to date.

`Get-HotFix` dated the installs October 12 even though I installed them on the evening of October 11. It only records the date, and it appears to be in UTC. The Defender and MSRT updates don't show up in `Get-HotFix` because it only lists operating system patches.

NIST 800-53: SI-2 (Flaw Remediation), SI-3 (Malicious Code Protection, for the Defender signatures)

Evidence: [screenshots/08-dc01-patched.png](../screenshots/08-dc01-patched.png)
