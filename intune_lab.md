# Intune Lab – Secure Boot Compliance & Configuration Policy

**Date:** October 2, 2026  
**Environment:** Windows 11 client (`COMP01`), VMware, Microsoft Intune, Microsoft Entra ID

## Session Goals

Today's session focused on troubleshooting an Intune compliance failure, remediating Secure Boot on my Windows 11 VM, validating the fix locally, forcing an Intune sync, and creating a basic Intune configuration profile targeted to a user security group.

---

## 1. Troubleshooting Intune Secure Boot Compliance

Intune reported that `COMP01` was noncompliant because the **Secure Boot** requirement was failing.

Rather than immediately changing the Intune compliance policy, I verified the endpoint itself first.

### Check Firmware and Secure Boot Status

On `COMP01`, I opened:

```text
Win + R
msinfo32
```

Initial results:

```text
BIOS Mode: UEFI
Secure Boot State: Off
```

This isolated the problem to the VM's firmware configuration. Windows was already booting through UEFI, but Secure Boot itself was disabled.

### PowerShell Verification

I also used:

```powershell
Confirm-SecureBootUEFI
```

When I initially ran this from a normal PowerShell session, I received an access/privilege error.

The command needs to be run from an **elevated PowerShell/Terminal session**.

After opening PowerShell as Administrator:

```powershell
Confirm-SecureBootUEFI
```

Expected result after remediation:

```text
True
```

---

## 2. Remediating Secure Boot in VMware

Because Secure Boot is a firmware-level feature, the remediation had to be performed in the VM configuration rather than through the logged-in Windows user.

Process:

```text
1. Fully shut down COMP01.
2. Open the COMP01 virtual machine settings in VMware.
3. Confirm the firmware type is UEFI.
4. Enable Secure Boot.
5. Save the VM configuration.
6. Boot COMP01.
```

After booting Windows again, I checked:

```text
Win + R
msinfo32
```

The system now reported:

```text
BIOS Mode: UEFI
Secure Boot State: On
```

This confirmed that Secure Boot was successfully enabled.

### Important Concept

Secure Boot exists at the firmware layer:

```text
VM / Physical Hardware
        ↓
UEFI Firmware
        ↓
Secure Boot
        ↓
Windows Bootloader
        ↓
Windows 11
        ↓
Intune / MDM
```

Therefore, the Windows user currently logged into the machine does not determine whether Secure Boot is enabled.

This helped reinforce the importance of identifying which layer of the system is actually responsible for a problem.

---

## 3. Syncing the Remediated Device With Intune

After fixing Secure Boot locally, I needed Intune to receive the updated device state.

On `COMP01`:

```text
Settings
→ Accounts
→ Access work or school
→ Connected work/school account
→ Info
→ Sync
```

The synchronization completed successfully.

However, Intune did not immediately change the compliance status.

This demonstrated an important endpoint-management concept:

> The current state of a device and the last state reported by Intune are not necessarily identical at every moment.

The device first needs to check in, Intune needs to process the new information, and the compliance policy needs to be reevaluated.

After allowing time for Intune to reevaluate `COMP01`, the Secure Boot compliance issue cleared.

### Troubleshooting Workflow

The complete troubleshooting process was:

```text
Intune reports compliance failure
        ↓
Identify failed compliance requirement
        ↓
Verify endpoint locally
        ↓
Determine responsible system layer
        ↓
Remediate the problem
        ↓
Verify remediation locally
        ↓
Trigger Intune / MDM sync
        ↓
Wait for cloud-side reevaluation
        ↓
Verify compliance in Intune
```

This was useful because the solution was not to modify the Intune compliance policy.

The policy was correctly identifying a real configuration problem on the endpoint.

---

# 4. Creating an Intune Configuration Profile

After resolving the compliance problem, I created a Windows configuration profile.

Navigation:

```text
Intune Admin Center
→ Devices
→ Configuration
→ Create / New Policy
```

Configuration:

```text
Platform: Windows 10 and later
Profile type: Settings catalog
```

For this introductory policy, I experimented with settings including:

- Requiring a PIN for wireless pairing
- Microsoft Defender/security configuration
- Protection/blocking behavior for potentially unwanted or invasive applications

The purpose was not to create a complete production security baseline.

The goal was to understand the Intune configuration lifecycle:

```text
Create Policy
      ↓
Configure Settings
      ↓
Assign Target
      ↓
User / Device Checks In
      ↓
Intune Evaluates Applicability
      ↓
Configuration Reaches Endpoint
      ↓
Verify Deployment Status
```

---

# 5. Understanding Scope Tags

While creating the configuration profile, I encountered **Scope Tags**.

A scope tag does **not** determine which users or devices receive a policy.

Scope tags are primarily used with Intune RBAC to control which Intune administrators can see or manage particular Intune resources.

For example:

```text
LA IT Admins
    ↓
LA Scope Tag
    ↓
LA Devices / Policies

NY IT Admins
    ↓
NY Scope Tag
    ↓
NY Devices / Policies
```

A Global IT administrator could potentially have broader visibility across both environments.

For my single-administrator homelab, the default scope tag is sufficient.

### Scope Tag vs Assignment

```text
Scope Tag
= Which administrators can see/manage the Intune object?

Assignment
= Which users/devices receive the policy?
```

This distinction is important because scope tags are an administrative/RBAC concept, while assignments control policy deployment.

---

# 6. User Groups vs Device Groups

I assigned my configuration profile to:

```text
SG-Sales
```

`SG-Sales` contains `jsmith` along with several other Sales users.

This introduced an important Intune targeting concept.

## User Targeting

Because `SG-Sales` contains users, the configuration is targeted toward those identities.

Conceptually:

```text
SG-Sales
    ↓
jsmith
    ↓
jsmith uses an applicable Intune-managed device
    ↓
User-targeted policy is evaluated/applied
```

`COMP01` does not need to be a member of `SG-Sales` simply because `jsmith` uses the computer.

The policy is being targeted toward the **identity**.

---

## Device Targeting

Some configurations should apply to a computer regardless of which user signs into it.

For those situations, a dedicated device group makes more sense.

For example:

```text
DG-Windows-Devices
        ↓
COMP01
        ↓
Device Configuration Policy
```

Now the configuration follows the endpoint rather than a particular user.

The basic mental model is:

```text
USER GROUP
Policy follows the targeted identity.

DEVICE GROUP
Policy follows the targeted endpoint.
```

This is conceptually similar to traditional Active Directory Group Policy:

```text
User Configuration
vs.
Computer Configuration
```

---

# 7. Example Enterprise Scenario

A useful example would be managing computers at a university.

Suppose I wanted to restrict games across all managed macOS computers in a campus computer lab.

The requirement belongs primarily to the **computers**, not a specific user.

I could create:

```text
DG-Campus-Lab-Macs
```

and place the managed lab Macs into that device group.

Then:

```text
DG-Campus-Lab-Macs
        ↓
Device Restriction Policy
        ↓
All Targeted Managed Macs
```

The restriction would apply to those managed endpoints regardless of which applicable user signs into them.

On the other hand, if a configuration specifically needed to follow students across managed devices, a user group could make more sense:

```text
SG-Students
      ↓
User-targeted policy
      ↓
Applicable managed endpoint
```

Therefore, when creating Intune assignments, I should ask:

> Does this requirement belong primarily to the user or to the endpoint?

---

# 8. Improving My Group Structure

My current Entra groups include:

```text
SG-Sales
SG-HR
SG-IT
```

These represent user security groups.

For endpoint management, I can introduce separate device groups such as:

```text
DG-Windows-Devices
DG-Lab-Devices
```

This makes the purpose of each group clearer.

For example:

```text
IDENTITY GROUPS

SG-Sales
SG-HR
SG-IT

        ↓

Users / Permissions / User Policies
```

versus:

```text
DEVICE GROUPS

DG-Windows-Devices
DG-Lab-Devices

        ↓

Compliance / Configuration / Applications
```

The exact targeting strategy depends on the policy, but separating identity-oriented and device-oriented groups makes the environment easier to understand and administer.

---

# 9. Key Lessons From Today's Session

### Secure Boot

- Secure Boot is a UEFI firmware security feature.
- Secure Boot is a device property, not a user-account property.
- `msinfo32` can quickly show UEFI and Secure Boot status.
- `Confirm-SecureBootUEFI` can verify Secure Boot through PowerShell.
- The PowerShell command should be run from an elevated session.
- VMware firmware settings can directly affect Windows 11 Intune compliance.

### Intune Compliance

- A compliance failure should be investigated before modifying the policy.
- Intune may correctly identify a configuration problem that exists outside Intune itself.
- Local remediation and Intune cloud reporting do not update simultaneously.
- A successful manual sync does not necessarily produce an immediate compliance-status change.
- The endpoint should be verified locally before assuming Intune is wrong.

### Intune Configuration

- Configuration profiles allow administrators to remotely configure managed endpoints.
- Settings Catalog provides granular Windows configuration options.
- Policies need assignments before they can affect users/devices.
- Intune targeting can occur through user groups or device groups.

### Scope Tags

- Scope tags are primarily an administrative/RBAC concept.
- They control which administrators can see/manage Intune resources.
- Scope tags do not determine which users/devices receive a policy.

### User vs Device Targeting

```text
User Group
→ Configuration follows the targeted identity.

Device Group
→ Configuration follows the targeted endpoint.
```

This distinction will become increasingly important as the lab grows.

---

# 10. Troubleshooting Mindset

One of the biggest lessons from this session was learning to identify **which layer owns the problem**.

A Microsoft-managed Windows environment can contain several different layers:

```text
Hardware / VM
        ↓
Firmware
        ↓
Windows
        ↓
Networking
        ↓
Active Directory
        ↓
Entra ID
        ↓
Intune / MDM
        ↓
Microsoft 365
        ↓
Applications
        ↓
User
```

A problem visible in Intune does not necessarily mean Intune caused the problem.

Today's Secure Boot incident was a good example:

```text
Problem visible in:
Intune

Actual problem located in:
VMware virtual firmware

Remediation:
Enable Secure Boot in VMware

Verification:
msinfo32 / PowerShell

Cloud verification:
Intune compliance reevaluation
```

This is a troubleshooting pattern I want to continue practicing rather than immediately changing settings in whichever application reports the error.

---

# Next Lab Tasks

The next Intune sessions will focus on:

- Create and use a dedicated Windows device group
- Compare user-targeted vs device-targeted configuration
- Deploy an application through Intune
- Verify application deployment on `COMP01`
- Practice Intune policy failures/conflicts
- Learn basic remote device actions
- Understand Windows Autopilot fundamentals
- Continue PowerShell/Microsoft Graph practice
- Begin Microsoft 365 administration

After completing the basic tooling, the lab will transition toward realistic help desk scenarios.

Example:

```text
Ticket Received
      ↓
Gather User Information
      ↓
Determine Scope
      ↓
Check Identity
      ↓
Check Device
      ↓
Check Group Membership
      ↓
Check Intune / M365
      ↓
Verify Local State
      ↓
Identify Root Cause
      ↓
Remediate
      ↓
Verify With User / Device
      ↓
Document Resolution
      ↓
Close or Escalate Ticket
```

The long-term goal is to move from following guided lab instructions toward independently owning junior-level IT tickets from initial report through diagnosis, remediation, verification, documentation, and escalation when necessary.
