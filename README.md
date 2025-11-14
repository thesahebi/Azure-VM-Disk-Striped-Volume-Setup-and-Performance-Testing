Azure VM 4‑Disk Striped Volume Setup and Performance Testing
Introduction: Disk Striping

When an Azure VM attaches multiple Premium SSDs, you can combine their performance using disk striping.

Windows: Storage Spaces (one column per disk)

Linux: mdadm (RAID/striping)

This guide covers:

Creating a 4‑disk striped volume (D:) on Windows

Running DiskSpd performance tests

Reviewing results and recommendations

Environment
Item	Specification
VM	Windows Server on Azure
Disks	4 × Premium SSD LRS (64 GiB)
Max IOPS per Disk	240
Max Throughput per Disk	50 MB/s
Target Volume	~256 GiB striped (Simple)
1 — Create the Striped Volume
Option A — Server Manager (GUI)

Open Server Manager → File and Storage Services → Storage Pools.

Create a new storage pool using all 4 disks.

Create a new virtual disk:

Layout: Simple (striped)

Provisioning: Fixed

Columns: 4

Initialize the disk (GPT).

Create a new partition, assign D: and format NTFS or ReFS.

Option B — PowerShell
Get-PhysicalDisk | Sort-Object FriendlyName | Format-Table FriendlyName, OperationalStatus, MediaType, Size
New-StoragePool -FriendlyName "StripePool" -StorageSubsystemFriendlyName "Storage Spaces*" -PhysicalDisks (Get-PhysicalDisk -CanPool $True)
New-VirtualDisk -StoragePoolFriendlyName "StripePool" -FriendlyName "Disk-Strip-Demo" -Size 256GB -ResiliencySettingName Simple -NumberOfColumns 4
$vd = Get-VirtualDisk -FriendlyName "Disk-Strip-Demo"
$disk = $vd | Get-Disk
Initialize-Disk -Number $disk.Number -PartitionStyle GPT
New-Partition -DiskNumber $disk.Number -UseMaximumSize -DriveLetter D | Format-Volume -FileSystem NTFS -NewFileSystemLabel "D_Stripe" -Confirm:$false
2 — Performance Testing with DiskSpd

Place diskspd.exe in a tools folder and run tests from PowerShell.

2.1 Sequential Write (5 minutes)
.\diskspd.exe -b64K -d300 -o4 -t4 -w100 -c10G D:\load.dat

Results:

Throughput: ~763 MB/s

IOPS: ~12,212

Even thread distribution

2.2 Random 4KB Mixed (50/50) — 5 minutes
.\diskspd.exe -b4K -d300 -o8 -t8 -w50 -r -c10G D:\load.dat

Results:

Type	MB/s	IOPS
Read	19.04	4,875
Write	19.05	4,876
Total	38.09	9,751
Cleanup
Remove-Item D:\load.dat -Force
3 — Interpretation & Notes

4 disks × 240 IOPS ≈ 960 theoretical → actual ~9,700 due to caching and bursting.

Sequential = bandwidth, random = IOPS.

NumberOfColumns = 4 ensures even load.

Simple layout = no redundancy (1 disk failure = data loss).

4 — Data Protection Options
Strategy	Description	Capacity	Fault Tolerance	Performance
Stripe + Backups	Fastest, rely on Veeam or Azure Backup	100%	None	Maximum
Mirrored Stripe	RAID‑10 style	50%	1 disk failure	High
Parity	Space‑efficient	~75%	1–2 failures	Slower writes
Snapshots + Backups	Logical protection	100%	Varies	Fast
5 — Recommendations

If speed is the priority → Simple stripe + daily backups.

If uptime matters → Mirrored stripe.

Monitor:

IOPS

Throughput

Queue depth

CPU usage

6 — Quick Reference Commands
Create Stripe
New-StoragePool -FriendlyName "StripePool" -StorageSubsystemFriendlyName "Storage Spaces*" -PhysicalDisks (Get-PhysicalDisk -CanPool $True)
New-VirtualDisk -StoragePoolFriendlyName "StripePool" -FriendlyName "Disk-Strip-Demo" -Size 256GB -ResiliencySettingName Simple -NumberOfColumns 4
DiskSpd Tests
.\diskspd.exe -b64K -d300 -o4 -t4 -w100 -c10G D:\load.dat
.\diskspd.exe -b4K -d300 -o8 -t8 -w50 -r -c10G D:\load.dat
7 — Summary
Item	Details
Configuration	4 × Premium SSD LRS (striped) → ~256 GiB
Performance	~763 MB/s sequential, ~9,751 IOPS random
Protection	Daily backups (or mirror for redundancy)
Recommendation	Stripe for speed + backup for recovery
References

Azure Premium Storage Performance

DiskSpd Documentation

Storage Spaces Overview
