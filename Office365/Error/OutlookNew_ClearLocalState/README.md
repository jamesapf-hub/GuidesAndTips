# Fix New Outlook for Windows Stuck Opening (olk.exe --clearLocalState)

## Overview
> **Short Description:** Fixes New Outlook for Windows stuck on launch or spinning on startup by clearing corrupted local cache using olk.exe --clearLocalState.

New Outlook for Windows (the modern web-based Outlook client built on WebView2) can sometimes become unresponsive on launch. Users frequently experience issues where the application gets stuck on the loading screen, hangs with a spinning ring, or crashes immediately upon opening.

This guide details how to execute the native command `olk.exe --clearLocalState` to forcefully purge the local app data state, cached WebView2 web storage, and corrupted local session parameters. This performs a clean reset of the client without deleting email, contacts, or calendar data stored on Microsoft 365 cloud servers.

### Known Issues Fixed
* **Stuck on Startup:** New Outlook hangs indefinitely on "Opening Outlook..." or splash screen.
* **Blank / White Screen on Launch:** Application window opens but displays a blank frame due to corrupted WebView2 cache.
* **Startup Crashing:** Client opens briefly and immediately closes or crashes.
* **Authentication Loop:** Continuous prompt for credentials on launch that fails to complete.

## Prerequisites
* **Access/Permissions:** Standard User context (Administrator rights not required).
* **Required Tools:** Windows Run Dialog (`Win + R`) or Command Prompt / PowerShell.
* **Target Audience:** Helpdesk Technicians, IT Engineers, End Users.
* **Supported OS:** Windows 10 / Windows 11 with New Outlook for Windows installed.

## Step-by-Step Guide

### 1. Close All Active Outlook Processes
Before clearing the local state, ensure all instances of New Outlook are closed:
1. Right-click the Windows **Start Menu** and select **Task Manager** (or press `Ctrl + Shift + Esc`).
2. Look for **Outlook (New)** or `olk.exe` in the processes list.
3. If running, select it and click **End Task**.
   
   > [!TIP]
   > You can also force-close New Outlook from Command Prompt or PowerShell:
   > ```cmd
   > taskkill /F /IM olk.exe
   > ```

### 2. Open the Windows Run Dialog
Press the **Windows Key + R** on your keyboard to open the **Run** dialog box.

### 3. Execute the Clear Local State Command
In the Run box, type or copy/paste the following exact command:

```cmd
olk.exe --clearLocalState
```

Press **Enter** or click **OK**.

> [!NOTE]
> `olk.exe` is the official binary executable alias for New Outlook for Windows registered under Windows App Execution Aliases.

### 4. Verify Re-initialization & Re-launch
1. New Outlook will launch automatically and reset its local application state.
2. The client will perform a fresh initialization and re-sync your Microsoft 365 mailbox profile.
3. Verify that the application opens cleanly to your inbox.

## Troubleshooting

> [!WARNING]
> **Common Issue:** `Windows cannot find 'olk.exe'. Make sure you typed the name correctly...`
> **Solution:** The App Execution Alias for New Outlook may be disabled or un-registered.
> 1. Open Windows **Settings** (`Win + I`) > **Apps** > **Advanced app settings** > **App execution aliases**.
> 2. Ensure the toggle switch for **olk.exe** (Outlook for Windows) is turned **On**.
> 3. If the alias remains missing, execute the clear local state directly via the package URL schema:
>    ```cmd
>    start ms-outlook:--clearLocalState
>    ```

> [!NOTE]
> **Alternative Reset via PowerShell:**
> If the command line switch does not resolve the startup crash, repair or reset the AppX package using PowerShell:
> ```powershell
> Get-AppxPackage -Name "Microsoft.OutlookForWindows" | Reset-AppxPackage
> ```

## Related Resources
* [Troubleshoot Outlook for Windows Issues (Microsoft Learn)](https://support.microsoft.com/en-us/office/troubleshoot-outlook-for-windows-issues-241bb0fc-b601-443f-a64f-4d43e82d7904)
