# DC01 Hardening Log (Windows Server 2022)

This is my working log for hardening FALCON-DC01 against the DISA (Defense Information Systems Agency) Windows Server 2022 STIG (Security Technical Implementation Guide) and its companion benchmarks. I scanned with DISA SCC (SCAP Compliance Checker) 5.15 before and after every major change so each claim below has a scan behind it.

## System context

| Item | Value |
|---|---|
| Host name | FALCON-DC01 |
| OS | Windows Server 2022 Standard |
| Role in this lab | Standalone server (DomainRole 2). It is named DC01 for the planned domain controller role, but it is not promoted yet, so I applied the Member Server policy set |
| Network | FALCON-NET NAT network, 10.10.10.0/24, gateway 10.10.10.1 |
| Management path | SSH from my host through a VirtualBox port forward bound to 127.0.0.1 only |
| Scanner | DISA SCC 5.15, SHA-256 verified before install |

## Results at a glance

| Benchmark | Baseline (Oct) | After GPO (Nov 9) | Final (Nov 9) |
|---|---|---|---|
| Windows Server 2022 STIG | 44.12% | 96.57% | **98.53%** |
| Windows Firewall with Advanced Security | not recorded | 100% | **100%** |
| Windows Updates | not recorded | 0% | **100%** |
| Microsoft Edge STIG | not recorded | 98.21% | **98.21%** |
| Microsoft Defender Antivirus | not recorded | 95.52% | **95.52%** |
| .NET Framework | not recorded | 85.71% | **92.86%** |
| Internet Explorer 11 | not recorded | 99.27% | **Removed, no longer applicable** |

Evidence files in this folder: `dc01-scores-after-gpo.txt`, `dc01-scores-final.txt`, `dc01-fails-final.txt`.

## What I did, in order

### 1. Patched before applying policy

I brought the server to build 20348.5622 in October and took an offline snapshot (`03-patched-pre-gpo`) before touching policy, so I had a clean rollback point.

### 2. Applied the DISA GPO package

I downloaded the July 2026 DISA GPO (Group Policy Object) package from the NIST NCP (National Checklist Program) and verified its SHA-256 hash before use. I applied it with Microsoft LGPO (Local Group Policy Object utility, Authenticode signature verified) because the server is not domain joined. I applied these policy sets:

* Windows Server 2022 Member Server v2r9
* Microsoft Defender Antivirus v2r9
* Windows Defender Firewall v2r2
* Microsoft Edge v2r5
* Internet Explorer 11 v2r7

### 3. Finding: the firewall policy locked out my own management path

Right after the GPO went on, SSH stopped responding. The DISA firewall policy sets inbound traffic to block and sets local firewall rules to "not merged," which means any rule created the normal way is ignored. Only rules stored in Group Policy count.

My first attempt to add a rule with `New-NetFirewallRule -PolicyStore localhost` failed with "No mapping between account names and security IDs was done." The DISA rules reference domain groups that do not exist on a standalone server, and the cmdlet fails while reading them.

**Fix:** I wrote a single allow rule directly into the local Group Policy registry store with LGPO. It allows TCP 22 only from 10.10.10.1, which is the only address my port forward comes from. Everything else inbound stays blocked.

```
Computer
SOFTWARE\Policies\Microsoft\WindowsFirewall\FirewallRules
FALCONSSH
SZ:v2.30|Action=Allow|Active=TRUE|Dir=In|Protocol=6|LPort=22|RA4=10.10.10.1|Name=FALCON SSH admin|
```

The firewall benchmark still scores 100% with this rule in place.

### 4. Finding: the STIG renamed the Administrator account

With the firewall open, SSH still refused my password. My first theory was the STIG right that denies network logon to local administrator accounts. Before changing anything I exported the user rights and checked. The deny network logon right only listed Guests, so that theory was wrong and I did not weaken it.

The OpenSSH event log gave the real answer: "Invalid user Administrator." The DISA policy renames the built in Administrator account to X_Admin (and Guest to Visitor) so attackers cannot target a known account name. I updated the user name in my SSH client config and access came back with no exceptions and no weakened settings.

Lesson I am keeping: read the log before changing a control. My guess would have created an unnecessary exception.

### 5. Rescan after GPO

The Windows Server 2022 STIG went from 44.12% to 96.57%. Eleven rules were still failing across all benchmarks. I sorted them into fix now, fix through VM settings, and document.

### 6. Fixes applied

| Rule | What it requires | What I did |
|---|---|---|
| APPNET0063 | .NET must validate strong names on full trust assemblies | Set `AllowStrongNameBypass = 0` in the 64 bit and 32 bit .NET keys |
| APPNET0075 | .NET must use strong cryptography for TLS (Transport Layer Security) | Set `SchUseStrongCrypto = 1` for v4. Still failing after the first pass, so I also set `SchUseStrongCrypto` and `SystemDefaultTlsVersions` for v2.0.50727 and v4.0.30319 in both registry views. To be confirmed in the Phase 5 rescan |
| WN22-UC-000010 | Preserve zone information on saved attachments | Set `SaveZoneInformation = 2` for the admin user. This is a user policy, and gpupdate had reported a user registry policy warning, which is why the GPO alone did not apply it |
| WN22-00-000210 | Passwords must expire | X_Admin was set to never expire. I confirmed the password was 36 days old (under the 60 day maximum) before turning expiry on, so I would not lock myself out of SSH. Password must be changed before December 3 |
| DTBI999-IE11 | Unsupported IE version | Removed the Internet Explorer 11 optional feature entirely. This also supports CM-7 (Least Functionality) |
| WN22-00-000090 | TPM (Trusted Platform Module) enabled and ready | Added a virtual TPM 2.0 with `VBoxManage modifyvm --tpm-type 2.0`. Windows reports TPM present, ready and enabled |
| WN22-00-000470 | Secure Boot enabled | The VM already used UEFI (Unified Extensible Firmware Interface) firmware, so no reinstall was needed. I enrolled the Microsoft signing keys and platform key and enabled Secure Boot with `VBoxManage modifynvram`. `Confirm-SecureBootUEFI` returns True |
| Windows Updates | No missing updates | Installed the November cumulative updates (KB5126654, KB5126655, KB5127687) through sconfig and rebooted. Build moved from 20348.5622 to 20348.5761 |

I took snapshot `04-gpo-hardened-96pct` before the firmware change. One mistake worth recording: I reinitialized the UEFI variable store after enrolling the keys, which wiped them, and Windows reported Secure Boot as False. I powered off, enrolled the keys again without the reset, and it came back True.

### 7. Final scan

Windows Server 2022 STIG finished at **98.53%**, with Firewall and Windows Updates at 100%.

## Remaining findings (going to the POA&M)

These go into the POA&M (Plan of Action and Milestones) in Phase 7.

| Rule | Why it is still open | Planned action |
|---|---|---|
| WN22-PK-000010 | DoD (Department of Defense) Root CA (Certificate Authority) certificates not installed | Install with the DISA InstallRoot tool. Not needed for an isolated lab with no DoD PKI (Public Key Infrastructure) dependencies |
| WN22-PK-000020 | DoD Interoperability cross certificates not in the Untrusted store | Same, InstallRoot |
| WN22-PK-000030 | US DoD CCEB Interoperability cross certificates not in the Untrusted store | Same, InstallRoot |
| EDGE-00-000001 | User control of proxy settings must be disabled | Requires an enterprise proxy configuration. The lab has no proxy, so this stays open with a documented reason |
| APPNET0075 | .NET TLS | Additional registry keys applied, pending confirmation in the Phase 5 rescan |

Defender Antivirus scores 95.52% with no rules in a failed state, so the gap comes from checks SCC could not evaluate automatically. I will review those manually in Phase 5.

## Open items I am tracking

* **OpenSSH Server on DC01:** I installed it as a management path. It is restricted by the firewall to 10.10.10.1 and only reachable through a port forward bound to my host's loopback address. I will either justify it in the SSP (System Security Plan) as an approved management service or remove it before the final package, in line with CM-7.
* **X_Admin password:** must be changed before December 3 now that expiry is enforced.
