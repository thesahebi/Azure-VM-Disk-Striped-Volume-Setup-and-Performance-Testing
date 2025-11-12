# Azure VM 4-Disk Striped Volume Setup and Performance Testing

---

## Disk Striping (Introduction)

When a high-scale VM is attached with several **Premium Storage persistent disks**, the disks can be **striped together** to aggregate their **IOPS**, **bandwidth**, and **storage capacity**.

- **Windows:** Use **Storage Spaces** to stripe disks. Configure **one column per physical disk** to ensure even distribution of I/O across disks.  
  - Server Manager UI supports up to **8 columns**  
  - For more disks, use **PowerShell** and set `NumberOfColumns` equal to the number of disks.

- **Linux:** Use the **`mdadm`** utility to build RAID/stripe sets.

> **Reference:** [Azure Premium Storage Performance](https://learn.microsoft.com/en-us/azure/virtual-machines/premium-storage-performance)

---

## Overview

This **README** provides a complete, step-by-step guide to:

1. Create a **high-performance 4-disk striped volume (D:)** on a **Windows Azure VM** using **Storage Spaces**
2. Test performance using **DiskSpd**
3. Summarize measured results and best practices

**Example configuration:**  
**4 × Premium SSD LRS, 64 GiB each**

---

## Environment

| Item | Specification |
|------|---------------|
| **VM** | Windows Server (Azure) |
| **Disks Attached** | 4 × Premium SSD LRS, 64 GiB each |
| **Max IOPS per Disk** | 240 |
| **Max Throughput per Disk** | 50 MB/s |
| **Target Volume (Striped D:)** | ~256 GiB usable (Simple/Striped) |

---

## 1 — Create the Striped Volume

### Option A — GUI (Server Manager)

1. Open **Server Manager** → **File and Storage Services** → **Storage Pools**
2. Click **New Storage Pool**
   - Select all **4 available disks**
   - Name: `StripePool`
3. After pool creation → **Tasks** → **New Virtual Disk**
   - **Storage Pool:** `StripePool`
   - **Layout:** `Simple (No resiliency)` *(this is striping)*
   - **Provisioning:** `Fixed` *(recommended for performance)*
   - **Enclosure awareness:** Enable only if disks are in separate enclosures
   - **Number of columns:** `4` *(one per disk)*
4. Create the virtual disk → **Initialize** → **GPT**
5. **New Partition** → Assign drive letter **D:**
6. **Format** → `NTFS` (or `ReFS`) → Label: `D_Stripe`

> **Tip:** `Simple` layout = **no redundancy**. For fault tolerance, use **Mirror** instead.

---

### Option B — PowerShell *(Recommended for Reproducibility)*

Run as **Administrator**:

```powershell
# 1. Verify available physical disks
Get-PhysicalDisk | Sort-Object FriendlyName | Format-Table FriendlyName, OperationalStatus, MediaType, Size

# 2. Create storage pool (uses all eligible disks)
New-StoragePool -FriendlyName "StripePool" `
                -StorageSubsystemFriendlyName "Storage Spaces*" `
                -PhysicalDisks (Get-PhysicalDisk -CanPool $True)

# 3. Create striped virtual disk (Simple = stripe), 4 columns
New-VirtualDisk -StoragePoolFriendlyName "StripePool" `
                -FriendlyName "Disk-Strip-Demo" `
                -Size 256GB `
                -ResiliencySettingName Simple `
                -NumberOfColumns 4

# 4. Initialize and format the virtual disk
$vd = Get-VirtualDisk -FriendlyName "Disk-Strip-Demo"
$disk = $vd | Get-Disk
Initialize-Disk -Number $disk.Number -PartitionStyle GPT
New-Partition -DiskNumber $disk.Number -UseMaximumSize -DriveLetter D | 
    Format-Volume -FileSystem NTFS -NewFileSystemLabel "D_Stripe" -Confirm:$false
```
##2 — Performance Testing (DiskSpd)
DiskSpd is Microsoft’s command-line I/O workload generator.
Download: DiskSpd on GitHub
Place DiskSpd.exe in a folder (e.g., C:\Tools) and run PowerShell as Admin from there.
Use -c<size> to create test file if it doesn’t exist.

2.1 Sequential Write (5 Minutes)
```powershell 
powershell.\diskspd.exe -b64K -d300 -o4 -t4 -w100 -c10G D:\load.dat
```

ParameterValueBlock size64KDuration300s (5 min)Outstanding I/O per thread4Threads4Workload100% write, sequential
Measured Results (Sequential Write)

MetricValueTotal Throughput~763 MB/sTotal IOPS~12,212Per-thread balance~190–192 MB/s (evenly distributed)

2.2 Random 4KB Mixed (50% Read / 50% Write) – 5 Minutes
```powershell 
powershell.\diskspd.exe -b4K -d300 -o8 -t8 -w50 -r -c10G D:\load.dat
```

ParameterValueBlock size4KDuration300sOutstanding I/O per thread8Threads8Workload50% read / 50% write, random
Measured Results (Random 4KB Mixed)

MetricMB/sIOPSRead19.044,875Write19.054,876Total38.099,751
Strong aggregated IOPS — nearly 4× single disk performance.

Cleanup
```powershell 
powershellRemove-Item D:\load.dat -Force
```

##3 — Interpretation & Key Notes

ObservationExplanationAggregation4 disks × 240 IOPS = ~960 theoretical IOPS → measured 9,751 due to burst, caching, and test parametersSequential vs RandomSequential = bandwidth-bound (~763 MB/s)
Random = IOPS-bound (~9.7K IOPS)Even I/O distributionConfirmed by per-thread balance → correct NumberOfColumns = 4No redundancySimple layout → 1 disk failure = data loss

##4 — Data Protection Options

StrategyDescriptionCapacityFault TolerancePerformanceStripe + Backups (Current)Daily Veeam backups100%None (restore from backup)MaxMirrored Stripe (RAID-10)Two-way mirror in Storage Spaces50%1 disk failureHighParityStorage Spaces parity~75%1–2 failuresSlower writesSnapshots + BackupsAzure or ReFS snapshots + Veeam100%Logical recoveryFast restore

5 — Recommendations

Performance Priority?
→ Keep Simple stripe + daily Veeam backups
High Availability Required?
→ Use two-way mirror instead of Simple
Test Veeam Restores regularly (RPO/RTO validation)
Monitor in Azure Portal:
Disk IOPS / Throughput
VM CPU, I/O queue length
Set alerts for warnings

Customize DiskSpd tests to match your app:
Try 8K, 16K, 32K
Vary -o (outstanding I/O) and -t (threads)

##6 — Quick Reference Commands
Create Stripe (PowerShell)
```powershell
New-StoragePool -FriendlyName "StripePool" `
                -StorageSubsystemFriendlyName "Storage Spaces*" `
                -PhysicalDisks (Get-PhysicalDisk -CanPool $True)

New-VirtualDisk -StoragePoolFriendlyName "StripePool" `
                -FriendlyName "Disk-Strip-Demo" `
                -Size 256GB `
                -ResiliencySettingName Simple `
                -NumberOfColumns 4

```
##DiskSpd Tests
```powershell
# Sequential Write (5 min)
.\diskspd.exe -b64K -d300 -o4 -t4 -w100 -c10G D:\load.dat

# Random 4K Mixed (5 min)
.\diskspd.exe -b4K -d300 -o8 -t8 -w50 -r -c10G D:\load.dat
```
##7 — Summary
Item,Details
Configuration,"4 × Premium SSD LRS (64 GiB), striped → ~256 GiB D:"
Performance,"~763 MB/s sequential write
~9,751 IOPS (4KB random mixed)"
Data Protection,"Daily Veeam backups (current)
→ Consider mirrored stripe for live redundancy"
Recommendation,"Stripe for speed + backups for recovery
OR Mirror for fault tolerance"

##References
Azure Premium Storage Performance
DiskSpd Documentation
Storage Spaces Overview
