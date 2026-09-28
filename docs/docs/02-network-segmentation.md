# 02 — Network Segmentation

## Goal

Isolate the vulnerable Metasploitable target from the home network, so scanning and exploiting it can never touch anything else on the LAN or the internet, while still allowing the Kali attacker VM to reach both.

## Concept

Proxmox bridges (`vmbrX`) function as virtual switches. A bridge only reaches the physical network if it has a **physical port** attached to it:

- `vmbr0` — bridged to the host's real network adapter → reaches the home network and internet.
- `vmbr1` — created with **no physical port** → an isolated switch that only connects VMs plugged into it to each other, with no path out.

This means isolation here isn't enforced by firewall rules — it's structural. There is simply no wire connecting the isolated network to anything else.

## Step 1 — Create the isolated bridge

**GUI:** Node (`pve`) → System → Network → Create → Linux Bridge → name `vmbr1`, leave Bridge Ports and IPv4/Gateway empty → Create → **Apply Configuration** (the bridge is inactive until this step).

**CLI equivalent:**
```bash
cat >> /etc/network/interfaces <<'EOF'

auto vmbr1
iface vmbr1 inet manual
    bridge-ports none
    bridge-stp off
    bridge-fd 0
EOF
ifreload -a
ip link show vmbr1
```

`vmbr0` was left untouched throughout, since it carries the host's own management connection — misconfiguring it risks losing access to the Proxmox GUI entirely.

## Step 2 — Attach VMs to the bridge

- Metasploitable (VM 200): single NIC, `e1000`, on `vmbr1` only.
- Kali (VM 100): a **second** NIC added on `vmbr1`, in addition to its existing NIC on `vmbr0`. This gives Kali one leg on the home network (for internet/tooling) and one leg on the isolated lab network (to reach the target).

```bash
qm set 100 --net1 virtio,bridge=vmbr1
```

## Step 3 — Assign IP addresses

Since `vmbr1` has no DHCP server (nothing is connected to hand out addresses), every device on it needs a manually assigned static IP, in a private range that isn't used elsewhere on the network: `10.10.10.0/24`.

**Metasploitable (legacy Debian networking, no NetworkManager):**
```bash
sudo nano /etc/network/interfaces
```
```
auto eth0
iface eth0 inet static
address 10.10.10.10
netmask 255.255.255.0
```
```bash
sudo /etc/init.d/networking restart
```

**Kali, second NIC (NetworkManager):**
```bash
sudo nmcli con add type ethernet ifname eth1 con-name lab \
  ipv4.method manual ipv4.addresses 10.10.10.20/24 ipv4.never-default yes
sudo nmcli con up lab
```

`ipv4.never-default yes` is important here: without it, this connection could attempt to become Kali's default route for all traffic, breaking its internet access through the `vmbr0` NIC.

## Step 4 — Verify

```bash
# From Kali
ping -c 3 10.10.10.10        # should succeed — same isolated network
nmap -sV 10.10.10.10         # should return Metasploitable's open services

# Confirm the isolated side truly has no internet path
ping -c 2 8.8.8.8             # run from Metasploitable — should fail
```

---

## Issue — SSH'd into the wrong host entirely

**Symptom:** repeated `Permission denied` when trying to SSH into "Kali" to enable copy-paste from the console.

**Cause:** the target IP being used (`192.168.100.2`) was actually the **Proxmox host itself** — the same address `scp` had just uploaded the Metasploitable disk to earlier in the session — not the Kali VM. Proxmox has no user account matching the one being tried, hence the repeated failures.

**Diagnosis:**
```bash
# Run inside the Kali console, not the host
ip -br a
```
This revealed Kali's actual interfaces and MAC addresses, which were then cross-checked against the VM's Hardware tab in Proxmox to confirm which NIC (`eth0`) was on which bridge (`vmbr0`).

**Follow-up issue — no IPv4 address on Kali's NICs at all**, discovered from the same `ip -br a` output: `eth1` only showed a link-local `fe80::` address, and `eth0` showed none. SSH could never have worked until an address existed.

**Fix:**
```bash
# Try DHCP first
sudo nmcli device connect eth0

# If no DHCP is available, assign statically (gateway confirmed via `ip route` on the Proxmox host)
sudo nmcli con add type ethernet ifname eth0 con-name home \
  ipv4.method manual ipv4.addresses 192.168.100.20/24 \
  ipv4.gateway 192.168.100.1 ipv4.dns 1.1.1.1
sudo nmcli con up home
```

**Lesson:** when SSH fails repeatedly with correct-looking credentials, verify *which host* is actually being targeted before assuming a credentials problem — `ip -br a` and matching MAC addresses against the Proxmox Hardware tab is the fastest way to confirm.

---

## Final architecture

| Device | Interface | IP | Bridge | Internet |
|---|---|---|---|---|
| Proxmox host | — | 192.168.100.2 | vmbr0 | Yes |
| Kali | eth0 | 192.168.100.20 | vmbr0 | Yes |
| Kali | eth1 | 10.10.10.20 | vmbr1 | No |
| Metasploitable | eth0 | 10.10.10.10 | vmbr1 | No |

## Planned improvement

Kali currently acts as an informal bridge between the two networks by virtue of having a leg on each. The next iteration replaces this with an **OPNsense** firewall VM sitting between zones, with explicit, logged rules controlling what can reach what — rather than relying on Kali simply not routing between its interfaces.
