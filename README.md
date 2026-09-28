# Proxmox Security Homelab

![Platform](https://img.shields.io/badge/platform-Proxmox%20VE-orange)
![Focus](https://img.shields.io/badge/focus-network%20segmentation%20%7C%20offensive%20security-blue)
![Status](https://img.shields.io/badge/status-active-brightgreen)

A self-hosted virtualization lab built on Proxmox VE, designed as a safe, segmented environment for practicing offensive security techniques — scanning, exploitation, and remote administration — without exposing anything to the internet or to the rest of the home network.

This repo documents the build process, the architecture decisions, and the real problems encountered along the way, with root causes and fixes.

## Table of contents

- [Why this lab exists](#why-this-lab-exists)
- [Architecture](#architecture)
- [Remote access](#remote-access)
- [What's documented here](#whats-documented-here)
- [Skills demonstrated](#skills-demonstrated)
- [Planned next steps](#planned-next-steps)
- [Disclaimer](#disclaimer)

## Why this lab exists

I'm a second-year Informatique student aiming for a cybersecurity career (SOC analyst / pentesting), and I wanted hands-on infrastructure to practice on rather than just reading theory. This lab is the foundation I'm building detection and exploitation exercises on top of.

## Architecture

The lab uses two virtual networks inside one Proxmox host, kept deliberately separate:

| Device | Role | IP | Bridge | Internet access |
|---|---|---|---|---|
| Proxmox host (`pve`) | Hypervisor | 192.168.100.2 | vmbr0 | Yes |
| Kali Linux (VM 100) — eth0 | Attacker box, home-network side | 192.168.100.20 | vmbr0 | Yes |
| Kali Linux (VM 100) — eth1 | Attacker box, lab side | 10.10.10.20 | vmbr1 | No |
| Metasploitable 2 (VM 200) | Intentionally vulnerable target | 10.10.10.10 | vmbr1 | No |

**vmbr0** is bridged to the host's physical network adapter — it's the "real" network, connected to the home router and the internet.

**vmbr1** is an internal-only bridge with no physical port attached. Nothing plugged into it can reach the internet or the home network — it's isolated by the simple fact that there's no physical path out, not by a firewall rule.

Kali sits with one interface on each network, acting as the attacker machine with internet access for tooling, while Metasploitable exists only on the isolated side.

```text
Internet / Home Network (192.168.100.0/24)
        │
      vmbr0 ──────────── Proxmox host (192.168.100.2)
        │
    Kali eth0 (192.168.100.20)
        │
    [ Kali VM ]
        │
    Kali eth1 (10.10.10.20)
        │
      vmbr1 (isolated, no physical port)
        │
    Metasploitable (10.10.10.10)
```

## Remote access

The Proxmox host is reachable remotely via **Tailscale**, a WireGuard-based mesh VPN, instead of port-forwarding the web GUI to the internet. No ports are opened on the home router; the management interface is only reachable by devices signed into the same Tailscale account.

## What's documented here

- [`docs/01-proxmox-and-vm-setup.md`](docs/01-proxmox-and-vm-setup.md) — installing the Kali VM, importing the Metasploitable 2 disk, and the storage/format issues hit along the way
- [`docs/02-network-segmentation.md`](docs/02-network-segmentation.md) — building the isolated bridge, assigning static IPs, and verifying isolation
- [`docs/03-remote-access-tailscale.md`](docs/03-remote-access-tailscale.md) — setting up Tailscale for secure remote management, including a repo-configuration issue hit mid-install

Each doc follows a **problem → cause → fix** format rather than a plain walkthrough, since the troubleshooting is the actual point.

## Skills demonstrated

- Type-2 hypervisor administration (Proxmox VE)
- Virtual networking: bridges, VLANs-ready design, network segmentation for security isolation
- Linux system administration (Debian/Kali), both NetworkManager and legacy `/etc/network/interfaces` configuration
- Disk image import/conversion and snapshot management
- VPN-based remote access as a secure alternative to port-forwarding
- Root-cause troubleshooting and technical documentation

## Planned next steps

- Deploy OPNsense as a proper firewall between zones, replacing Kali as the informal bridge between networks
- Deploy Wazuh for detection, and generate/detect attacks against Metasploitable and an Active Directory lab
- Automate the lab build with Terraform and Ansible

## Disclaimer

All target systems (Metasploitable 2) are intentionally vulnerable machines designed for security testing, running in an isolated network I own and control. No systems outside this lab are scanned or attacked.
