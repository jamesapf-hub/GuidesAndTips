# Fix Teams Rooms Display Stuck on Single Screen or Zoomed Resolution (SMBIOS AllInOne Mode)

## Overview

Microsoft Teams Rooms (MTR) for Windows deployed on mini-PC compute units (such as Intel or ASUS NUC 13 Pro paired with a touch console like Logitech Tap and external room displays) may encounter a display malfunction where Teams Room only outputs to a single screen, completely ignores secondary displays, or renders an oversized, heavily zoomed-in interface.

This failure occurs because the compute unit's firmware has an active Whiteboard (`WB:{}`) tag provisioned in its SMBIOS Type 11 (OEM Strings) table. When MTR launches, its internal hardware certification engine reads this flag as `touchBoardCapability: true` and forces the application into `--rooms_experience="AllInOne"`. In this mode, Teams Room assumes it is running on a standalone interactive touch board (such as a Surface Hub or Neat Board) and deliberately suppresses standard dual-screen routing and front-of-room stage separation.

This guide provides an end-to-end diagnostic and remediation workflow to confirm the active launch parameters via elevated PowerShell, inspect the firmware capability logs, and overwrite the SMBIOS OEM String using the ASUS NUC Aptio DMI Editor (`iDmiEditEfi64.EFI`) in a pre-boot UEFI shell, permanently restoring standard multi-screen operation (`--rooms_experience="Standard"`).

---

## Prerequisites

* **Access & Permissions:** 
  * Local Administrator privileges on the Teams Room Windows endpoint (for diagnostic queries).
  * Physical access to the compute unit for BIOS/UEFI setup.
* **Hardware & Required Tools:** 
  * Standard USB flash drive (formatted as FAT32 / MBR).
  * 64-bit UEFI Shell binary (`BOOTX64.EFI`).
  * ASUS NUC BIOS Configuration Tool / Aptio DMI Editor (`iDmiEditEfi64.EFI`).
  * Physical USB keyboard connected directly to the NUC.
* **Target Audience:** Unified Communications (UC) Engineers, Audio-Visual (AV) Technicians, Field Support Engineers, Systems Administrators.

---

## Step-by-Step Guide

### 1. Diagnose the Active MTR Process Launch Mode

To verify whether Teams Room is running in standard dual-display mode or forced into single-screen AllInOne mode, query the active process command line while the `Skype` user is signed in:

1. Connect a keyboard or start an elevated PowerShell session (Run as Administrator).
2. Execute the following command:

```powershell
(Get-CimInstance Win32_Process -Filter "Name = 'ms-teams.exe'").CommandLine
```

#### The Diagnosis
* **Standard Expected Output:** The process argument should contain `--rooms_experience="Standard"`.
* **Faulty Output (Root Cause Trigger):** If the command line returns:
  ```text
  --rooms_experience="AllInOne"
  ```
* **What This Means:** Teams Room is actively ignoring secondary HDMI/DisplayPort outputs because it believes the hardware is an all-in-one touch whiteboard. Windows display settings and dual-monitor configurations are overridden by this application flag.

---

### 2. Confirm the SMBIOS String in MTR Diagnostic Logs

To confirm why MTR selected `AllInOne` mode, inspect the local MTR certification telemetry logs:

1. In File Explorer or PowerShell, navigate to the MTR tracing directory:
   `C:\Users\Skype\AppData\Local\Packages\Microsoft.SkypeRoomSystem_8wekyb3d8bbwe\LocalState\Tracing`
2. Search the most recent log file for `MTRCertificationManager` or `touchBoardCapability`.
3. Locate the JSON hardware capability block:

```json
"smbios": {
  "manufacturer": "Intel(R) Client Systems",
  "model": "NUC13L3Kv5",
  "oemJsonString": "{MTR:{PRD:3.1,Level:3,WB:{}}}",
  "touchBoardCapability": true
}
```

> [!IMPORTANT]
> **The Root Cause:**  
> The motherboard firmware contains `WB:{}` (Whiteboard) inside its SMBIOS Type 11 OEM strings. Even when connected to a standalone console (e.g. Logitech Tap) and a front-of-room TV, MTR prioritizes the firmware's `touchBoardCapability: true` flag over Windows display settings and collapses the UI into single-screen touch board mode.

---

### 3. Prepare the Bootable UEFI Shell USB Drive

Because modern Windows 11 builds enforce Hypervisor-Enforced Code Integrity (HVCI) and driver blocklists, executing low-level DMI modification tools inside Windows will fail with `10 - Error: Unable to load driver`. To modify SMBIOS tables safely, use a pre-boot UEFI shell environment.

1. **Format USB Drive:**
   * Insert a USB drive into a workstation.
   * Format the partition as **FAT32** with an **MBR** partition scheme.
2. **Setup UEFI Shell:**
   * Create the directory path `\EFI\BOOT\` on the root of the USB drive.
   * Place a standard 64-bit UEFI Shell binary named `BOOTX64.EFI` inside `\EFI\BOOT\`.
3. **Copy ASUS DMI Utility:**
   * Download the official ASUS NUC BIOS Configuration Tool / Aptio DMI Editor package (from the ASUS support page for the NUC 13 Pro family).
   * Extract `iDmiEditEfi64.EFI` and copy it directly to the root of the USB drive (`\iDmiEditEfi64.EFI`).

---

### 4. Boot into UEFI Shell & Inspect Current OEM Strings

1. Insert the prepared USB drive and a USB keyboard into the NUC.
2. Power on or restart the NUC and repeatedly press **F2** to enter BIOS Setup.
3. In BIOS Setup, navigate to **Boot** > **Secure Boot** and set **Secure Boot** to **Disabled**.
4. Press **F10** to save settings and exit.
5. As the unit reboots, repeatedly tap **F10** to open the Boot Menu and select your **UEFI USB Drive**.
6. The system will drop into the interactive UEFI Shell (`Shell>`).
7. Switch to the USB filesystem (typically `fs0:`):

```shell
fs0:
```

8. Verify the current OEM strings programmed in the motherboard firmware:

```shell
iDmiEditEfi64.EFI /OS
```

> [!NOTE]
> The utility will return the active OEM strings. Confirm that OEM String #1 reads:
> ```text
> OEM string #1: "{MTR:{PRD:3.1,Level:3,WB:{}}}"
> ```

---

### 5. Flash Updated SMBIOS OEM Strings via DMI Tool

1. Overwrite **OEM String #1** to strip out the `WB:{}` whiteboard flag while retaining product and certification levels:

```shell
iDmiEditEfi64.EFI /OS 1 "{MTR:{PRD:3.1,Level:3}}"
```

2. Clear **OEM String #2** to eliminate any stale fallback parameters:

```shell
iDmiEditEfi64.EFI /OS 2 ""
```

3. Query the table to verify the changes have been written to the EEPROM:

```shell
iDmiEditEfi64.EFI /OS
```

Ensure String #1 is now `"{MTR:{PRD:3.1,Level:3}}"` and String #2 is empty.

4. Restart the system:

```shell
reset
```

5. *(Optional / Recommended)* On startup, tap **F2** to re-enter BIOS Setup, re-enable **Secure Boot**, and save changes (**F10**).

---

### 6. Verify Resolution and Dual-Display Routing in Windows

1. Boot normally into Windows and allow the `Skype` user to log in and launch Teams Rooms.
2. Open an elevated PowerShell session and re-verify the active process command line:

```powershell
(Get-CimInstance Win32_Process -Filter "Name = 'ms-teams.exe'").CommandLine
```

3. Confirm that the argument is now `--rooms_experience="Standard"`.
4. Review the MTR tracing logs: `touchBoardCapability` will now record `false`.

#### Expected Resolution & UI Mapping
* **Front-of-Room Display (TV / Projector):** Correctly renders the meeting stage, participant gallery, and shared content in native 1080p/4K resolution.
* **Room Console (Logitech Tap / Touch Controller):** Correctly displays the unzoomed room control interface, call controls, and calendar schedule.

---

## Troubleshooting

> [!WARNING]
> **Issue:** Running DMI tools inside Windows produces `10 - Error: Unable to load driver`.  
> **Solution:** Windows 11 Core Isolation (HVCI) and vulnerable driver blocklists prevent kernel-level DMI drivers from loading in the OS. Always use the UEFI shell method (`iDmiEditEfi64.EFI`) booted via USB as described in Steps 3–5.

> [!WARNING]
> **Issue:** `fs0:` is not recognized or reports "Cannot find drive" in UEFI Shell.  
> **Solution:** Run `map -r` in the shell to re-scan storage devices. If your USB drive is mapped to `fs1:` or `blk0:`, enter `fs1:` and run `ls` to locate `iDmiEditEfi64.EFI`. Also verify that the USB was formatted as FAT32 with MBR (not GPT).

> [!WARNING]
> **Issue:** Windows display output is still mirrored or misplaced after firmware update.  
> **Solution:** In Windows (Admin account), navigate to **Settings > System > Display**. Ensure **Multiple displays** is set to **Extend these displays**, with both displays set to their native resolution (1080p or 4K) and 100% DPI scale.

---

## Related Resources

* [Microsoft Learn: Microsoft Teams Rooms on Windows Known Issues & Display Configuration](https://learn.microsoft.com/en-us/microsoftteams/rooms/rooms-known-issues)
* [Microsoft Learn: Microsoft Teams Rooms Requirements & Certified Peripherals](https://learn.microsoft.com/en-us/microsoftteams/rooms/requirements)
* [ASUS Support: ASUS NUC 13 Pro BIOS & Firmware Configuration Tools](https://www.asus.com/support/)
