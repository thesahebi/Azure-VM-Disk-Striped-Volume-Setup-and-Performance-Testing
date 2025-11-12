# Azure VM 4-Disk Striped Volume Setup and Performance Testing

## Disk Striping Overview
When a high-scale VM is attached with several **Premium Storage persistent disks**, the disks can be **striped together** to aggregate their IOPs, bandwidth, and storage capacity.

- **Windows:** Use **Storage Spaces** to stripe disks. Configure one column per disk to ensure optimal performance.  
  - Server Manager UI supports up to 8 columns per striped volume.  
  - For more than 8 disks, use PowerShell and set `NumberOfColumns` equal to the number of disks.  
- **Linux:** Use the **MDADM** utility to stripe disks together.  

> Reference: [Azure Premium Storage Performance](https://learn.microsoft.com/en-us/azure/virtual-machines/premium-storage-performance)

---

## Overview
This guide explains how to create a high-performance **4-disk striped volume (D:)** on a Windows Azure VM using **Storage Spaces**, and how to test its performance using **DiskSpd**.  
The setup maximizes **IOPS and throughput** while leveraging Azure Premium SSDs. Daily backups using **Veeam Backup & Replication** provide data protection.

---

## 1. Environment
- **VM:** Windows Server on Azure  
- **Disks:** 4 × Premium SSD LRS, 64 GiB each  
- **Max IOPS per disk:** 240  
- **Max throughput per disk:** 50 MB/s  
- **Total D: volume size:** 256 GiB (striped)

---

## 2. Creating the Striped Volume

### Option 1: Using Storage Spaces GUI

1. Open **Server Manager → File and Storage Services → Storage Pools**.  
2. Click **New Storage Pool**, select the available disks, and give it a friendly name (e.g., `StripePool`).  
3. Create a **New Virtual Disk**:
   - Layout: **Simple (No redundancy)**  
   - Provisioning: Thin or Fixed  
   - Enclosure Awareness: Enable  
   - Number of columns: 4  
4. Initialize the disk, create a new partition, and assign drive letter **D:**  
   - Format with **NTFS**  
   - Set file system label: `D_Stripe`

### Option 2: Using PowerShell
```powershell
# List all physical disks
Get-PhysicalDisk | Sort FriendlyName | Format-Table FriendlyName, OperationalStatus, MediaType, Size

# Create storage pool with all four disks
New-StoragePool -FriendlyName "StripePool" -StorageSubsystemFriendlyName "Storage Spaces*" -PhysicalDisks (Get-PhysicalDisk -CanPool $True)

# Create striped virtual disk
New-VirtualDisk -StoragePoolFriendlyName "StripePool" -FriendlyName "Disk-Strip-Demo" -Size 256GB -ResiliencySettingName Simple -NumberOfColumns 4

# Initialize, create partition, and format
Initialize-Disk -VirtualDisk (Get-VirtualDisk -FriendlyName "Disk-Strip-Demo") -PartitionStyle GPT
New-Partition -DiskNumber <diskNumber> -UseMaximumSize -DriveLetter D | Format-Volume -FileSystem NTFS -NewFileSystemLabel "D_Stripe"
