# Full‑Disk Backup & Restore Ritual  
### Using `groot`, Btrfs Snapshots, gzip Compression, and SHA‑256 Verification

---

## 🏷️ Credits

**Author:** Damian A. James Williamson (Willtech)  
**Privilege wrapper:** `groot` by Willtech  
**AI collaborator:** Microsoft Copilot (technical synthesis, procedural structuring)

---

## 📘 Overview

This document describes a deterministic, reproducible, and compression‑assisted workflow for:

- Creating a consistent full‑disk backup of `/dev/nvme0n1`
- Streaming it directly onto `/dev/sda` using gzip
- Verifying the clone using SHA‑256 block hashing
- Restoring from `/dev/sda` back to the NVMe disk
- Managing the lifecycle of a Btrfs snapshot used for consistency

All privileged operations use the Willtech privilege wrapper:  
**[`groot`](https://github.com/Willtech/groot)**

This workflow is suitable for disaster recovery, disk migration, lineage‑clean archival, and reproducible system state capture.

---

# 1. Create a Btrfs Snapshot (Optional but Recommended)

A read‑only snapshot ensures the backup is consistent even while the system is running.

```bash
groot btrfs subvolume snapshot -r / /.snap-nvme-backup
```

If you want to inspect the snapshot:

```bash
groot mkdir -p /mnt/snap
groot mount -o ro,subvol=.snap-nvme-backup / /mnt/snap
```

---

# 2. Clone `/dev/nvme0n1` → `/dev/sda` Using gzip Compression

This pipeline performs a compressed, block‑accurate clone directly between disks:

```bash
groot dd if=/dev/nvme0n1 bs=1M status=progress | gzip -1 | groot dd of=/dev/sda
```

### What this does

- Reads the entire NVMe disk  
- Compresses the stream (`gzip -1` = fastest compression)  
- Immediately decompresses it into `/dev/sda`  
- No intermediate `.img.gz` file  
- Fully deterministic byte‑for‑byte clone  

This is the cleanest possible “compressed disk teleport”.

---

# 3. Verify Clone Integrity Using SHA‑256

Even though `/dev/nvme0n1` is live, hashing raw blocks is safe and deterministic.

### Hash the source disk:

```bash
groot dd if=/dev/nvme0n1 bs=1M status=none | sha256sum
```

### Hash the clone:

```bash
groot dd if=/dev/sda bs=1M status=none | sha256sum
```

If the hashes match, the clone is perfect.

---

# 4. Restore `/dev/sda` → `/dev/nvme0n1`

You can restore using the same compressed pipeline:

```bash
groot dd if=/dev/sda bs=1M status=progress | gzip -1 | groot dd of=/dev/nvme0n1
```

Or restore without compression:

```bash
groot dd if=/dev/sda of=/dev/nvme0n1 bs=1M status=progress
```

Both produce a byte‑accurate restoration of the entire disk.

---

# 5. Clean Up the Btrfs Snapshot

If you mounted it:

```bash
groot umount /mnt/snap
```

Delete the snapshot:

```bash
groot btrfs subvolume delete /.snap-nvme-backup
```

Confirm removal:

```bash
groot btrfs subvolume list /
```

Snapshot lifecycle complete.

---

# 6. Full Ritual Summary

```
# Optional: create Btrfs snapshot
groot btrfs subvolume snapshot -r / /.snap-nvme-backup

# Clone NVMe → SATA with gzip compression
groot dd if=/dev/nvme0n1 bs=1M status=progress | gzip -1 | groot dd of=/dev/sda

# Verify clone integrity
groot dd if=/dev/nvme0n1 bs=1M status=none | sha256sum
groot dd if=/dev/sda       bs=1M status=none | sha256sum

# Restore (if needed)
groot dd if=/dev/sda bs=1M status=progress | gzip -1 | groot dd of=/dev/nvme0n1

# Clean up snapshot
groot btrfs subvolume delete /.snap-nvme-backup
```

---

# 7. The Fast Way

```
# Optional: create Btrfs snapshot
groot btrfs subvolume snapshot -r / /.snap-nvme-backup

# (Optional) Mount snapshot for inspection
# NOTE: Snapshots are NOT mounted automatically. Depends on filesystem use CoPilot.
groot mkdir -p /mnt/snap
groot mount -o ro,subvol=.snap-nvme-backup /dev/nvme0n1p1 /mnt/snap

# Inspect snapshot contents under /mnt/snap
# (e.g., ls /mnt/snap, check configs, verify state, etc.)

# Clone NVMe → SATA using conv=sync,noerror
groot dd if=/dev/nvme0n1 of=/dev/sda bs=1M conv=sync,noerror status=progress

# Verify clone integrity
groot dd if=/dev/nvme0n1 bs=1M status=none | sha256sum
groot dd if=/dev/sda       bs=1M status=none | sha256sum

# Restore (if needed)
groot dd if=/dev/sda of=/dev/nvme0n1 bs=1M conv=sync,noerror status=progress

# Clean up snapshot
groot btrfs subvolume delete /.snap-nvme-backup
```

---

## 📎 Related Projects

- **SecureWipe** — [https://github.com/Willtech/SecureWipe](https://github.com/Willtech/SecureWipe)  
- **groot (Privilege Wrapper)** — [https://github.com/Willtech/groot](https://github.com/Willtech/groot)  
---

## 📜 License

This document may be included in Willtech repositories under the same licensing terms as the parent project.
