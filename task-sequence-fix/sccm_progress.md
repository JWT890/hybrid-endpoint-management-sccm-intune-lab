# SCCM 7-Zip Deployment – Troubleshooting Progress Log
**Lab:** `SCCM.lab.local` | **Site Code:** `PS1` | **Client:** `CLIENT01`
**Goal:** Deploy 7-Zip 26.01 (x64) via SCCM Software Center

---

## ✅ Resolved Issues

### 1. Management Point (MP) Instability
- **Problem:** MP app pools (`SMS Management Point Pool`, `SMS Windows Auth Management Point Pool`) were crashing repeatedly due to Rapid Fail Protection.
- **Fix:** Restarted both app pools and disabled Rapid Fail Protection permanently:
  ```powershell
  Set-ItemProperty "IIS:\AppPools\SMS Management Point Pool" -Name failure.rapidFailProtection -Value $false
  Set-ItemProperty "IIS:\AppPools\SMS Windows Auth Management Point Pool" -Name failure.rapidFailProtection -Value $false
  ```
- **Status:** MP responding HTTP 200 ✅ — but NOTE: `iisreset` re-enables rapid fail protection temporarily; avoid using it.

---

### 2. Content Distribution – Missing Source File
- **Problem:** `\\SCCM.lab.local\Sources\Applications\7-Zip\7z2601-x64.msi` did not exist; only the `.exe` was present. SCCM was distributing an empty/invalid content package, causing persistent `0x80070002` digest errors on the client.
- **Fix:**
  - Created folder: `C:\Sources\Applications\7-Zip\`
  - Downloaded MSI: `Invoke-WebRequest -Uri "https://www.7-zip.org/a/7z2601-x64.msi" -OutFile "C:\Sources\Applications\7-Zip\7z2601-x64.msi"`
  - Verified file: ~2MB ✅
  - Redistributed content to DP (`SCCM.lab.local`) — `distmgr.log` confirmed success.
- **Status:** Content successfully distributed ✅

---

### 3. Deployment Type – "Do Not Download Content" Setting
- **Problem:** Deployment options were set to **"Do not download content"**, so the client tried to run the installer from a local path (`C:\Sources\...`) that doesn't exist on `CLIENT01`.
- **Fix:** Changed to **"Download content from distribution point and run locally"** in the deployment type Content tab.
- **Status:** Fixed ✅

---

### 4. Detection Method – Wrong File Reference
- **Problem:** Detection method was checking for `7z2601-x64.exe` (the installer), not whether 7-Zip was actually installed.
- **Fix:** Replaced with a registry-based detection rule:
  - **Hive:** `HKEY_LOCAL_MACHINE`
  - **Key:** `SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\{23170F69-40C1-2702-2601-000001000000}`
  - **Value:** `DisplayName` | **Operator:** Equals | **Data:** `7-Zip 26.01 (x64 edition)`
- **Status:** Fixed ✅ — client now correctly evaluates 7-Zip as "NotInstalled"

---

### 5. Client HTTPS Forced Communication (Partial)
- **Problem:** `CLIENT01` was configured to prefer HTTPS due to:
  - Cloud Services policy pushing CMG/Entra ID settings
  - Self-signed SMS certificate in `Cert:\LocalMachine\SMS` causing `CcmHttpsState = 1216`
  - Client kept hitting `https://SCCM.lab.local:443/CCMTOKENAUTH_SMS_DP_SMSPKG$` and receiving HTTP 401/405
- **Fixes Applied:**
  - Disabled in Default Client Settings → Cloud Services:
    - "Automatically register devices with Microsoft Entra ID" → **No**
    - "Enable clients to use a cloud management gateway" → **No**
  - Registry keys set on `CLIENT01`:
    ```
    HKLM:\SOFTWARE\Microsoft\CCM
      HTTPS = 0
      CcmHttpsState = 0
      UsePKICert = 0
    ```
  - Removed HTTPS 443 binding from Default Web Site on SCCM server.
  - `CcmHttpsState` now shows empty (not forcing HTTPS) ✅
- **Status:** Partially resolved — client switched to HTTP port 80, but new error encountered (see Current Blocker below).

---

### 6. Site Code Confusion
- **Problem:** Commands were using site code `LAB` instead of the correct code `PS1`, causing WMI namespace errors.
- **Fix:** Identified correct site code as `PS1` via `Get-PSDrive`.
- **Status:** Resolved ✅

---

## 🔴 Current Blocker

### HTTP 405 / WebDAV Issue on DP
- **Where we are:** Client is now correctly attempting content download over **HTTP port 80** (progress from 443 HTTPS).
- **Error:** `GetDirectoryList_HTTP` returns **405 Method Not Allowed** on `http://SCCM.lab.local:80/SMS_DP_SMSPKG$/Content_...`
- **Root cause:** WebDAV not properly enabled or configured on the Default Web Site in IIS.
- **Attempted fix:** Enabling WebDAV via PowerShell failed (config section locked at parent level). Attempted via IIS Manager. `iisreset` after changes caused:
  - MP app pools to drop (503 errors)
  - HTTPS 443 binding to return (SCCM auto-recreates it on iisreset)
- **Next step:** Investigate `CCMTOKENAUTH_SMS_DP_SMSPKG$` virtual directory authentication settings, OR properly configure WebDAV authoring rules without triggering iisreset.

---

## 📋 Environment Notes

| Item | Detail |
|---|---|
| Site Code | PS1 |
| SCCM Server | SCCM.lab.local |
| Client | CLIENT01 |
| App | 7-Zip 26.01 (x64 edition) |
| Deployment Type Name | "Installing Application" |
| Install Command | `msiexec /i "7z2601-x64.msi" /qm` |
| Content ID | `Content_eefec8cc-a785-4d88-80a8-259b472eb54a` |
| Package ID | PS100009 |
| Source Path | `C:\Sources\Applications\7-Zip\` (share: `\\SCCM.lab.local\Sources\`) |
| Client Version | 5.00.9141.1011 |

---

## ⚠️ Known Quirks / Watch-Outs

- **Avoid `iisreset`** — it re-enables Rapid Fail Protection on MP app pools and restores the HTTPS 443 binding. Use `Restart-WebAppPool` instead.
- **`TriggerSchedule` WMI method** on CLIENT01 sometimes fails with "not found" — use Control Panel → Configuration Manager → Actions tab as an alternative.
- **Registry HTTPS overrides are temporary** — `CcmExec` can reset them on restart. The permanent fix is the Cloud Services client policy change (already made).
- **Backtick (`) line continuation** in PowerShell causes the shell to wait for more input — always paste commands as a single line.
