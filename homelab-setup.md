# My Home Lab Setup

## Goals
- Learning new technologies
- Testing applications
- Having Fun while doing it.

## Hardware
- **Server**: Promox VE 8.4.11 (OLD HP Laptop)
  - CPU: Intel CPU
  - RAM: 6GB
- **Laptop**: Lenovo ThinkPad T470
  - CPU: Intel i5
  - RAM: 16 GB

## Software
- **Operating System**: Ubuntu 22.04
- **Containerization**: LXC, Docker (Portainer),

## Network Configuration
- **VPN Router**: TP Link OMADA E605
- **Switch**: TP Link 5 port switch
- **IP Addressing**: 
  -**VLANs - 255.255.255.0          Switch Port**
    HOME_LAN - 192.168.11.1 /24-    PORT 3
    WORK_LAN - 172.16.11.1 /24 -    PORT 2
    WIFI_IOT - 192.168.12.1 /24 -   PORT 5
    VM_LAN   - 192.168.13.1 /24 -   PORT 4

