# 03 — Remote Access via Tailscale

## Goal

Reach the Proxmox web GUI (and SSH) from anywhere, without exposing it to the public internet.

## Why not port-forwarding

Forwarding router port 8006 to the Proxmox host would put its login page directly on the public internet, where it is routinely found and attacked by automated scanners within hours of exposure. A VPN overlay avoids this entirely: nothing is opened on the router, and only devices explicitly authenticated into the same private network can reach the host.

## Why Tailscale

Tailscale is a mesh VPN built on WireGuard. Every device signed into the same account gets a private address in the `100.64.0.0/10` range (a block reserved for exactly this kind of carrier-grade/overlay networking) and can reach every other device on that same account's network ("tailnet") directly, over an encrypted tunnel — without any port-forwarding or public exposure. It's free for personal use.

## Setup

**Install on the Proxmox host:**
```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up
```
`tailscale up` prints a login URL — opened on any browser and signed in with a Tailscale account (created via Google/Microsoft/GitHub SSO or similar).

**Install on other devices** (Windows PC, phone): install the Tailscale app/client and sign into the **same account**. All devices on the same account see each other; devices on different accounts are invisible to one another by default.

**Find the host's Tailscale address:**
```bash
tailscale ip -4
```

**Access from any device on the tailnet:**
```
https://<tailscale-ip>:8006
```
`https://`, not `http://` — Proxmox only serves the GUI over TLS on 8006. The browser will show a certificate warning since Proxmox uses a self-signed certificate by default; this is expected and not a sign of a problem.

---

## Issue — apt errors during Tailscale install

**Error during `apt-get update` (triggered by the Tailscale install script):**
```
Err:4 https://enterprise.proxmox.com/debian/ceph-squid trixie InRelease  401 Unauthorized
Err:6 https://enterprise.proxmox.com/debian/pve trixie InRelease  401 Unauthorized
```

**Cause:** unrelated to Tailscale. Proxmox ships with the **enterprise repositories** enabled by default, which require a paid subscription to access. Any `apt update` on a fresh, no-subscription Proxmox install will show these 401 errors. The Tailscale repository itself fetched successfully in the same run.

**Fix — installed Tailscale directly, since its own repo was unaffected:**
```bash
apt install -y tailscale
```

**Longer-term fix — switch to the free no-subscription repo** so future `apt update` runs don't show these errors:
```bash
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/ceph.list
sed -i 's/^deb/#deb/' /etc/apt/sources.list.d/pve-enterprise.list
echo "deb http://download.proxmox.com/debian/pve trixie pve-no-subscription" \
  > /etc/apt/sources.list.d/pve-no-subscription.list
apt update
```

**Lesson:** errors during a multi-package `apt update` don't necessarily mean the package being installed failed — checking which specific repository each error came from (here, `enterprise.proxmox.com`, not `pkgs.tailscale.com`) avoided chasing the wrong problem.

---

## Verification

```bash
tailscale status              # confirms the host and other devices are online on the tailnet
ss -tlnp | grep 8006          # confirms pveproxy is listening on all interfaces, Tailscale included
```

## Security notes

- No ports are forwarded on the home router.
- 2FA enabled on the Proxmox root account (Datacenter → Permissions → Two Factor) as an additional layer on top of Tailscale's own account authentication.
- The isolated lab network (`10.10.10.0/24`) is intentionally **not** advertised as a Tailscale subnet route — only the home network segment would be, if that feature is used later. The lab network stays reachable only from Kali.

## Result

Proxmox is reachable securely from any authenticated device, from anywhere, with no public-facing attack surface.
