# Home Lab Setup

A documented overview of my home lab environment, networking layout, and self-hosted services. This repository is used to track infrastructure, experiments, and future improvements.

## Overview

My lab includes:

- A segmented home network with VLANs.
- A Proxmox virtualization host.
- LXC containers and VMs for core services.
- Cloud test instances for cybersecurity and service validation.
- A few additional services I plan to deploy and test.

## Network Hardware

- ISP modem/router with a 1 Gbps connection.
- TP-Link Omada ER605 VPN router.
- TP-Link Omada EAP650 access point.
- TP-Link Omada SDN cloub-based controller.

## Network Segmentation

I separate devices and services into VLANs to keep traffic organized and isolated.

- `HOME_LAN` for general home devices.
- `WORK_LAN` for work-related devices.
- `WIFI_IOT` for IoT and wireless devices.
- `VM_LAN` for virtualization and lab workloads.

## Virtualization Host

My main virtualization platform is Proxmox, which I use to run containers and virtual machines for lab services, testing, and self-hosted applications.

## Containers and Services

### LXC Containers

- pfSense - Firewall service.
- Portainer for Docker management.
- Stirling PDF for document editing.
- Nginx Proxy Manager for reverse proxy handling.
- RustDesk server components for remote access.
- Tailscale container for connectivity.
- Nextcloud for self-hosted file management.
- AdGuard for local DNS filtering.
- Tailscale exit node.

### Virtual Machines

- Kasm Workspaces for spinning up disposable Linux workspaces.
- Kali Purple for defensive cybersecurity testing.

## Cloud Test Instances

I also use AWS cloud instances for remote testing and validation.

- Kali EC2 instance for cybersecurity tooling.
- Ubuntu EC2 instance for service testing and experimentation.

## Services I Plan to Add

- Ansible for configuration management and automation.
- Uptime Kuma for monitoring.
- Jellyfin for media management, if I add NAS storage.
- An AI service or engine for future experimentation.

## My Goals

- Improve segmentation and security.
- Document infrastructure clearly.
- Test self-hosted tools in isolated environments.
- Expand automation and monitoring over time.
- have fun while i learn various technologies

## Notes

This repository intentionally avoids exposing sensitive configuration details, private IP addresses, credentials, and public access endpoints.

---

## Disclaimer

This project is for educational and personal lab use only. Any exposed services are intended for testing and controlled access.
