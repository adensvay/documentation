# Microsoft Intune Fundamentals Lab

**Date:** September 30, 2026

## Objective

The goal of this session was to gain hands-on experience with Microsoft Intune and understand how Entra ID identities, security groups, managed Windows devices, policies, compliance requirements, and applications interact in a Microsoft cloud environment.

This session focused on learning the basic Intune administration workflow rather than advanced endpoint management.

---

## Resources

### Andy Malone - Windows 11 / Microsoft Intune Tutorial

I followed an Andy Malone Microsoft Intune tutorial to get an introduction to:

- Windows device management
- Configuration profiles
- Compliance policies
- Application deployment
- Device synchronization
- Group-based assignments

The tutorial provided a useful starting point for exploring the Intune admin center and then testing these concepts in my own lab environment.

---

## Lab Environment

Current Microsoft lab environment:

- Windows Server 2022 Active Directory
- Windows 11 test endpoint
- Microsoft Entra ID
- Microsoft Entra Connect
- Microsoft Intune
- Microsoft Graph
- PowerShell 7
- Entra security groups for IT, Sales, and HR

Basic architecture:

On-Premises Active Directory
        |
        | Entra Connect
        v
Microsoft Entra ID
        |
        +--- Users
        +--- Security Groups
        +--- Devices
        |
        v
Microsoft Intune
        |
        +--- Configuration Profiles
        +--- Compliance Policies
        +--- Application Deployment
        |
        v
Managed Windows 11 Endpoint

---

# What I Practiced

## 1. Intune Device Management

I explored an existing Windows 11 endpoint that had previously been enrolled in Microsoft Intune.

I reviewed information such as:

- Device name
- Operating system
- Management status
- Compliance status
- User association
- Device check-in
- Device synchronization

This helped reinforce the distinction between Entra ID and Intune.

### Key Concept

Microsoft Entra ID primarily handles identity and access:

- Users
- Groups
- Authentication
- Device identities
- Roles

Microsoft Intune primarily handles endpoint management:

- Device configuration
- Compliance
- Application deployment
- Device monitoring
- Endpoint management actions

---

## 2. Configuration Policies

I created and assigned a Windows configuration policy through Intune.

Basic workflow:

Configuration Policy
        |
        v
Security Group Assignment
        |
        v
Managed Windows Device
        |
        v
Device Sync
        |
        v
Policy Applied

This demonstrated how administrators can centrally configure Windows endpoints rather than manually configuring every computer.

---

## 3. Compliance Policies

I created a Windows compliance policy and assigned it using Entra security groups.

Compliance policies allow Intune to evaluate whether managed endpoints meet organizational requirements.

I learned the difference between configuration and compliance:

**Configuration Policy**

> Tells the endpoint how it should be configured.

**Compliance Policy**

> Evaluates whether the endpoint satisfies defined requirements.

This distinction is important because a device can be managed by Intune while still being considered noncompliant.

---

## 4. Group-Based Application Deployment

I configured an application deployment and targeted it toward specific Entra security groups.

Example:

SG-Sales
    |
    v
Intune Application Assignment
    |
    v
Sales Users / Devices

This demonstrated how applications can be deployed based on organizational membership instead of manually installing software on every endpoint.

I also explored the distinction between:

- Required applications
- Available applications
- Uninstall assignments

This provides centralized software lifecycle management for managed endpoints.

---

## 5. User vs. Device Targeting

One important concept from this session was understanding the difference between targeting users and targeting devices.

Example user-based deployment:

John Smith
    |
    v
SG-Sales
    |
    v
Sales Application
    |
    v
John receives applicable assignment

Example device-based deployment:

WIN11-CLIENT
    |
    v
SG-Intune-TestDevices
    |
    v
Windows Configuration Policy
    |
    v
Configuration applied to device

User targeting is useful when a resource should follow a particular employee.

Device targeting is useful when a configuration should apply to a particular endpoint regardless of the user.

---

# Troubleshooting Observation

During testing, I attempted to sign into the Windows 11 VM using a cloud-created Entra user.

Windows returned an error indicating that the domain was unavailable.

The user currently exists in Entra ID but was created after synchronization problems occurred between my on-premises Active Directory environment and Entra ID.

Because the Windows VM is associated with the existing domain environment, this created an identity mismatch between:

- On-premises Active Directory
- Entra ID
- Windows authentication

I intentionally left this unresolved during this session because the objective was learning Intune rather than troubleshooting Entra Connect.

This issue will be investigated separately.

---

# Intune Troubleshooting Workflow

A useful troubleshooting process I learned from this lab:

User reports missing policy/application
        |
        v
Identify User + Device
        |
        v
Verify Intune Enrollment
        |
        v
Check Last Device Check-In
        |
        v
Check User/Device Group Membership
        |
        v
Check Intune Assignment
        |
        v
Check Deployment Status
        |
        v
Sync Device
        |
        v
Review Errors / Conflicts / Applicability
        |
        v
Verify Resolution

This provides a structured process for investigating Intune-related support tickets instead of immediately changing settings.

---

# Key Takeaways

- Entra ID and Intune perform different but complementary roles.
- Entra security groups can control Intune policy and application assignments.
- Configuration policies define endpoint settings.
- Compliance policies evaluate endpoint requirements.
- Applications can be centrally deployed through Intune.
- Intune assignments can target users or devices.
- Group membership is an important part of troubleshooting deployments.
- Device synchronization/check-in is important when testing new assignments.
- Successful endpoint management requires understanding both identity and device state.

---

# Next Steps

Future lab sessions will include:

- Troubleshoot Entra Connect synchronization
- Practice additional Intune troubleshooting scenarios
- Test application deployment from the endpoint perspective
- Explore Microsoft 365 administration
- Continue PowerShell / Microsoft Graph automation
- Practice complete employee onboarding and offboarding workflows
