# Phase 5: Before and After Comparison

This is where I measure what Phase 4 actually achieved. I compared each system's baseline scan from Phase 3 with its final scan after hardening, then broke the remaining items into three groups: real failures, scanner errors, and checks that need a human to answer.

## Headline numbers

| System | Benchmark | Baseline | Final | Change |
|---|---|---|---|---|
| FALCON-APP01 (Ubuntu 22.04) | DISA STIG (Security Technical Implementation Guide) V2R3, via Ubuntu Security Guide | 40.78% | **95.43%** | +54.65 points |
| FALCON-DC01 (Windows Server 2022) | Windows Server 2022 STIG | 44.12% | **98.53%** | +54.41 points |

Both systems more than doubled their compliance scores.

## DC01 final scan detail

Final SCC (SCAP Compliance Checker) 5.15 session: `2026-11-09_170849`. Full output is in `dc01-phase5-results.txt`.

| Benchmark | Final score |
|---|---|
| Windows Server 2022 STIG | 98.53% |
| Windows Firewall with Advanced Security | 100% |
| Windows Updates | 100% |
| .NET Framework | 100% |
| Microsoft Edge | 98.21% |
| Microsoft Defender Antivirus | 95.52% |
| Internet Explorer 11 | Removed, no longer applicable |

The .NET result confirms the fix I added at the end of Phase 4. APPNET0075 needed strong cryptography and system default TLS (Transport Layer Security) versions set for both the v2.0 and v4.0 framework keys, not just v4.0.

## APP01 final scan detail

APP01's final scan was the 95.43% run at the end of Phase 4. No configuration changed on APP01 after that scan, so it stands as the final result. Its 18 remaining items are already categorized in `app01-hardening-log.md`:

* 9 need infrastructure the lab does not have: FIPS (Federal Information Processing Standards) mode, smart card authentication (5 rules) and remote log offloading (3 rules)
* 2 are accepted deviations: the faillock unlock time and system library group ownership
* 7 are configured correctly, but the scanner disagrees: 4 faillock rules, the fdisk and journal audit rules, and journal folder permissions

## What is left on DC01, and why

### Real failures (4)

| Rule | Issue | Disposition |
|---|---|---|
| WN22-PK-000010 | DoD (Department of Defense) Root CA (Certificate Authority) certificates not installed | POA&M (Plan of Action and Milestones). Fix with the DISA InstallRoot tool |
| WN22-PK-000020 | DoD Interoperability cross certificates not in the Untrusted store | POA&M, same fix |
| WN22-PK-000030 | US DoD CCEB Interoperability cross certificates not in the Untrusted store | POA&M, same fix |
| EDGE-00-000001 | User control of proxy settings not disabled | POA&M. Needs an enterprise proxy, which the lab does not have |

### Scanner errors (3)

Defender's 95.52% does not come from failures. Three rules returned **error**, meaning SCC could not evaluate them:

* WNDF-AV-000005: no file exclusions
* WNDF-AV-000006: no process exclusions
* WNDF-AV-000007: automatic exclusions

All three look for Defender exclusion policy keys. On a clean system with no exclusions configured, those keys do not exist, so the check errors out instead of passing. I will verify manually that no exclusions are set and record the result in the SAR (Security Assessment Report).

### Manual review items (Not Checked)

SCC also reported a set of rules as **Not Checked**. These cannot be automated, because they ask about process and documentation rather than settings. Examples:

* separate admin and user accounts (WN22-00-000010)
* application allowlisting (WN22-00-000080)
* antivirus and host intrusion detection in place (WN22-00-000110, 000120)
* audit log offloading (WN22-AU-000010, 000020)
* data at rest encryption (WN22-00-000250)
* documented roles and features (WN22-00-000270)

These are not failures. They become manual answers in the Phase 7 RMF (Risk Management Framework) package, and several map directly to SSP (System Security Plan) sections.

## Findings worth calling out

1. **The STIG locked me out twice, and both times that was the control working.** On DC01, the firewall policy blocked my SSH path, and the STIG renamed the Administrator account. On APP01, pam_faillock locked my admin account after three bad sudo attempts during this phase. That gives me live evidence for AC-7 (Unsuccessful Logon Attempts).
2. **Reading logs before changing controls saved me from a bad exception.** I almost relaxed the "deny network logon" right on DC01 based on a guess. The OpenSSH log showed the real cause, which was the renamed account.
3. **Credential recovery exposed a physical access gap.** I could not remember the APP01 admin password. The recovery path is to boot the VM from install media and reset it, which works even with a GRUB password. Anyone with access to the virtual disk could do the same. That is an argument for disk encryption, and it lines up with the open data at rest item (WN22-00-000250). It goes in the POA&M.

## Carried into Phase 6 and 7

* Recover the APP01 admin password through install media, and document the procedure
* Manually verify the Defender exclusion settings
* Change the DC01 X_Admin password before December 3, when expiry kicks in
* Decide whether OpenSSH on DC01 stays as an approved management service or gets removed (CM-7 Least Functionality)
* Answer the manual review items in the SSP and SAR
