# 🗒️ ZFS Management Cheat Sheet

---

## 📦 Pools

**Create a pool (mirror, RAIDZ, stripe):**

```bash
# Stripe (fast, no redundancy)
zpool create poolname /dev/sdX /dev/sdY

# Mirror (RAID1-like)
zpool create poolname mirror /dev/sda /dev/sdb

# RAIDZ1 (like RAID5)
zpool create poolname raidz1 /dev/sd[a-c]

# RAIDZ2 (like RAID6)
zpool create poolname raidz2 /dev/sd[a-e]
```

**Add vdev to existing pool:**

```bash
zpool add poolname mirror /dev/sdX /dev/sdY
```

**Replace a failed disk:**

```bash
zpool replace poolname old-disk new-disk
```

**Export/import pool:**

```bash
zpool export poolname
zpool import poolname
```

---

## 🔍 Pool Status & Health

**Check health:**

```bash
zpool status
```

**Detailed I/O:**

```bash
zpool iostat -v 1
```

**Show available pools:**

```bash
zpool list
```

---

## 🧩 Datasets & Filesystems

**Create dataset:**

```bash
zfs create poolname/dataset
```

**Set mountpoint:**

```bash
zfs set mountpoint=/mnt/data poolname/dataset
```

**Show datasets:**

```bash
zfs list
```

**Destroy dataset:**

```bash
zfs destroy poolname/dataset
```

---

## 🖼️ Snapshots & Clones

**Take snapshot:**

```bash
zfs snapshot poolname/dataset@snapname
```

**List snapshots:**

```bash
zfs list -t snapshot
```

**Rollback to snapshot:**

```bash
zfs rollback poolname/dataset@snapname
```

**Clone snapshot (writable):**

```bash
zfs clone poolname/dataset@snapname poolname/clone
```

---

## 🧹 Scrubbing & Maintenance

**Start scrub:**

```bash
,.l
```

**Pause scrub (OpenZFS 2.1+):**

```bash
zpool scrub -p poolname
```

**Stop scrub:**

```bash
zpool scrub -s poolname
```

**View scrub progress:**

```bash
zpool status poolname
```

**Clear errors after replacing/rebuilding:**

```bash
zpool clear poolname
```

---

## ⚙️ Properties & Tuning

**View properties:**

```bash
zfs get all poolname/dataset
```

**Set compression:**

```bash
zfs set compression=lz4 poolname/dataset
```

**Set quota:**

```bash
zfs set quota=100G poolname/dataset
```

**Enable deduplication (⚠️ RAM heavy, use cautiously):**

```bash
zfs set dedup=on poolname/dataset
```

---

## 🛠️ Recovery & Troubleshooting

**Import pool after crash:**

```bash
zpool import -f poolname
```

**Import with recovery options:**

```bash
zpool import -F poolname
```

**Offline a disk (for replacement):**

```bash
zpool offline poolname /dev/sdX
```

**Online a disk:**

```bash
zpool online poolname /dev/sdX
```

---

## 🕒 Useful One-Liners

* **Check scrub history:**

  ```bash
  zpool history | grep scrub
  ```

* **Monitor ARC (Linux):**

  ```bash
  arcstat.py 1
  ```

  (install `arcstat` from OpenZFS tools)

* **Estimate space (including snapshots):**

  ```bash
  zfs list -o space
  ```
