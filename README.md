# Azure VM 4-Disk Striped Volume Setup and Performance Testing

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

### Using Storage Spaces (PowerShell)
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



