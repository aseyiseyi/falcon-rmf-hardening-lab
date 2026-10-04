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
