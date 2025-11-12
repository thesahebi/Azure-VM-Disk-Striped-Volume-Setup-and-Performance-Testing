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
