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

##2 PowerShell *(recommended for reproducibility)*
DiskSpd is a Microsoft I/O testing tool. Place DiskSpd.exe in a folder on the VM and run commands from that folder (PowerShell as Admin). Use -c<size> to create the test file if it doesn't exist****

###2.1 Sequential Write (5 minutes)


