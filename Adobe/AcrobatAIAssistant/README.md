# Disable Adobe Acrobat AI Assistant (Enterprise Policy Lockdown)

A standardized registry-based policy override to completely disable and remove the **"Ask AI" (Generative AI Assistant)** buttons, floating sidebars, and contextual prompt menus from Adobe Acrobat Reader and Adobe Acrobat Professional.

This policy applies at the machine layer (`HKEY_LOCAL_MACHINE`) and acts as a structural restriction via Adobe's Enterprise `FeatureLockDown` hive. This ensures the configuration cannot be altered by end-users or reset during background application updates.

---

## Technical Details

* **Registry Value Name:** `bEnableGentech`
* **Registry Value Type:** `REG_DWORD`
* **Value Data:** `0` (Disabled)

### Targeted Registry Paths
To account for varying enterprise application architectures (32-bit legacy deployments vs. modern unified 64-bit installations), the policy targets both standard and architecture-redirected hives:

* **Native 64-bit Path:**
  `HKLM\SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown`
* **32-bit Architecture Redirected Path (WOW64):**
  `HKLM\SOFTWARE\WOW6432Node\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown`

> [!NOTE]
> For environments utilizing licensed versions of **Adobe Acrobat Professional/Standard** instead of the free Reader edition, substitute the `Acrobat Reader` folder segment with `Adobe Acrobat` within the key path.

---

## Deployment via Group Policy (GPO)

To ensure universal coverage across mixed 32-bit and 64-bit endpoints, deploy **two companion registry items** within the same Group Policy Object.

1. Open the **Group Policy Management Editor** and target a policy linked to your endpoint or AVD Session Host OU.
2. Navigate to: **Computer Configuration** ➔ **Preferences** ➔ **Windows Settings** ➔ **Registry**.
3. Right-click in the right pane and select **New** ➔ **Registry Item**.

### Item 1: 32-bit App Architecture (WOW64 Redirect)
* **Action:** `Update`
* **Hive:** `HKEY_LOCAL_MACHINE`
* **Key Path:** `SOFTWARE\WOW6432Node\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown` *(Ensure `HKLM\` is omitted from the beginning of this string)*
* **Value Name:** `bEnableGentech`
* **Value Type:** `REG_DWORD`
* **Value Data:** `0`

### Item 2: Native 64-bit App Architecture
* **Action:** `Update`
* **Hive:** `HKEY_LOCAL_MACHINE`
* **Key Path:** `SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown` *(Ensure `HKLM\` is omitted from the beginning of this string)*
* **Value Name:** `bEnableGentech`
* **Value Type:** `REG_DWORD`
* **Value Data:** `0`

---

## Verification & Troubleshooting

### PowerShell Verification Command
Run the following elevated PowerShell command on the target endpoint to confirm both registry paths have been successfully written:

```powershell
Get-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown", "HKLM:\SOFTWARE\WOW6432Node\Policies\Adobe\Acrobat Reader\DC\FeatureLockDown" -Name "bEnableGentech" -ErrorAction SilentlyContinue | Select-Object @{Name="RegistryPath"; Expression={$_.PSPath.Substring($_.PSPath.IndexOf("HKEY"))}}, bEnableGentech
```