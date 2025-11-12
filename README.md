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
New-Partition -DiskNumber <diskNumber> -UseMaximumSize -DriveLetter D | Format-Volume -FileSystem NTFS -NewFileSystemLabel "D_Stripe"```

##Step 2: Create a Storage Pool
```New-StoragePool -FriendlyName "StripePool" -StorageSubsystemFriendlyName "Storage Spaces*" -PhysicalDisks (Get-PhysicalDisk -CanPool $True)```

##Step 3: Create Striped Virtual Disk
```New-VirtualDisk -StoragePoolFriendlyName "StripePool" -FriendlyName "Disk-Strip-Demo" -Size 256GB -ResiliencySettingName Simple -NumberOfColumns 4```

##Step 4: Initialize and Format the Disk
```Initialize-Disk -VirtualDisk (Get-VirtualDisk -FriendlyName "Disk-Strip-Demo") -PartitionStyle GPT
New-Partition -DiskNumber <diskNumber> -UseMaximumSize -DriveLetter D | Format-Volume -FileSystem NTFS -NewFileSystemLabel "D_Stripe"```

Tip: "Simple" layout = no redundancy. For critical data, consider a mirrored stripe (RAID 10 equivalent) to survive disk failures.

2. Performance Testing

We used DiskSpd to test sequential and random I/O performance.

Sequential Write Test
```.\diskspd.exe -b64K -d300 -o4 -t4 -w100 -c10G D:\load.dat```


Block size: 64 KB

Duration: 5 minutes

Threads: 4

Sequential write only

Results (Sequential Write):

Total throughput: ~763 MB/s

Total IOPS: ~12,212

Random 4 KB Mixed Read/Write Test
.\diskspd.exe -b4K -d300 -o8 -t8 -w50 -r -c10G D:\load.dat


Block size: 4 KB

Read/Write ratio: 50/50

Random I/O

Threads: 8

Results (Random 4 KB Read/Write):

Metric	MB/s	IOPS
Read	19.04	4,875
Write	19.05	4,876
Total	38.09	9,751

The results show excellent sequential throughput and strong random IOPS performance for a 4-disk striped setup.

3. Recommendations

Backup Strategy: Daily Veeam backups cover data loss risk for striped volume.

High Availability: For critical workloads, use a mirrored stripe (RAID 10) to survive single disk failure.

Monitoring: Track disk health, I/O queue length, and VM metrics in Azure.

Workload Testing: Consider testing with 8 KB or 16 KB block sizes to simulate real-world applications.

4. Summary

Configuration: 4 × Premium SSDs, 256 GiB, striped (simple)

Performance: ~763 MB/s sequential, ~9,751 random 4 KB IOPS

Data Protection: Daily Veeam backups, optional mirrored stripe for fault tolerance

Use Case: High-performance workloads where throughput and IOPS are critical and backups provide protection

