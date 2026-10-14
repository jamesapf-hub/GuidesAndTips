# Guides & Tips Repository Agent Guidelines

These rules and standards apply **STRICTLY AND EXCLUSIVELY** to the `GuidesAndTips` repository (`jamesapf-hub/GuidesAndTips`).

---

## 1. Context & Repository Isolation

1. **Documentation & SOP Scope**: This repository stores technical IT guides, troubleshooting runbooks, architecture designs, and Standard Operating Procedures (SOPs) rendered dynamically inside the Helper Portal's `GuidesTipsBrowserView`.
2. **Zero PowerShell Fast-Execute Cross-Pollination**: NEVER insert PowerShell repository "Fast Execute" banners (`irm phnx.it/... | iex`), `.intunewin` packager templates, or `Start-Gui.bat` launchers into `GuidesAndTips` articles. Diagnostic CLI snippets (PowerShell, CMD, UEFI Shell, Registry) inside guide steps are encouraged where relevant to the troubleshooting procedure.
3. **Deployment Safety**: **ALWAYS** ask for explicit, fresh confirmation from the user before pushing commits to the GitHub remote.

---

## 2. Directory Architecture (Wrapped Topic Pattern)

Guides must be organized under top-level vendor or technology category folders, using either a wrapped topic folder or an `Error/<IssueName>` subfolder containing a `README.md` and any local diagram/screenshot assets:

```text
<Category>/
├── <TopicName>/
│   ├── README.md                    # Main guide document
│   └── <diagram-or-screenshot>.png  # Optional local visual assets
└── Error/                           # Optional grouping folder for specific error remediations
    └── <IssueIdentifier>/
        └── README.md
```

### Established Categories
*   **Adobe:** Enterprise policy lockdowns, Registry customizations, and Acrobat Reader/Pro administration.
*   **Edge:** Microsoft Edge internals, casting/media router diagnostics, and browser policy management.
*   **General:** Portal usage documentation and general cross-platform SOPs.
*   **Intune:** Endpoint architecture designs, Kiosk lockdowns, Autopilot provisioning, and MDM workflows.
*   **Office365:** Microsoft 365, New Outlook (`olk.exe`), Exchange Online, and identity error remediations.
*   **TeamsRooms:** Microsoft Teams Rooms (MTR) for Windows hardware, SMBIOS/UEFI, display, and peripheral troubleshooting.
*   **Windows:** Windows 10/11 OS shell repairs, registry fixes, and system component recovery.

---

## 3. Standardized Guide Structure (`README.md`)

Every guide `README.md` must follow a clean, scannable Markdown hierarchy optimized for the Helper Portal's unified Markdown renderer (`src/utils/markdownUtils.tsx`):

1. **Title (`# <Clear Action-Oriented Title>`)**:
   - The first paragraph immediately below `# Title` (or a `> **Short Description:** ...` callout) is automatically extracted by `sync_baselines.cjs` as the card summary in `metadata.json`. Ensure the opening paragraph is a concise, high-signal 1–2 sentence overview (at least 15 characters).
2. **`## Overview`**:
   - Explain the symptom, the underlying technical root cause, and what the remediation achieves.
3. **`## Prerequisites`** (or Registry/Policy Summary):
   - Required permissions (e.g., Local Administrator, Intune Service Admin, Physical BIOS/UEFI access), target OS/software versions, and required tools.
4. **`## Step-by-Step Guide`** (or Architectural Sections):
   - Numbered subsections (`### 1. ...`, `### 2. ...`) with copy-ready code blocks (`powershell`, `shell`, `reg`, `json`, `txt`).
   - Use GitHub-style alerts (`> [!NOTE]`, `> [!TIP]`, `> [!IMPORTANT]`, `> [!WARNING]`, `> [!CAUTION]`) to highlight root causes, registry caveats, or destructive actions.
5. **`## Verification & Expected Behaviour`**:
   - Clear checklist or command output confirming the issue is resolved.

---

## 4. Mandatory Metadata & Cryptographic Baseline Synchronization

1. **Dynamic `metadata.json` Generation**: The Helper Portal reads `GuidesAndTips/metadata.json` to display folder titles, descriptions, child counts, and `isGuideOrPackage` flags. Never hand-edit `metadata.json` manually if `sync_baselines.cjs` can generate it.
2. **Mandatory Baseline Sync (`npm run sync:baselines`)**: Whenever an agent creates, edits, moves, or deletes any `.md` file in `GuidesAndTips`, the agent **MUST** run `npm run sync:baselines` inside `c:\Users\james\OneDrive\Documents\GitHub\Helper`. This single command:
   - Dynamically regenerates `GuidesAndTips/metadata.json` with extracted titles, summaries, and git timestamps.
   - Normalizes line endings (`\r\n` -> `\n`) and updates the SHA-256 cryptographic trust anchors in Cloudflare D1 (`script_baselines`) so the guide immediately displays as `Verified` in the Helper Portal.
