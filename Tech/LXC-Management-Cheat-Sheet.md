# 📦 LXC Management Cheat Sheet

## 🔧 Container Basics

```bash
# List all containers
pct list

# Show detailed info
pct config <CTID>
pct status <CTID>
```

---

## ▶️ Lifecycle

```bash
# Create container
pct create <CTID> <TEMPLATE> \
  -storage <STORAGE> -hostname <NAME> -memory <MB> \
  -net0 name=eth0,bridge=vmbr0,ip=dhcp

# Start / Stop / Reboot
pct start <CTID>
pct stop <CTID>
pct reboot <CTID>

# Shutdown gracefully
pct shutdown <CTID>

# Destroy container
pct destroy <CTID>
```

---

## 🔑 Access

```bash
# Attach to console (interactive shell as root)
pct enter <CTID>

# Run command inside container
pct exec <CTID> -- <command>

# Example: run apt update inside CT
pct exec <CTID> -- apt update
```

---

## 💾 Storage & Disks

```bash
# Resize root disk
pct resize <CTID> rootfs +10G

# Mount additional storage
pct set <CTID> -mp0 /host/path,mp=/container/path

# Check usage
pct df <CTID>
```

---

## 🌐 Networking

```bash
# Add network interface
pct set <CTID> -net0 name=eth0,bridge=vmbr0,ip=dhcp

# Assign static IP
pct set <CTID> -net0 name=eth0,bridge=vmbr0,ip=192.168.1.50/24,gw=192.168.1.1

# Limit bandwidth
pct set <CTID> -net0 rate=10
```

---

## 🔒 Permissions & Features

```bash
# Allow nested containers
pct set <CTID> -features nesting=1

# Allow fuse mounts
pct set <CTID> -features fuse=1

# Enable privileged container
pct set <CTID> -unprivileged 0
```

---

## 📦 Backups & Snapshots

```bash
# Backup container
vzdump <CTID> --storage <STORAGE> --mode snapshot

# Restore container
pct restore <NEW_CTID> /path/to/vzdump-lxc-<CTID>.tar.zst

# Snapshot
pct snapshot <CTID> <SNAPNAME>

# Rollback
pct rollback <CTID> <SNAPNAME>
```

---

## 🛠️ Useful Tricks

```bash
# Copy file into container
pct push <CTID> /local/file /remote/path

# Copy file from container
pct pull <CTID> /remote/file /local/path

# Temporary resource limit
pct set <CTID> -memory 2048 -cores 2

# Remove unused CT configs
ls /etc/pve/lxc/
```

---

⚡ **Pro Tips**

* Use **unprivileged containers** whenever possible for security.
* Store templates in `local` or `local-lvm` (`pveam update` to refresh list).
* Combine with **Proxmox storage pools** (ZFS/CEPH) for snapshots/replication.
* Use `pct set` liberally — it’s safe to adjust configs on the fly.


