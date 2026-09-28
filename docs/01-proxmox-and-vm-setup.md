# 01 — Proxmox and VM Setup

## Goal

Stand up a Kali Linux attacker VM and a Metasploitable 2 target VM on a single Proxmox VE host.

## Kali Linux VM (VM 100)

Created via the standard Proxmox "Create VM" wizard, ISO uploaded to local storage. Network device was initially set to the default bridge (`vmbr0`) so the VM has internet access during install and for tool updates.

**Decision point:** whether to attach Kali to the default bridge or wait to build a firewall (OPNsense) first.

- Kali needs internet access to complete installation and updates.
- Proxmox networking is not fixed at creation time — the bridge on a VM's NIC can be changed later from Hardware → Network Device without reinstalling.
- Conclusion: attach to `vmbr0` now, migrate to a segmented zone once a firewall exists. The only exception is intentionally vulnerable targets, which should never touch `vmbr0`.

## Metasploitable 2 VM (VM 200)

Metasploitable 2 ships as a VMware disk (`.vmdk`), not an installable ISO, so the process is: create an empty VM, then import the existing disk into it.

```bash
# Copy the disk to the Proxmox host
scp Metasploitable.vmdk root@<proxmox-ip>:/root/

# Create an empty VM shell
qm create 200 --name metasploitable2 --memory 512 --cores 1 --ostype l26 --net0 e1000,bridge=vmbr1

# Import the disk into Proxmox-managed storage
qm importdisk 200 /root/Metasploitable.vmdk local-lvm
qm set 200 --ide0 local-lvm:vm-200-disk-0
qm set 200 --boot order=ide0
```

Network card type is `e1000` rather than `virtio`, since Metasploitable runs an old Ubuntu 8.04 kernel that handles the older emulated NIC more reliably. Disk bus is `ide` for the same reason.

---

## Issue 1 — Wrong disk volume name on `local` storage

**Command that failed:**
```bash
qm set 200 --ide0 local:vm-200-disk-0
```
**Error:**
```
unable to parse directory volume name: vm-200-disk-0
```

**Cause:** the target storage (`local`) is a **directory-type** storage, not LVM. Directory storage names disks differently — as a file inside a per-VM folder, e.g. `local:200/vm-200-disk-0.raw` — not the flat `local-lvm:vm-200-disk-0` naming used by LVM-backed storage.

**Fix:**
```bash
# Find the real disk path/name
qm config 200
# → shows: unused0: local:200/vm-200-disk-0.raw

# Attach it using the exact name
qm set 200 --ide0 local:200/vm-200-disk-0.raw
qm set 200 --boot order=ide0
```

**Lesson:** the correct volume identifier depends on the storage *type*, not just its name. Always confirm with `qm config <id>` (or the Hardware tab, which lists it as "Unused Disk") rather than assuming the naming convention.

---

## Issue 2 — VM fails to start: bridge does not exist

**Error:**
```
bridge 'vmbr1' does not exist
kvm: -netdev type=tap,id=net0,ifname=tap200i0,script=/usr/libexec/qemu-server/pve-bridge,...: network script /usr/libexec/qemu-server/pve-bridge failed with status 512
TASK ERROR: start failed: QEMU exited with code 1
```

**Cause:** VM 200 was created referencing `vmbr1` as its network bridge before that bridge actually existed on the host. Proxmox doesn't validate this at VM-creation time — it only fails at boot.

**Fix:** create the bridge (see [`02-network-segmentation.md`](02-network-segmentation.md)), then start the VM again. No change to the VM config was needed once the bridge existed.

**Lesson:** plan the network before referencing it in VM configs, or double-check `ip link show <bridge>` on the host before relying on it in a VM definition.

---

## Issue 3 — Snapshot fails on the Metasploitable disk

**Symptom:** "Take Snapshot" fails / is unavailable for VM 200.

**Cause:** the imported disk was in **raw** format on `local` (directory) storage. Raw disks on directory-type storage don't support Proxmox's live snapshot mechanism — that needs `qcow2` format, or a storage backend like LVM-thin/ZFS that supports snapshots natively.

**Fix — convert the disk to qcow2:**
```bash
qm move-disk 200 ide0 local --format qcow2 --delete 1
```
(GUI equivalent: VM → Hardware → select disk → Disk Action → Move Storage → target `local`, format `qcow2`, tick "Delete source".)

After conversion, snapshotting worked normally (name: `clean`, taken before any exploitation).

**Lesson:** disk format and storage backend both affect which Proxmox features (snapshots, live migration) are available — not just capacity/performance. Worth checking storage type before importing, though converting after the fact is a simple one-command fix.

---

## Result

- VM 100 (Kali): running, `vmbr0` for internet, later given a second NIC on `vmbr1` for the lab network.
- VM 200 (Metasploitable): running on `vmbr1` only, disk converted to `qcow2`, snapshot `clean` taken pre-exploitation.
