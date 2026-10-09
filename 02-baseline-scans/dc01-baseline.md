# FALCON-DC01 Baseline Scan

On October 4, 2026, I ran a baseline compliance scan against FALCON-DC01 before making any security changes. I wanted an honest picture of how a default Windows Server 2022 install holds up against DoD requirements.

I used DISA's SCAP Compliance Checker, version 5.15. Before installing it, I checked the download's SHA-256 hash against the checksum file DISA publishes, and it matched. I ran a local scan using the MAC-1_Classified profile. That's the strictest profile available, and I picked it on purpose so the scan would show me every gap instead of a softer version of the problem.

The server is running build 20348.587, which is the original 2022 release. I haven't applied any updates yet.

## Results

| Benchmark | Score |
|---|---|
| Windows Server 2022 STIG | 44.12% |
| .NET Framework | 85.71% |
| Defender Antivirus | 26.87% |
| Windows Firewall | 25.00% |
| Internet Explorer 11 | 0.73% |
| Edge | 0.00% |
| Windows Updates | 0.00% |

## What I took away from it

The number that matters most is 44.12% on the Windows Server 2022 STIG. A clean install from Microsoft fails more than half of the settings DoD requires, which tells me "new" does not mean "secure."

Patching is my first priority. The server is about four years behind on updates, and that's the kind of gap attackers look for first.

Defender and the firewall are both turned on, but they're running on default settings that don't meet the STIG. Having a control in place isn't the same as having it configured correctly.

Edge and Internet Explorer 11 scored close to zero. Nobody will browse the web from this server, so part of my hardening plan is deciding whether to configure those components or remove what isn't needed.

Next, I'll work through the non-compliance report, harden the server, and rescan to measure the improvement.

Evidence: [screenshots/06-dc01-baseline-scores.png](../screenshots/06-dc01-baseline-scores.png)
