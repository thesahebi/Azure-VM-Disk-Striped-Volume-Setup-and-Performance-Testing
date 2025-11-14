Azure VM 4‑Disk Striped Volume Setup and Performance Testing
Introduction: Disk Striping

When an Azure VM attaches multiple Premium Storage disks, they can be striped to combine IOPS, bandwidth, and capacity.

Windows: Uses Storage Spaces with one column per disk.

Linux: Uses mdadm for RAID/striping.

Overview

This document guides you through:

Creating a 4‑disk striped volume (D:) on Windows using Storage Spaces

Running DiskSpd performance tests

Reviewing results and recommendations

Example configuration: 4 × Premium SSD LRS (64 GiB each)

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

Initialize disk (GPT) and create a new partition.

Assign drive letter D: and format (NTFS or ReFS).

Option B — PowerShell
Get-PhysicalDisk | Sort-Object FriendlyName | Format-Table FriendlyName, OperationalStatus, MediaType, Size


New-StoragePool -FriendlyName "StripePool" `
                -StorageSubsystemFriendlyName "Storage Spaces*" `
                -PhysicalDisks (Get-PhysicalDisk -CanPool $True)


New-VirtualDisk -StoragePoolFriendlyName "StripePool" `
                -FriendlyName "Disk-Strip-Demo" `
                -Size 256GB `
                -ResiliencySettingName Simple `
                -NumberOfColumns 4


$vd = Get-VirtualDisk -FriendlyName "Disk-Strip-Demo"
$disk = $vd | Get-Disk
Initialize-Disk -Number $disk.Number -PartitionStyle GPT
New-Partition -DiskNumber $disk.Number -UseMaximumSize -DriveLetter D |
    Format-Volume -FileSystem NTFS -NewFileSystemLabel "D_Stripe" -Confirm:$false
2 — Performance Testing with DiskSpd

Place diskspd.exe in a tools folder and run from PowerShell.

2.1 Sequential Write (5 minutes)
.\u200bdiskspd.exe -b64K -d300 -o4 -t4 -w100 -c10G D:\load.dat

Results:

Throughput: ~763 MB/s

IOPS: ~12,212

Even thread distribution

2.2 Random 4KB Mixed (50/50) – 5 minutes
.\u200bdiskspd.exe -b4K -d300 -o8 -t8 -w50 -r -c10G D:\load.dat

Results:

Type	MB/s	IOPS
Read	19.04	4,875
Write	19.05	4,876
Total	38.09	9,751
Cleanup
Remove-Item D:\load.dat -Force
3 — Interpretation & Notes

4 disks × 240 IOPS = ~960 theoretical → measured ~9,700 due to caching and bursting.

Sequential tests hit bandwidth limits; random tests hit IOPS limits.

Using NumberOfColumns = 4 ensures even distribution.

Simple layout = no redundancy (1 disk failure → data loss).

4 — Data Protection Options
Strategy	Description	Capacity	Fault Tolerance	Performance
Stripe + Backups	Fastest, rely on Veeam	100%	None	Maximum
Mirrored Stripe	RAID‑10 style	50%	1 disk failure	High
Parity	Space‑efficient	~75%	1–2 failures	Slower writes
Snapshots + Backups	Fast restore	100%	Logical protection	Fast
5 — Recommendations

If performance is the priority → use Simple stripe with reliable daily backups.

If uptime matters → use mirrored stripe.

Monitor: IOPS, throughput, queue depth, CPU.

Customize DiskSpd for your workload (block size, thread count, queue depth).

6 — Quick Reference Commands
Create Stripe
New-StoragePool -FriendlyName "StripePool" `
                -StorageSubsystemFriendlyName "Storage Spaces*" `
                -PhysicalDisks (Get-PhysicalDisk -CanPool $True)


New-VirtualDisk -StoragePoolFriendlyName "StripePool" `
                -FriendlyName "Disk-Strip-Demo" `
                -Size 256GB `
                -ResiliencySettingName Simple `
                -NumberOfColumns 4
DiskSpd Tests
.\u200bdiskspd.exe -b64K -d300 -o4 -t4 -w100 -c10G D:\load.dat
.\u200bdiskspd.exe -b4K -d300 -o8 -t8 -w50 -r -c10G D:\load.dat
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
