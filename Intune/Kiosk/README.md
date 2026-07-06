# Architecture Design: Intune-Managed Mock Thin Client

This document outlines the design, implementation details, and workflow for the **Mock Thin Client** solution. The goal of this solution is to repurpose standard Windows devices into secure, single-purpose thin clients managed via Microsoft Intune, providing a streamlined user experience without the complexity of traditional thin-client operating systems or write filters.

---

## Architecture Overview

The Mock Thin Client leverages **Windows Autopilot Self-Deploying mode** to provision devices, installs the **Windows 365 app (Windows App)** as the primary user interface, locks the session environment using **Intune Kiosk configuration**, restricts logon access using **Entra ID Security Groups**, and implements a custom **PowerShell-driven Scheduled Task** to manage the user session lifecycle.

---

## 1. Device Enrollment & Provisioning (Autopilot)

To automate the setup of Mock Thin Clients with zero touch from IT, we utilize Windows Autopilot.

### Zero-Touch Self-Deployment
* **Autopilot Group Tag**: Devices designated as Mock Thin Clients are registered in Windows Autopilot with a specific Group Tag, for example: `MockThinClient`.
* **Dynamic Device Group**: An Entra ID dynamic device group targets these devices using the Group Tag:
  ```txt
  (device.devicePhysicalIds -any (_ -eq "[OrderID]:MockThinClient"))
  ```
* **Self-Deploying Profile**: The dynamic group is assigned an Autopilot profile configured for **Self-Deploying mode** (Preview).
* **No User Login Required during Provisioning**: 
  * Unlike standard user-driven Autopilot, Self-Deploying mode requires **zero user interaction or credentials** during the Out-of-Box Experience (OOBE).
  * The device joins Entra ID, registers with Intune, downloads policies, and installs the Windows 365 app entirely in the background.
  * Once the automated provisioning is complete, the device boots straight to the Windows logon screen, fully secured and ready for an authorized user to sign in.

### Wiping and Re-provisioning
Should a device experience issues or require a fresh deployment, resetting it is highly streamlined:
* **Intune Remote Wipe**: Administrators can trigger an **Autopilot Reset** or **Wipe** command directly from the Microsoft Intune Admin Center.
* **Dell OS Support Recovery**: If utilizing Dell hardware, technicians or on-site staff can boot into the built-in **Dell SupportAssist OS Recovery** tool to quickly restore the system image. Once the OS is recovered to factory settings and boots up, Windows Autopilot immediately takes over and automatically re-provisions the machine back into a Mock Thin Client without IT intervention.

---

## 2. Security & Access Control

Access to the Mock Thin Client is tightly controlled to prevent unauthorized usage and ensure only administrators can access the full operating system.

### Entra ID Group Restriction (`SG_Intune_MockThinClient_User`)
* Only users belonging to the Entra ID security group **`SG_Intune_MockThinClient_User`** are allowed to sign in.
* **Local Administrators**: Only local administrator accounts configured directly on the device are permitted to sign in to perform maintenance or troubleshooting. 
  > [!IMPORTANT]
  > Testing has shown that Entra ID Join Administrators do not have access; sign-in is strictly limited to local administrator accounts.

---

## 3. Kiosk Mode & Windows App Execution

Once an authorized user signs in, the device behaves like a dedicated thin client by constraining the user's environment.

* **Single-App Kiosk Configuration**: 
  * Configured via an Intune Kiosk Device Configuration profile.
  * The logon type is set to **Azure AD User** (allowing users to log in with their own corporate credentials).
  * The application type is set to run the **Windows 365 app**.
* **Environment Lockdown**:
  * Windows Explorer, the Taskbar, the Start Menu, and standard desktop shortcuts are completely disabled.
  * The Windows 365 app launches automatically in full-screen mode immediately after the user logs in.

---

## 4. Session Lifecycle Management (Automatic Logoff)

Because the user is running a single-app kiosk, closing the application should end their Windows session and return the device to the login screen for the next user. We implement an automatic logoff mechanism using a PowerShell script and a Scheduled Task.

### The Platform Script & Scheduled Task
Intune deploys a **PowerShell Platform Script** running in the `SYSTEM` context that configures a local session monitor.
* **Deploying the Monitor Script**: The platform script writes a monitoring PowerShell script to a secure local directory.
* **Creating the Scheduled Task**: The platform script registers a local Scheduled Task that runs at logon of any standard user.
* **Session Monitoring & Logoff**: The scheduled task runs a background monitoring script in the user's session context. It waits for the Windows 365 app to launch, monitors its execution, and automatically initiates a Windows logoff immediately when the Windows 365 app is closed. This ensures that users cannot access any underlying OS interface on exit.

---

## 5. Benefits of the Solution

Implementing the Mock Thin Client with this design yields several operational and security advantages over traditional Windows lockdowns:

| Feature | Mock Thin Client Approach | Traditional Windows Lockdown / Thin PC |
| :--- | :--- | :--- |
| **Write Filter** | **No Unified Write Filter (UWF)**. Devices use standard NTFS write capabilities. | Uses UWF, which often causes disk space exhaustion, BSODs, and breaks Windows Updates. |
| **Account Type** | **Individual Entra ID Accounts**. Users sign in with their own corporate credentials. | Generic shared local accounts, posing security, compliance, and auditing risks. |
| **Security Controls** | Regulated by group membership (`SG_Intune_MockThinClient_User`) and Conditional Access. | Rely on physical access control or shared passwords. |
| **Maintenance** | Standard Intune updates, patching, and policy syncs function seamlessly. | Complex orchestration required to disable UWF, reboot, patch, re-enable UWF, and reboot. |

---

## 6. Caveats & Operational Considerations

While this design is highly efficient, there are several caveats to keep in mind:

> [!WARNING]
> **Admin Logon Bypass**: Because local administrators bypass the Kiosk environment to perform maintenance, ensure that local administrator credentials are secure (e.g., managed via Windows LAPS).

> [!IMPORTANT]
> **Group Membership Sync**: Since access is governed by the `SG_Intune_MockThinClient_User` group, a newly added user must wait for Entra ID group membership replication before they can successfully sign in to a Mock Thin Client device.
