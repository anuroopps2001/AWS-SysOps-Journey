## 🚀 EBS Snapshot Restore & EC2 Storage Deep-Dive — Complete Guide

### 1. EC2 Metadata (IMDSv2)
IMDSv2 Token
```bash
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
```

Metadata Retrieval
```bash
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/
```

User Data
```bash
curl http://169.254.169.254/latest/user-data
```

Key Points

- Metadata is only available at 169.254.169.254

- Not public, not tied to public IP

- Works only inside EC2

### 2. Creating AMIs
AMI Contains

- Reference to snapshot(s)

- Block device mapping

- Root volume configuration

- Boot configuration

Important

- Creating an AMI always creates at least one snapshot

- New EC2 created from AMI gets a fresh root EBS volume

- You can modify the size/type of the root volume (bigger is fine)

### 3. Understanding Snapshots
Snapshot = Backup of EBS Volume

- Point-in-time backup

- Incremental

- Stored in S3 internally

Used for:

 - Backup

 - Restore

 - Migration across AZ/region

 - Launch new volumes

 - AMI creation

#### Snapshot ≠ Volume

Snapshot = template

Volume = actual disk

### 4. Creating Snapshots

#### Manual Snapshot

From EC2 → Volumes → Create Snapshot

#### Automatic Snapshot

Created automatically when you:

- Create an AMI

- Use Lifecycle Manager

### ## 5. Restoring Snapshots
Create Volume From Snapshot

- Choose correct AZ

- Create volume (size must be ≥ snapshot size)

- Attach to EC2

#### Device Name Mapping

AWS Console:
```bash
/dev/sdf
```

Linux Nitro Instances:
```bash
/dev/nvme1n1
```
This is `expected`.

### 6. Mounting Restored Snapshot Volumes
Correct Procedure

List devices:
```bash
lsblk
```
You will see:
```bash
nvme1n1      (disk)
└─nvme1n1p1  (partition with filesystem)
```

Mount partition (not disk):
```bash
sudo mount /dev/nvme1n1p1 /<any_directory>
```

If filesystem unknown:
```bash
sudo file -s /dev/nvme1n1p1
sudo blkid /dev/nvme1n1p1
```

### 7. XFS Duplicate UUID Issue (Advanced — Real AWS Issue)
If the instance was created from an AMI, and you also attach a snapshot created from the same root volume, both disks will have the same XFS UUID.

❗ XFS forbids mounting two filesystems with the same UUID

You will see:
```bash
mount: wrong fs type, bad superblock
```
Confirm Duplicate UUID:
```bash
sudo xfs_admin -u /dev/nvme0n1p1
sudo xfs_admin -u /dev/nvme1n1p1
```

#### Fix (Recommended)

Generate a new UUID for the snapshot:
```bash
sudo xfs_admin -U generate /dev/nvme1n1p1
```

Then mount:
```bash
sudo mount /dev/nvme1n1p1 /<any_directory>
```

### 8. Root Disk vs Snapshot Disk
Typical after restore:
```bash
nvme0n1     → root disk of EC2
nvme1n1     → restored snapshot disk
```
Mount points:
```bash
/           → nvme0n1p1
/<any_directory>    → nvme1n1p1
```

This is correct.

### 9. Unmounting & Cleanup
Unmounting:
```bash
sudo umount /<any_directory>
```

Detach volume:

- EC2 → Storage → Detach volume

Delete to avoid cost:

- Volumes → Delete

- Snapshots → Delete

### 10. Full Workflow Summary (Start to Finish)

- Launch EC2 instance

- Install software (e.g., Apache)

- Create AMI → snapshot is generated

- Launch new EC2 from AMI

- Create volume from manual/AMI snapshot

- Attach restored volume

- Identify device mapping (nvme1n1)

- Detect partition (nvme1n1p1)

- Fix XFS UUID conflict

- Mount and verify data

- Unmount + detach + cleanup
