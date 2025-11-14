# Azure IOPS & Throughput Hack: Beating VM Disk Limits with Striping
Azure Disk Striped Volume Setup and Performance Testing
---
## Overview
### Disk Striping (Introduction)

When a high-scale VM is attached with several **Premium Storage persistent disks**, the disks can be **striped together** to aggregate their **IOPS**, **bandwidth**, **storage capacity** and **Less price**.

**Why Striping Disks in Azure Gives You More IOPS and Throughput (And why your numbers beat the VM’s "max")**

When you attach a single Premium SSD to an Azure VM, that disk has its own limits:

-IOPS per disk

-Throughput per disk

-Size per disk

For example, each of your disks had:

-240 IOPS max

-50 MB/s throughput max

If you attach just one of those disks, your VM can only use that disk’s limits.

But Azure lets you attach multiple disks.
And each disk has its own independent performance budget.

### This is where striping comes in.
### What Disk Striping Actually Does

Striping means:

**You take multiple physical disks Combine them into one virtual volume And data is written across all disks in parallel**

Think of it like:

**Instead of 1 worker carrying all the load, you now have 4 workers lifting at the same time.**

So:

Item	Single Disk	4-Disk Stripe	What You See 

--IOPS	240, 240 × 4 = 960 IOPS, Your random test ~9,751 IOPS (CPU-limited test)

--Throughput	50 MB/s	50 × 4 = 200 MB/s	Your sequential test ~763 MB/s

--Capacity	64 GiB	64 × 4 = 256 GiB	D: shows 256 GiB

### Why Your Results Are Even Higher Than 960 IOPS and 200 MB/s
Because Azure has two limits:

1. Disk limits

– Each disk has its own performance.

2. VM limits

– The VM itself has a maximum IOPS + throughput budget it can push.

You combined:

4 disks × 240 IOPS = 960 theoretical disk IOPS

VM limit (for your size) is way higher than 1,000 IOPS

DiskSpd uses queue depth + multi-threading

Data spreads across all disks evenly (good column count)

**Result**: you achieved far above the raw disk spec, because the VM had unused headroom.
### Also: Sequential striping always boosts throughput massively
Azure Premium SSDs perform better under:

High queue depth (you used QD=4 / QD=8)

Sequential write tests (64 KB blocks)

Multi-threading

That's why you got:

~763 MB/s sequential write
(Expected: 200 MB/s… but real: 3.8× higher)

Azure disks have deeper throughput burst behavior and caching layers, and striping allows all disks to push in parallel.

### So What Did You Actually Achieve?

You essentially "cheated" Azure’s single-disk limits by using a legal, supported method:

--✔ Combined IOPS of all 4 disks

--✔ Combined throughput of all 4 disks

--✔ A single large fast volume (D:)

--✔ Better performance than a larger single-tier disk

--✔ Without paying for bigger Premium SSD tiers

--✔ Fully supported by Microsoft

### Why Azure Allows This 

Azure disks are designed to scale horizontally:

Azure does not want you to buy one big expensive disk.
They want you to scale out using multiple small disks.

That’s why striping is supported in:

-- SQL Server

-- Veeam proxy servers

-- Elasticsearch clusters

-- Backup repositories

-- TempDB volumes

-- Cache servers

-- Log volumes

Microsoft documents the behavior:

The IOPS and throughput of each disk add together when they are striped.

> **Reference:** [Azure Premium Storage Performance](https://learn.microsoft.com/en-us/azure/virtual-machines/premium-storage-performance)

---

## How should we acheive it?

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

## 1- Create the Striped Volume
### Add disk to your azure VM
1. Add 4 disk of Premium SSD LRS to your exisitng VM

| LUN | Disk Name | Storage Type       | Size (GiB) | Max IOPS | Max Throughput (MBps) | Encryption     | Host Caching |
|-----|-----------|--------------------|------------|----------|-------------------------|----------------|--------------|
| 1   | Disk02    | Premium SSD LRS    | 64         | 240      | 50                      | SSE with PMK   | Read/Write   |
| 2   | Disk03    | Premium SSD LRS    | 64         | 240      | 50                      | SSE with PMK   | Read/Write   |
| 3   | Disk04    | Premium SSD LRS    | 64         | 240      | 50                      | SSE with PMK   | Read/Write   |
| 0   | Disk05    | Premium SSD LRS    | 64         | 240      | 50                      | SSE with PMK   | Read/Write   |

### 1.1- Create a drive using added disk in windows VM

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
# 1.2- Verify configuration
```powershell
Get-VirtualDisk | Format-Table FriendlyName, ResiliencySettingName, Size, NumberOfColumns
Get-PhysicalDisk | Format-Table FriendlyName, OperationalStatus, CanPool, Size
```
## 2 - Performance Testing (DiskSpd)
DiskSpd is Microsoft’s command-line I/O workload generator.
Download: DiskSpd on GitHub
Place DiskSpd.exe in a folder (e.g., C:\Tools) and run PowerShell as Admin from there.
Use -c<size> to create test file if it doesn’t exist.

### 2.1 Sequential Write (5 Minutes)
```powershell
powershell.\diskspd.exe -b64K -d300 -o4 -t4 -w100 -c10G D:\load.dat
```
```powershel
ParameterValueBlock size64KDuration300s (5 min)Outstanding I/O per thread4Threads4Workload100% write, sequential
Measured Results (Sequential Write)
```
### Result:
#### MetricValueTotal 

| Item | Specification |
|------|---------------|
| **Throughput** | ~763 MB/sTotal  |
| **IOPS** | ~12,212Per-thread  |
| **balance** | ~190–192 MB/s (evenly distributed) |

 

### 2.2 Random 4KB Mixed (50% Read / 50% Write) – 5 Minutes
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
### 3— Interpretation & Key Notes

- **Aggregation**  
  · 4 disks × 240 IOPS = ~960 theoretical IOPS  
  · Measured 9,751 IOPS due to burst, caching, and multi-threaded test parameters

- **Sequential vs Random**  
  · Sequential = bandwidth-bound (~763 MB/s)  
  · Random = IOPS-bound (~9.7K IOPS)

- **Even I/O Distribution**  
  · Confirmed by per-thread balance → correct NumberOfColumns = 4

- **No Redundancy**  
  · Simple layout → 1 disk failure = data loss

### 4— Recommendations

**Performance Priority?**
→ Keep Simple stripe + daily Veeam backups
**High Availability Required?**
→ Use two-way mirror instead of Simple
Test Veeam Restores regularly (RPO/RTO validation)
Monitor in Azure Portal:
Disk IOPS / Throughput
VM CPU, I/O queue length
Set alerts for warnings

Customize DiskSpd tests to match your app:
Try 8K, 16K, 32K
Vary -o (outstanding I/O) and -t (threads)



### 5- Quick Reference Commands
Create Stripe (PowerShell)
```powershell
powershellNew-StoragePool -FriendlyName "StripePool" `
                -StorageSubsystemFriendlyName "Storage Spaces*" `
                -PhysicalDisks (Get-PhysicalDisk -CanPool $True)
```
```powershell
New-VirtualDisk -StoragePoolFriendlyName "StripePool" `
                -FriendlyName "Disk-Strip-Demo" `
                -Size 256GB `
                -ResiliencySettingName Simple `
                -NumberOfColumns 4
(Initialize/format: see full script above)
```
DiskSpd Tests
powershell# Sequential Write (5 min)
.\diskspd.exe -b64K -d300 -o4 -t4 -w100 -c10G D:\load.dat

# Random 4K Mixed (5 min)
```powershell
.\diskspd.exe -b4K -d300 -o8 -t8 -w50 -r -c10G D:\load.dat
```
### 6- Summary
## Summary & Recommendations

This setup demonstrates how to maximize **IOPS and throughput** on Azure using multiple Premium SSDs in a striped configuration.

| Item               | Details |
|-------------------|---------|
| **Configuration** | 4 × Premium SSD LRS (64 GiB each) → striped → 256 GiB D: volume |
| **Performance**   | Sequential Write: ~763 MB/s<br>Random 4 KB Mixed Read/Write: ~9,751 IOPS |
| **Data Protection** | Daily Veeam backups (current). For live fault tolerance, consider mirrored stripe (RAID 10) |
| **Recommendation** | - Use striping for maximum performance when backups are in place<br>- Use mirrored stripe for fault tolerance if backup solutions are not available<br>- Monitor disk health and Azure VM IOPS/throughput metrics regularly<br>- Test with various block sizes (4 KB, 8 KB, 16 KB) to simulate real-world workloads |
| **Key Notes** | - Striping aggregates disk IOPS and throughput, often exceeding single-disk limits<br>- Simple layout has no redundancy: 1 disk failure = total data loss<br>- Sequential workloads benefit most from striping, random workloads benefit from higher IOPS and queue depth |

> **Insight:** This approach effectively "cheats" Azure disk limits legally by combining multiple disks, providing higher throughput and IOPS than a single disk of the same size while staying fully supported.


## References

Azure Premium Storage Performance
DiskSpd Documentation
Storage Spaces Overview
