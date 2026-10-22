
# Windows 11 Workstation Provisioning — VMware Homelab

**Project:** Hybrid Active Directory / Entra ID / Intune Homelab  
**Device:** COMP02  
**Operating System:** Windows 11 Pro  
**Hypervisor:** VMware Workstation Pro  
**Domain:** aden.local  
**Environment:** Windows Server 2022, Active Directory, Entra ID, Intune  
**Date:** October 22, 2026  
**Status:** In Progress

---

## 1. Overview

This document describes the process of provisioning a Windows 11 workstation inside VMware Workstation Pro, configuring its network connection, joining it to an Active Directory domain, and validating functionality.

COMP02 serves as a disposable troubleshooting endpoint for practicing enterprise IT support scenarios.

### Objectives

- Deploy Windows 11 using installation media.
- Configure virtual hardware and networking.
- Install VMware Tools and Windows updates.
- Verify TCP/IP, DHCP, DNS, and internet connectivity.
- Join the workstation to an Active Directory domain.
- Verify domain authentication and Group Policy.
- Create snapshots for troubleshooting and recovery.
- Prepare the endpoint for future simulated IT support tickets.

### Technologies

- VMware Workstation Pro
- Windows 11 Pro
- Windows Server 2022
- Active Directory Domain Services (AD DS)
- DNS and DHCP
- TCP/IP
- Windows PowerShell
- Group Policy
- Microsoft Entra ID
- Microsoft Intune

---

## 2. Existing Homelab Infrastructure

The environment consists of a physical Lenovo ThinkCentre running Windows and VMware Workstation Pro.

### Physical Host

| Component | Configuration |
|---|---|
| Computer | Lenovo ThinkCentre M920q |
| Hypervisor | VMware Workstation Pro |
| RAM | 16 GB DDR4 |
| Storage | NVMe SSD, approximately 237 GB |
| Available storage before installation | Approximately 100 GB |
| Host operating system | Windows |
| Network | Home LAN |

### Virtual Machines

| Device | Operating System | Purpose |
|---|---|---|
| DC01 (logical name) | Windows Server 2022 | Domain Controller, DNS, AD DS |
| COMP01 | Windows 11 | Existing domain-joined endpoint |
| COMP02 | Windows 11 Pro | New troubleshooting endpoint |
| Ubuntu Server | Ubuntu Linux | Docker and network services |

**Note:** The existing domain controller's actual hostname is `WIN-8QGPUAN7QO8`. DC01 is used as a logical label in documentation.

### Domain Information

```text
Active Directory Domain: aden.local
Domain Controller: WIN-8QGPUAN7QO8
New Workstation: COMP02
```

The domain controller manages Active Directory authentication and internal DNS resolution.

COMP02 will initially operate as a standalone workstation before joining the domain.

---

## 3. Resource Planning

Before creating a new virtual machine, inspect available system resources.

### Check Host RAM

Open:

```text
Task Manager > Performance > Memory
```

Record:

- Installed physical memory
- Current memory utilization
- Available memory
- Number of running virtual machines

### Check Host Storage

Open:

```text
Settings > System > Storage
```

Ensure sufficient free storage for:

- Windows installation ISO
- Virtual machine disk
- Windows updates
- VMware snapshots
- Application installations

### Recommended COMP02 Configuration

| Component | Allocation |
|---|---|
| Operating System | Windows 11 Pro x64 |
| Memory | 4096 MB |
| CPU | 2 vCPUs |
| Virtual Disk | 64 GB |
| Disk Provisioning | Growable |
| Firmware | UEFI |
| Secure Boot | Enabled |
| TPM | Virtual TPM 2.0 |
| Network Adapter | Bridged |

A growable virtual disk does not immediately consume its entire configured capacity.

However, Windows updates, applications, and snapshots will gradually increase host storage consumption.

### Resource Management

Because the physical host has 16 GB RAM, avoid running all virtual machines simultaneously.

Recommended operating configuration:

```text
Windows Host
    |
    |-- Windows Server 2022 DC (4 GB)
    |
    |-- COMP02 (4 GB)
    |
    |-- COMP01 (Powered Off)
    |
    |-- Ubuntu Server (Powered Off)
```

Monitor actual memory usage and paging when running multiple virtual machines.

---

## 4. Download Windows 11 Installation Media

### Step 1: Visit Microsoft's Website

Official download page:

https://www.microsoft.com/software-download/windows11

### Step 2: Obtain the ISO

If using the Windows Media Creation Tool:

1. Launch the tool.
2. Accept the license terms.
3. Select the language and edition.
4. Choose **ISO file**.
5. Select a destination folder.
6. Wait for the ISO to finish downloading.

Alternatively, download the Windows 11 ISO directly from Microsoft's website.

### Step 3: Store Installation Media

Example location:

```text
C:\VM-ISOs\Windows11.iso
```

Verify that the ISO exists before proceeding.

**Important:** Do not execute `setup.exe` from the ISO on the physical host. The installation should occur inside the new virtual machine.

---

## 5. Create the Virtual Machine

### Step 1: Open VMware Workstation

Launch VMware Workstation Pro.

Navigate to:

```text
File > New Virtual Machine
```

Select:

```text
Typical (Recommended)
```

Click Next.

### Step 2: Select Installation Media

Choose:

```text
Installer disc image file (ISO)
```

Browse to the Windows 11 ISO.

If VMware offers Easy Install, either follow its prompts or choose manual installation for greater control.

### Step 3: Configure Guest Operating System

Select:

```text
Guest OS: Microsoft Windows
Version: Windows 11 x64
```

### Step 4: Name the VM

```text
Virtual Machine Name: COMP02
```

Choose a location on the host with sufficient available storage.

Example:

```text
C:\Virtual Machines\COMP02
```

### Step 5: Configure Virtual Disk

Recommended settings:

```text
Maximum Disk Size: 64 GB
Disk Type: Growable
```

Do not select an option that allocates the entire disk capacity immediately if host storage is limited.

### Step 6: Customize Hardware

Before finishing, open:

```text
Customize Hardware
```

Configure:

```text
Memory: 4096 MB
Processors: 2 vCPUs
Network Adapter: Bridged
Firmware: UEFI
Secure Boot: Enabled
TPM: Virtual TPM 2.0
```

Some VMware Workstation versions require virtual machine encryption before adding a virtual TPM.

Follow VMware's supported encryption workflow and securely store the encryption password.

Finish creating the VM.

---

## 6. Install Windows 11

Power on COMP02.

The VM should boot from the Windows 11 installation ISO.

### Installation Procedure

1. Select language and keyboard layout.
2. Start Windows installation.
3. Choose Windows 11 Pro if edition selection is available.
4. Accept applicable license terms.
5. Select the custom installation option when appropriate.
6. Select the empty virtual disk.
7. Allow Windows to install and restart.
8. Complete the initial Windows setup.

Windows 11 Pro is preferred because it supports traditional Active Directory domain joining.

### Windows Activation

A Windows installation may be usable for evaluation without immediate activation, but this does not eliminate licensing requirements.

Use an appropriately licensed Windows installation for production environments.

---

## 7. Install VMware Tools

VMware Tools provides improved guest integration and device support.

In VMware Workstation, navigate to:

```text
VM > Install VMware Tools
```

Inside COMP02:

1. Open File Explorer.
2. Navigate to the mounted VMware Tools installer.
3. Run the installer.
4. Complete the installation.
5. Restart Windows if prompted.

### Verify Installation

Check:

```text
Settings > Apps > Installed Apps
```

Confirm that VMware Tools appears.

Also verify:

- Mouse integration
- Display resizing
- Network adapter functionality
- General VM responsiveness

---

## 8. Configure Windows Networking

### Initial Network Configuration

For initial provisioning, use:

```text
VMware Network Adapter: Bridged
```

Bridged networking allows COMP02 to connect to the physical LAN as another network device.

The workstation should receive an IP address from the network's DHCP service.

### Check IP Configuration

Open PowerShell:

```powershell
ipconfig /all
```

Record:

```text
IPv4 Address:
Subnet Mask:
Default Gateway:
DHCP Server:
DNS Server:
MAC Address:
```

### Verify Local Connectivity

Identify the default gateway from `ipconfig /all`.

Then run:

```powershell
ping <default-gateway-ip>
```

Replace the placeholder with the actual gateway address.

### Verify External IP Connectivity

```powershell
ping 1.1.1.1
```

Successful replies suggest external IP connectivity is functioning.

ICMP may be blocked on some networks, so a failed ping alone does not prove that the network is down.

### Verify DNS Resolution

```powershell
nslookup microsoft.com
```

### Verify HTTPS Connectivity

```powershell
Test-NetConnection microsoft.com -Port 443
```

A successful TCP connection indicates that the workstation can reach the remote HTTPS service.

---

## 9. Configure Active Directory DNS

Active Directory relies on DNS to locate domain controllers and other domain services.

Before joining COMP02 to the domain, configure its DNS resolver to use the domain controller's LAN IP address.

### Identify the Domain Controller IP

On the Windows Server 2022 domain controller:

```powershell
ipconfig /all
```

Record the correct LAN IPv4 address.

### Configure DNS on COMP02

Open:

```text
Control Panel
    > Network and Internet
    > Network Connections
```

Right-click the active Ethernet adapter.

Select:

```text
Properties
    > Internet Protocol Version 4 (TCP/IPv4)
    > Properties
```

Keep IP addressing automatic if DHCP is being used.

Configure the preferred DNS server to the domain controller's actual LAN IP address.

Do not configure public DNS as an alternate resolver for domain-joined clients unless the DNS architecture explicitly supports it.

### Verify Domain DNS

```powershell
nslookup aden.local
```

Check domain controller service discovery:

```powershell
nslookup -type=SRV _ldap._tcp.dc._msdcs.aden.local
```

Expected result:

The query should return an appropriate domain controller service record.

### Test Domain Controller Discovery

```powershell
nltest /dsgetdc:aden.local
```

Successful discovery indicates that Windows can locate a domain controller.

---

## 10. Rename the Workstation

Before domain joining, assign the intended computer name.

Open PowerShell as Administrator:

```powershell
Rename-Computer -NewName "COMP02" -Restart
```

After restarting:

```powershell
hostname
```

Expected output:

```text
COMP02
```

---

## 11. Create a Standalone Baseline Snapshot

Before joining Active Directory, create a snapshot of the working standalone workstation.

In VMware Workstation:

```text
VM > Snapshot > Take a Snapshot
```

Snapshot name:

```text
COMP02 - Clean Standalone Baseline
```

Description:

```text
Fresh Windows 11 installation.
VMware Tools installed.
Bridged networking verified.
Internet and DNS connectivity tested.
Not joined to Active Directory.
```

This snapshot provides a recovery point for standalone Windows and networking exercises.

**Important:** VMware snapshots are not substitutes for backups.

---

## 12. Join Active Directory

### Prerequisites

Confirm:

- The domain controller is running.
- COMP02 can reach the domain controller.
- DNS points to the domain controller.
- Domain controller discovery succeeds.
- System clocks are synchronized.
- Authorized domain credentials are available.

### Method A: Graphical Interface

Open:

```text
Win + R
```

Enter:

```text
sysdm.cpl
```

Navigate to:

```text
Computer Name > Change
```

Select:

```text
Member of: Domain
```

Enter:

```text
aden.local
```

Provide authorized domain credentials.

Restart when prompted.

### Method B: PowerShell

Run PowerShell as Administrator:

```powershell
Add-Computer `
    -DomainName "aden.local" `
    -Credential (Get-Credential) `
    -Restart
```

Enter authorized domain credentials when prompted.

Avoid placing passwords directly into scripts.

### Verify Domain Membership

After restarting:

```powershell
(Get-CimInstance Win32_ComputerSystem).PartOfDomain
```

Expected output:

```text
True
```

Check the domain name:

```powershell
(Get-CimInstance Win32_ComputerSystem).Domain
```

Expected output:

```text
aden.local
```

---

## 13. Verify Active Directory Authentication

Sign in using an authorized domain user account.

Check identity:

```powershell
whoami
```

Check domain information:

```powershell
whoami /all
```

Verify domain controller discovery:

```powershell
nltest /dsgetdc:aden.local
```

### Verify the Computer Object

On the domain controller, open:

```text
Active Directory Users and Computers
```

Locate the COMP02 computer object.

If necessary, move it to the intended organizational unit.

Example:

```text
aden.local
    |
    +-- USA
         |
         +-- Computers
              |
              +-- COMP02
```

### Verify Group Policy

On COMP02:

```powershell
gpupdate /force
```

Then:

```powershell
gpresult /r
```

For a detailed report:

```powershell
gpresult /h C:\Windows\Temp\gp-report.html
```

Open the HTML report and inspect applied policies.

---

## 14. Create Domain-Joined Baseline Snapshot

After verifying domain authentication and Group Policy:

```text
VM > Snapshot > Take a Snapshot
```

Name:

```text
COMP02 - Domain Joined Baseline
```

Description:

```text
Windows 11 Pro installed.
VMware Tools installed.
Network connectivity verified.
DNS configured for AD.
Joined to aden.local.
Domain authentication verified.
Group Policy tested.
```

### Snapshot Warning

Reverting a domain-joined VM to an older snapshot can create inconsistencies between the workstation and Active Directory.

Potential issues include:

- Machine account password mismatch
- Broken domain trust
- Outdated device configuration
- Stale security state

After restoring a snapshot, verify domain connectivity and authentication.

Refresh baseline snapshots periodically.

---

## 15. Prepare COMP02 for Troubleshooting

COMP02 is intended to simulate real IT support incidents.

### Networking Scenarios

- DHCP lease failure
- APIPA address assignment
- Incorrect subnet mask
- Incorrect default gateway
- DNS server misconfiguration
- Internal DNS resolution failure
- VPN routing failure
- Network adapter configuration problems

### Windows Endpoint Scenarios

- Windows service failures
- Application installation failures
- Windows Update problems
- High CPU or memory utilization
- Driver problems
- Printer spooler failures

### Active Directory Scenarios

- Failed domain authentication
- Incorrect group membership
- Group Policy failures
- Domain trust issues
- SMB share access failures

### Microsoft 365 and Intune Scenarios

- Device registration problems
- Compliance policy failures
- Application deployment errors
- M365 authentication problems
- Endpoint configuration conflicts

---

## 16. Useful Troubleshooting Commands

### IP Configuration

```powershell
ipconfig /all
```

### Renew DHCP Lease

```powershell
ipconfig /release
ipconfig /renew
```

### Clear DNS Resolver Cache

```powershell
ipconfig /flushdns
```

### Test DNS

```powershell
nslookup aden.local
```

### Test TCP Connectivity

```powershell
Test-NetConnection <server-ip> -Port 445
```

Port 445 is commonly used for SMB.

### View Routing Table

```powershell
route print
```

### Trace Network Path

```powershell
tracert 1.1.1.1
```

### Inspect Network Adapters

```powershell
Get-NetAdapter
```

### Inspect IP Configuration

```powershell
Get-NetIPConfiguration
```

### Inspect DNS Configuration

```powershell
Get-DnsClientServerAddress
```

### Inspect Windows Services

```powershell
Get-Service
```

### View Recent System Events

```powershell
Get-WinEvent -LogName System -MaxEvents 20
```

### Verify Group Policy

```powershell
gpresult /r
```

### Verify Domain Controller Discovery

```powershell
nltest /dsgetdc:aden.local
```

---

## 17. Common Troubleshooting Problems

### Problem: Windows Setup Cannot Proceed

Possible causes:

- TPM 2.0 unavailable
- Secure Boot configuration
- Insufficient RAM
- Incorrect virtual firmware
- Unsupported virtual hardware configuration

Troubleshooting:

1. Shut down the VM.
2. Inspect VMware hardware settings.
3. Verify UEFI firmware.
4. Verify virtual TPM configuration.
5. Confirm memory and storage requirements.
6. Restart installation.

### Problem: COMP02 Receives an APIPA Address

Example:

```text
169.254.x.x
```

Possible causes:

- DHCP server unavailable
- Incorrect virtual network configuration
- Network adapter disconnected
- DHCP traffic blocked

Troubleshooting:

```powershell
ipconfig /all
Get-NetAdapter
ipconfig /renew
```

Verify VMware network settings and DHCP availability.

Do not change DHCP services on the production or home network without authorization.

### Problem: Internet Works but Domain Join Fails

Possible causes:

- Incorrect DNS server
- Domain controller unavailable
- AD DNS records missing
- Firewall restrictions
- Time synchronization problems

Troubleshooting:

```powershell
nslookup aden.local
nltest /dsgetdc:aden.local
Test-NetConnection <dc-ip> -Port 53
Test-NetConnection <dc-ip> -Port 389
```

A successful TCP test does not verify every protocol required for domain joining.

Review DNS, authentication, and domain controller event logs as needed.

### Problem: Domain User Cannot Sign In

Possible causes:

- Incorrect credentials
- Disabled account
- Account lockout
- DNS or DC connectivity failure
- Logon restrictions
- Broken domain trust

Troubleshooting:

```powershell
whoami
nltest /dsgetdc:aden.local
```

Review relevant authentication events and account settings.

### Problem: Group Policy Does Not Apply

Possible causes:

- Computer located in incorrect OU
- GPO link or security filtering
- DNS connectivity failure
- WMI filtering
- Policy processing errors

Troubleshooting:

```powershell
gpupdate /force
gpresult /r
```

Review Group Policy operational events.

---

## 18. Security and Operational Best Practices

- Use least-privilege accounts.
- Avoid storing credentials in scripts.
- Apply Windows security updates.
- Enable Windows Defender and Firewall.
- Document changes before making them.
- Take snapshots before destructive testing.
- Avoid changing domain controller networking during experiments.
- Isolate risky network tests from the home LAN.
- Never introduce unauthorized DHCP servers onto a shared network.
- Verify functionality after each change.
- Document the rollback procedure.

### Network Isolation

Bridged networking connects the VM to the physical LAN.

For destructive network experiments, use a dedicated VMware host-only or isolated virtual network.

Do not assume a bridged VM is isolated from other devices.

---

## 19. Enterprise IT Support Relevance

This homelab workflow simulates portions of workstation provisioning performed by IT support technicians and endpoint administrators.

### Related Enterprise Responsibilities

- Operating system deployment
- Workstation configuration
- Device naming conventions
- Network troubleshooting
- Active Directory domain joining
- DNS configuration
- Group Policy verification
- Endpoint validation
- Incident documentation

### Differences From Production Environments

Production organizations may use:

- Windows Autopilot
- Microsoft Intune
- Microsoft Configuration Manager
- Windows Deployment Services
- Automated provisioning scripts
- Standardized security baselines
- Asset inventory systems
- Endpoint detection and response tools

VMware snapshots are a lab recovery mechanism and are not equivalent to enterprise endpoint backup or deployment management.

Production workflows also require approved change procedures, licensing, security controls, and organizational policies.

---

## 20. Deployment Verification Checklist

### Installation

- [ ] Windows 11 ISO downloaded
- [ ] COMP02 VM created
- [ ] Virtual hardware configured
- [ ] Windows 11 installed
- [ ] VMware Tools installed
- [ ] Windows updates completed

### Networking

- [ ] Network adapter connected
- [ ] Valid IPv4 address assigned
- [ ] Default gateway identified
- [ ] Internet connectivity verified
- [ ] DNS resolution verified
- [ ] Domain controller reachable

### Active Directory

- [ ] Computer renamed COMP02
- [ ] Standalone baseline snapshot created
- [ ] Domain DNS configured
- [ ] Domain controller discovery verified
- [ ] COMP02 joined to aden.local
- [ ] Domain user authentication verified
- [ ] Computer object verified in AD
- [ ] Group Policy verified
- [ ] Domain-joined baseline snapshot created

### Documentation

- [ ] Actual IP configuration recorded
- [ ] Installation problems documented
- [ ] Screenshots captured
- [ ] Validation results recorded
- [ ] Final VM configuration documented
- [ ] Rollback procedure tested

---

## 21. Deployment Record

Complete this section after installation.

### Actual Configuration

```text
Hostname:
Operating System:
Windows Build:
VMware Workstation Version:
Allocated RAM:
Allocated vCPUs:
Virtual Disk:
Network Mode:
IPv4 Address:
Subnet Mask:
Default Gateway:
DNS Server:
Domain:
Organizational Unit:
Installation Date:
```

### Problems Encountered

```text
Issue:

Symptoms:

Troubleshooting Steps:

Root Cause:

Resolution:

Verification:
```

### Final Verification

```text
[ ] Windows boots successfully
[ ] Network connection operational
[ ] DNS resolves correctly
[ ] Domain authentication works
[ ] Group Policy applies
[ ] Snapshot created
[ ] COMP02 ready for ticket simulations
```

---

## 22. Key Takeaways

Provisioning a Windows workstation involves more than installing an operating system.

A properly configured enterprise endpoint requires:

1. Appropriate hardware allocation.
2. Correct operating system installation.
3. Functional network connectivity.
4. Reliable DNS resolution.
5. Successful domain authentication.
6. Appropriate Group Policy configuration.
7. Security and update management.
8. Validation and documentation.

This procedure establishes a repeatable foundation for deploying and troubleshooting Windows endpoints.

The next phase will use COMP02 to simulate realistic IT support incidents involving networking, Windows services, authentication, SMB, VPN, and printer troubleshooting.

---

## References

- [Microsoft Windows 11 Download](https://www.microsoft.com/software-download/windows11)
- [Microsoft Learn — Windows Documentation](https://learn.microsoft.com/windows/)
- [Microsoft Learn — Active Directory Domain Services](https://learn.microsoft.com/windows-server/identity/ad-ds/)
- [Microsoft Learn — PowerShell Documentation](https://learn.microsoft.com/powershell/)
- [Microsoft Learn — Microsoft Intune](https://learn.microsoft.com/intune/)
