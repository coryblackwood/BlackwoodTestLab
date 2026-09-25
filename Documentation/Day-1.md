# Document Day 1 infrastructure deployment — Proxmox, Windows Server, Active Directory & DNS

**Project:** Blackwood IT Infrastructure Lab
**Day:** 1
**Focus:** Virtualization, Windows Server deployment, Active Directory, DNS, and network troubleshooting
---
## Objective
Build the initial infrastructure for the Blackwood homelab and establish a functional Windows Server domain environment.

Day 1 objectives were:
* Deploy Proxmox on the primary server
* Create a Windows Server virtual machine
* Configure static networking
* Deploy Active Directory Domain Services
* Configure DNS
* Create the initial Active Directory structure
* Create a test domain user
* Connect a physical Windows client to the lab network
* Document troubleshooting and lessons learned
---
## Lab Hardware

### MS01 — Proxmox Host
* **System:** MSI Aegis Ti3 VR7RD
* **CPU:** Intel Core i7-6700K
* **RAM:** 48 GB
* **GPU:** NVIDIA GT 1030
* **Storage:** 512 GB Intel Optane H-series drive
* **Hypervisor:** Proxmox VE
* **Management IP:** `192.168.50.10`

### WS01 — Windows Server
* **Platform:** Proxmox VM on MS01
* **OS:** Windows Server 2022
* **RAM:** 8 GB
* **vCPU:** 4
* **Virtual disk:** 80 GB
* **IP:** `192.168.50.20`
* **Role:** Active Directory Domain Controller + DNS

### OPS01
* Linux Mint laptop used as the primary administration workstation.

### AC01
* Acer Aspire-X
* Windows 10 Home
* Intended as a physical Windows client
* Currently configured for network/DNS testing
* Not domain-capable because Windows 10 Home does not support traditional Active Directory domain joining
---
# Network Configuration
The initial lab is operating on the existing home LAN.
| Device | Role            | Address         |
| ------ | --------------- | --------------- |
| Router | Default gateway | `192.168.50.1`  |
| MS01   | Proxmox host    | `192.168.50.10` |
| WS01   | AD/DNS server   | `192.168.50.20` |
| AC01   | Windows client  | DHCP            |

Subnet:
`192.168.50.0/24`
Default gateway:
`192.168.50.1`
WS01 DNS:
`192.168.50.20`
---

# Proxmox Deployment
Proxmox VE was successfully installed on MS01.
The Proxmox management interface was configured on the physical Ethernet connection and assigned:
`192.168.50.10`
The Proxmox web interface was successfully accessed remotely from OPS01.
## Storage Troubleshooting

### Problem
During the Proxmox installation process, the internal 512 GB drive was not initially visible.
The installer only showed the installation USB and another USB device.

### Investigation
The system BIOS showed the internal storage as an Intel Optane H-series device:
`HBRPEKNX0202AH`
The BIOS storage configuration was using Intel RST/RAID mode.
Proxmox did not detect the drive under this configuration.

### Resolution
BIOS storage mode was changed from:
**Intel RST/RAID**
to:
**AHCI**
Secure Boot was also disabled.
After changing the storage configuration, Proxmox detected the internal drive and installation proceeded successfully.

### Result
The Optane drive became available to Proxmox and was used for the host installation and VM storage.

### Lesson Learned
Hypervisor storage visibility can depend on the firmware storage controller mode. When a physical drive appears in firmware but not in Linux, controller mode and storage-driver compatibility should be investigated before assuming the drive is defective.
---

# WS01 Deployment
A Windows Server 2022 VM was created on MS01.
Initial VM configuration included:
* UEFI/OVMF firmware
* 4 vCPUs
* 8 GB RAM
* 80 GB virtual disk
* Virtual networking
* Windows Server 2022 installation ISO
## Windows Installer Storage Issue

### Problem
Windows Server 2022 booted successfully, but the installer did not display the 80 GB virtual disk.

### Investigation
The virtual disk was attached through the VirtIO SCSI storage controller.
The Windows installation environment did not have the required VirtIO storage driver loaded.

### Resolution
The original blank virtual disk was detached and a new 80 GB virtual disk was created using the SATA bus.

### Result
Windows Server 2022 successfully detected the virtual disk and installation proceeded.

### Lesson Learned
Paravirtualized storage can require additional drivers in Windows installation environments. Using a more universally supported virtual storage controller is a practical workaround when building an initial lab system.
---

# Windows Server Networking Issue

### Problem
After Windows Server installation, Device Manager showed an Ethernet controller with no compatible driver.

### Investigation
The VM's network adapter was configured as:
**VirtIO (paravirtualized)**
Windows Server did not have the required VirtIO network driver installed.

### Resolution
The virtual network adapter was changed from VirtIO to:
**Intel E1000**
Windows immediately recognized the virtual network adapter.

### Result
WS01 obtained network connectivity and was subsequently configured with a static address.

### Lesson Learned
Virtual hardware still depends on operating-system driver support. When a VM has no network connectivity, checking Device Manager and identifying the virtual hardware model is an effective troubleshooting step.
---
# WS01 Static Network Configuration
WS01 was assigned:
IP address:      192.168.50.20
Subnet mask:     255.255.255.0
Default gateway: 192.168.50.1
DNS:             192.168.50.1
```
After Active Directory and DNS were installed, WS01's DNS configuration was changed to:
DNS: 192.168.50.20
```
This allows the domain controller to use its own AD-integrated DNS service.
Network connectivity was verified using:
```powershell
ping 192.168.50.1
ping 192.168.50.10
ping google.com
```
All tests returned successfully.
---
# Active Directory Deployment
Active Directory Domain Services was installed on WS01.
WS01 was promoted to the first domain controller in a new forest.

### Domain
`blackwood.local`
### NetBIOS Name
`BLACKWOOD`
### Domain Controller
`WS01`

### Services
* Active Directory Domain Services
* DNS
* Global Catalog
The domain controller was rebooted after promotion.
---
# Active Directory Verification
After reboot, the domain login screen displayed:
`BLACKWOOD\Administrator`
The original Windows Server Administrator password successfully authenticated.
The following command was used to verify the logged-in domain account:
```powershell
whoami
```
Result:
```text
blackwood\administrator
```
The domain configuration was also verified with:
```powershell
Get-ADDomain
```
The command returned the expected Blackwood domain information.
---
# DNS Verification
DNS was tested from WS01 using:
```powershell
nslookup blackwood.local
```
and:
```powershell
nslookup google.com
```
Both resolved successfully.
This verified both internal domain DNS functionality and external name resolution.
---
# Active Directory Organizational Structure
The following custom OUs were created:
```text
blackwood.local
│
├── Groups
├── Servers
└── Workstations
```
The default Windows containers were left intact:
```text
Builtin
Computers
Domain Controllers
ForeignSecurityPrincipals
Managed Service Accounts
Users
```
No default AD containers were deleted or modified unnecessarily.
---
# Domain User
A standard test account was created:
**Username:**
`BLACKWOOD\labuser`
This account will eventually be used to test domain authentication from a Windows client.
The purpose is to demonstrate the difference between:
* Domain administration
* Normal user authentication
* Group membership
* Workstation/domain interaction
---
# AC01 Client Testing
AC01 is a physical Acer Windows 10 system.
The machine was configured with:
* DHCP IP addressing
* Gateway: `192.168.50.1`
* DNS: `192.168.50.20`
Connectivity to WS01 was verified:
```cmd
ping 192.168.50.20
```
DNS resolution was verified:
```cmd
nslookup blackwood.local
```
Both tests succeeded.
## Domain Join Attempt
AC01 was then tested for Active Directory domain membership.
The join attempt failed.

### Investigation
The system was running:
**Windows 10 Home**
Windows 10 Home does not support traditional Active Directory domain joining.

### Resolution
No changes were made to the system.
AC01 will remain useful as a physical network client for testing, while a Windows Pro-capable client will be used for future domain authentication and Group Policy testing.

### Lesson Learned
Operating-system edition is an important prerequisite when planning Windows domain environments. Network connectivity and DNS can be completely correct while domain joining remains unavailable because of an edition limitation.
---
# Current Architecture
                         Internet
                            │
                     192.168.50.1
                       Home Router
                            │
                         Switch
                            │
             ┌──────────────┼──────────────┐
             │              │              │
       MS01 .10         OPS01           AC01
       Proxmox         Linux Mint     Win 10 Home
             │
             │
            WS01
          .20
             │
      ┌──────┴──────┐
      │             │
 Active Directory  DNS
      │
 blackwood.local
```
---
# Day 1 Troubleshooting Summary
| Problem                                     | Root Cause                   | Resolution                                                 |
| ------------------------------------------- | ---------------------------- | ---------------------------------------------------------- |
| Proxmox could not see internal storage      | Intel RST/RAID configuration | Changed storage mode to AHCI                               |
| Server 2022 installer could not see VM disk | Missing VirtIO SCSI driver   | Used SATA virtual disk                                     |
| Windows Server had no network adapter       | Missing VirtIO NIC driver    | Changed virtual NIC to Intel E1000                         |
| AC01 could not join domain                  | Windows 10 Home limitation   | Retained AC01 for network testing; plan Pro-capable client |
---
# Day 1 Results
The initial Blackwood infrastructure is operational.
The lab now has:
* A functioning Proxmox hypervisor
* A Windows Server 2022 domain controller
* Active Directory
* Internal DNS
* A working domain
* Organizational Units
* A standard domain user
* A configured physical Windows client
* Verified network connectivity
* Verified DNS resolution
* Documented troubleshooting procedures
The infrastructure foundation is ready for the next phase.
---
# Day 2 Planned Work
1. Deploy a Windows Pro-capable domain client.
2. Join the client to `blackwood.local`.
3. Log in using `BLACKWOOD\labuser`.
4. Verify domain authentication.
5. Build **SRV01** as the Linux server.
6. Configure SSH and basic Linux administration.
7. Begin Group Policy configuration.
8. Expand AD users and security groups.
9. Begin documenting the client/server management workflow.
---
# Skills Demonstrated
Day 1 provided hands-on experience with:
* Proxmox VE
* UEFI/virtual machine configuration
* Virtual storage
* Virtual networking
* Windows Server 2022
* Active Directory Domain Services
* Active Directory Users and Computers
* DNS
* Static IPv4 configuration
* Windows domain architecture
* Windows troubleshooting
* Virtual hardware drivers
* Network troubleshooting
* DNS troubleshooting
* Basic PowerShell
* Infrastructure documentation
---
