# FALCON-APP01 Baseline Scan

On October 11, 2026, I ran a baseline STIG audit against FALCON-APP01 before making any security changes.

DISA's SCAP Compliance Checker 5.15 doesn't list Ubuntu as a supported platform, so I used the Ubuntu Security Guide (USG 22.04.19), Canonical's compliance tool for Ubuntu. Under the hood it runs OpenSCAP with current content. To get it, I attached the server to a free Ubuntu Pro subscription. I pasted the subscription token with a leading space so Bash would leave it out of the shell history.

I scanned against the Canonical Ubuntu 22.04 LTS STIG, V2R3, using the disa_stig profile. USG's audit mode only reads the system, so nothing changed during the scan. I connected to the server over SSH through a port forward bound only to 127.0.0.1 on my host, so the server isn't reachable from my home network.

## Results

Overall score: **40.78%**

| Result | Rules |
|---|---|
| Pass | 49 |
| Fail | 70 |
| Not applicable | 90 |
| Not checked (manual review) | 5 |

Of the 119 rules that were actually evaluated, the server failed 70.

## What I took away from it

The 90 "not applicable" results are almost all audit logging rules. They don't apply because auditd isn't installed. That's the most important finding here: right now the server keeps no audit trail of who uses sudo, changes accounts, or mounts drives. Installing auditd will make those rules count, so I expect the number of evaluated rules to go up during hardening, and I'll need to pass those too.

Attaching Ubuntu Pro also unlocked 9 ESM security updates. I left them unapplied on purpose so this baseline stays an accurate "before" picture. Patching is the first step of hardening.

The package manager also lists software the server doesn't need, like modemmanager and usb-modeswitch. Removing it falls under least functionality (NIST CM-7).

When I built the server, I deliberately left SSH password login enabled and skipped disk encryption. Both are STIG requirements, and I'll address them during hardening or document them in the POA&M.

## Comparison with FALCON-DC01

| Server | Baseline score |
|---|---|
| FALCON-DC01, Windows Server 2022 | 44.12% |
| FALCON-APP01, Ubuntu 22.04 | 40.78% |

Both default installs fail more than half of the DoD requirements.

Full report: [app01-usg-baseline.html](app01-usg-baseline.html) (download it to view; GitHub shows the raw HTML)
Evidence: [screenshots/07-app01-baseline-score.png](../screenshots/07-app01-baseline-score.png)
