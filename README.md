# Azure VM 4-Disk Striped Volume Setup and Performance Testing

## Disk Striping (Intro)

When a high-scale VM is attached with several Premium Storage persistent disks, the disks can be **striped together** to aggregate their IOPs, bandwidth, and storage capacity.

- **Windows:** Use **Storage Spaces** to stripe disks. Configure one column per physical disk to ensure even distribution of I/O across disks. Server Manager UI supports up to 8 columns; for more disks use PowerShell and set `NumberOfColumns` equal to the number of disks.
- **Linux:** Use the **mdadm** utility to build RAID/stripe sets.

**Reference:** [Azure Premium Storage Performance](https://learn.microsoft.com/en-us/azure/virtual-machines/premium-storage-performance)

---

## Overview

This README shows how to create a high-performance **4-disk striped volume (D:)** on a Windows Azure VM using Storage Spaces, how to test it using DiskSpd, and summarizes the measured results and recommendations. The example uses **4 × Premium SSD LRS, 64 GiB**.

---

## Environment

- **VM:** Windows Server (Azure)
- **Disks attached:** 4 × Premium SSD LRS, 64 GiB each
- **Max IOPS per disk:** 240
- **Max throughput per disk:** 50 MB/s
- **Target volume (striped D:):** ~256 GiB usable (simple/striped)

---

## 1 — Create the Striped Volume

### Option A — GUI (Server Manager)

1. Open **Server Manager → File and Storage Services → Storage Pools**.
2. Click **New Storage Pool** → select available disks → name the pool (e.g., `StripePool`).
3. After pool creation, choose **Tasks → New Virtual Disk**:
   - Storage pool: `StripePool`
   - Layout: **Simple (No resiliency)** *(this is striping)*
   - Provisioning: **Fixed** for best performance *(or Thin if you prefer)*
   - Enclosure awareness: enable only if you have disks across separate enclosures
   - **Number of columns:** `4` *(one column per disk)*
4. Create the virtual disk, then format:
   - Initialize the virtual disk → **GPT**
   - New Partition → assign drive letter **D:**
   - Format → **NTFS** *(or ReFS)* and set label (e.g., `D_Stripe`)

> **Tip:** `Simple` = no redundancy. If you need fault tolerance, use mirrored layout (two-way/three-way) instead.

---

### Option B — PowerShell *(recommended for reproducibility)*

Run the following as **Administrator** in the VM:

```powershell
# 1. Verify available physical disks
Get-PhysicalDisk | Sort FriendlyName | Format-Table FriendlyName, OperationalStatus, MediaType, Size

# 2. Create storage pool (uses all CanPool $True disks)
New-StoragePool -FriendlyName "StripePool" -StorageSubsystemFriendlyName "Storage Spaces*" -PhysicalDisks (Get-PhysicalDisk -CanPool $True)

# 3. Create striped virtual disk (Simple = stripe), set columns to 4
New-VirtualDisk -StoragePoolFriendlyName "StripePool" -FriendlyName "Disk-Strip-Demo" -Size 256GB -ResiliencySettingName Simple -NumberOfColumns 4

# 4. Initialize and format the virtual disk
$vd = Get-VirtualDisk -FriendlyName "Disk-Strip-Demo"
$disk = ($vd | Get-Disk)
Initialize-Disk -Number $disk.Number -PartitionStyle GPT
New-Partition -DiskNumber $disk.Number -UseMaximumSize -DriveLetter D | Format-Volume -FileSystem NTFS -NewFileSystemLabel "D_Stripe" -Confirm:$false

# 5. Verify
Get-VirtualDisk | Format-Table FriendlyName, ResiliencySettingName, Size, NumberOfColumns
Get-PhysicalDisk | Format-Table FriendlyName, OperationalStatus, CanPool, Size

2 — Performance Testing (DiskSpd)
DiskSpd is a Microsoft I/O testing tool. Place DiskSpd.exe in a folder on the VM and run commands from that folder (PowerShell as Admin). Use -c<size> to create the test file if it doesn't exist.
2.1 Sequential Write (5 minutes)
powershell# Create a 10GB test file and run 5-minute sequential write test
.\diskspd.exe -b64K -d300 -o4 -t4 -w100 -c10G D:\load.dat

Block size: 64 KB
Duration: 300s (5 min)
Threads: 4
Workload: 100% write (sequential)

Measured (Sequential Write)





















MetricValueTotal throughput~763 MB/sTotal IOPS~12,212Balance~190–192 MB/s per thread (even distribution)

2.2 Random Mixed 4 KB (5 minutes)
powershell.\diskspd.exe -b4K -d300 -o8 -t8 -w50 -r -c10G D:\load.dat

Block size: 4 KB
Duration: 300s
Threads: 8
Outstanding IOs per thread: 8
Random I/O, 50% read / 50% write

Measured (Random 4KB Mixed)

























MetricMB/sIOPSRead19.044,875Write19.054,876Total38.099,751
This shows strong aggregated IOPS for 4-disk stripe on random workloads.

Cleanup (after tests)
powershellRemove-Item D:\load.dat -Force

3 — Interpretation & Notes

Aggregation: Striping combines per-disk IOPS/throughput so 4 × 64 GiB (240 IOPS, 50 MB/s each) can deliver far more than a single disk.
Sequential vs Random:
Sequential throughput (~763 MB/s) is bandwidth-bound.
Random small-block IOPS (~9.7k total) is IOPS-bound.

Why measured > theoretical per-disk: Caching, sequential burst behavior, and testing parameters can produce higher sustained numbers than the single-disk baseline. The stripe evenly distributed IO (good NumberOfColumns setting).
No redundancy: The chosen Simple layout gives no fault tolerance. A single disk failure will render the volume unreadable unless restored from backup.


4 — Protection Options (Choose per Risk Profile)






























OptionDescriptionTrade-offKeep stripe + backupsUse daily Veeam backups. Good for workloads that tolerate data loss between backups.Simple, fast, max capacityMirrored stripe (RAID 10)Two-way mirror → tolerate 1 disk failure. Use Storage Spaces mirror layout.50% capacity lossParity (Storage Spaces)More efficient than mirror, tolerates 1–2 failures; slower writes.Write penaltySnapshots & frequent backupsCombine snapshots + Veeam for faster restores and logical corruption protection.Requires management

5 — Recommendations

If maximum performance is the priority and backups are reliable, keep stripe + Veeam daily backups.
If high availability is required (survive a disk failure with no restore needed), use mirrored stripe (two-way mirror) instead of Simple.
Test restores from Veeam regularly to validate RPO and RTO.
Monitor Azure VM and disk metrics (I/O queue length, latency, % CPU) and set alerts.
Consider different DiskSpd profiles (8K, 16K, varying outstanding IOs) to emulate your application load.


6 — Quick Reference Commands
Create Pool & Stripe (PowerShell)
powershellNew-StoragePool -FriendlyName "StripePool" -StorageSubsystemFriendlyName "Storage Spaces*" -PhysicalDisks (Get-PhysicalDisk -CanPool $True)
New-VirtualDisk -StoragePoolFriendlyName "StripePool" -FriendlyName "Disk-Strip-Demo" -Size 256GB -ResiliencySettingName Simple -NumberOfColumns 4
# Initialize & format: see full script above
DiskSpd Examples
powershell# Sequential write (5 min)
.\diskspd.exe -b64K -d300 -o4 -t4 -w100 -c10G D:\load.dat

# Random 4K mixed (5 min)
.\diskspd.exe -b4K -d300 -o8 -t8 -w50 -r -c10G D:\load.dat

7 — Summary



ItemDetailsConfiguration4 × Premium SSD LRS, 64 GiB each, striped (Simple) → ~256 GiB D:Performance (measured)~763 MB/s sequential write, ~9,751 IOPS (4KB random mixed)Data protectionDaily Veeam backups (current) — consider mirrored stripe for live redundancyAdviceStripe for performance + backups for recovery, or mirrored stripe if you need fault tolerance without relying solely on backups.

References

Azure Premium Storage Performance
DiskSpd Tool
