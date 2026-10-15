# Lab Journal

## Oct 3, 2026

Downloaded Windows Server 2022 Evaluation and Ubuntu Server 22.04.5 ISOs.
Installed VirtualBox 7.2.20 and the Extension Pack.
Confirmed CPU virtualization is enabled (8 cores, 16 logical processors).
Issue: VirtualBox showed "Can't enumerate USB devices." USB passthrough is not needed for this lab, so I dismissed it.
Installed Git and the GitHub CLI, authenticated to GitHub, and built the repo structure from PowerShell.

## Oct 4, 2026

Created FALCON-NET, a NAT network (10.10.10.0/24) with DHCP, as the system's authorization boundary.
Built FALCON-DC01 entirely from the command line with VBoxManage: 8 GB RAM, 4 vCPU, UEFI, 60 GB dynamic disk.
Installed Windows Server 2022 Standard (Desktop Experience) and VirtualBox Guest Additions.
Renamed the host to FALCON-DC01 and set the time zone to Eastern. Accurate time matters for audit log integrity.
Issue: a live snapshot hung at 90% for over 20 minutes. Diagnosed with showvminfo (state stuck at livesnapshotting), powered off the VM, and took an offline snapshot instead, which completed in seconds.
Lesson: take snapshots with the VM powered off.
Snapshot 01-clean-install saved as the pre-hardening baseline.

## Oct 4, 2026 (continued)

Built FALCON-APP01 from the command line: Ubuntu Server 22.04.5, 4 GB RAM, 2 vCPU, 25 GB disk, LVM, OpenSSH server. DHCP assigned 10.10.10.4 on FALCON-NET.
Left disk encryption (LUKS) and SSH password authentication enabled on purpose. Both will be documented as STIG findings and tracked in the POA\&M.
Issue: apt upgrade froze at 0% for over 25 minutes. Task Manager showed 0 MB/s disk and one CPU thread pinned, meaning the VM was hung rather than slow. Reset the VM and repaired the package database with dpkg --configure -a.
Root cause investigation: the VirtualBox log showed "fall back to NEM: VT-x is not available." Windows was holding the CPU virtualization extensions, forcing VirtualBox into a slower compatibility mode. This also explained the earlier hung live snapshot.
Disabling the hypervisor with bcdedit and turning off Memory Integrity was not enough. Querying Win32\_DeviceGuard showed VBS still running, and the DeviceGuard registry keys showed it was being enforced by EnableVirtualizationBasedSecurity and Windows Hello Enhanced Sign-in Security. Disabled both, restarted, and confirmed in the log that VirtualBox now uses VT-x with nested paging.
Risk decision: accepted reduced host hardening in exchange for a stable lab. Documented as reversible.
Completed apt upgrade with zero pending updates. Snapshot 01-clean-install taken for FALCON-APP01.

## Oct 11, 2026

Added an SSH port forward bound to 127.0.0.1:2222 so I can manage FALCON-APP01 from my host terminal instead of the VM console. Attached the server to Ubuntu Pro, installed USG, and ran the DISA STIG baseline audit: 40.78%. Found auditd isn't installed, which is why 90 audit rules came back not applicable. Copied the report to the host with scp. Phase 3 complete.



## Oct 11, 2026 (continued)

Started Phase 4 on FALCON-APP01. Patched, removed 16 unneeded packages, installed auditd with 31 rules, and switched SSH to key-only. Ran a command on the wrong machine twice tonight (the key got generated on the server), so I now check the prompt before every command. Found my dpkg log was corrupted by the Oct 4 hard reset.

## Oct 11, 2026 (late)

Patched FALCON-DC01 with the 2026-09 cumulative update. The build went from 20348.587 to 20348.5622. Took snapshot 02-pre-hardening first. Copy and paste still doesn't work in DC01, so I keep commands short and type them one line at a time.

## 


## Oct 15, 2026
APP01 went from 40.78% to 71.46% on the STIG today. Closed 2 of 4 CAT I findings and about 23 rules total. A typo in a heredoc leaked my SSH crypto settings into the login banner, and a scan froze because of pasted text. Both caught and fixed. Lesson: check the clipboard before pasting into a root shell.

## Oct 15, 2026 (evening)
Finished APP01: 95.43%, up from 40.78%. Added password policy, AIDE, and a GRUB password, and tested an unattended reboot. Chose not to apply one benchmark fix because it would have weakened security. Kept the GRUB hash out of the repo.
