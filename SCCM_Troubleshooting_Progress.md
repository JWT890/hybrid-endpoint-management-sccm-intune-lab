# SCCM Lab Troubleshooting Progress Report
**Date:** May 30, 2026  
**Environment:** VirtualBox Lab — DC VM, SCCM VM (Primary Site PS1), CLIENT01  
**Goal:** Fix Task Sequence "Installing Application" (PS10000A) failing to install 7-Zip on CLIENT01 via Software Center

---

## Environment Overview

| Component | Details |
|-----------|---------|
| SCCM Server | SCCM.lab.local (192.168.10.20) |
| Client | CLIENT01 (192.168.10.100) |
| Site Code | PS1 |
| Network | 192.168.10.0/24 (VirtualBox internal) |
| Enrollment | Hybrid — all VMs enrolled in Microsoft Intune (Microsoft Entra hybrid joined) |
| Task Sequence | "Installing Application" (PS10000A) — installs 7-Zip via Install Application step |

---

## Root Causes Identified (In Order of Discovery)

### 1. Site Communication Security Set to "HTTPS Only" ❌ → Fixed ✅
**Location:** Administration → Site Configuration → Sites → PS1 → Properties → Communication Security

- Site was configured for **HTTPS only** with no valid PKI infrastructure
- This caused the Management Point (MP) to crash repeatedly with cert errors
- **Fix:** Changed to **"HTTPS or EHTTP"** and forced `SslState = 0` via WMI (console change didn't save due to cert errors blocking the write)

```powershell
$mp = Get-WmiObject -Namespace "root\SMS\site_PS1" -Class "SMS_SCI_SysResUse" |
    Where-Object { $_.RoleName -eq "SMS Management Point" }
$props = $mp.Props
($props | Where-Object { $_.PropertyName -eq "SslState" }).Value = 0
$mp.Props = $props
$mp.Put()
```

---

### 2. Broken SMS Certificate Chain ❌ → Partially Addressed
**Symptom:** `Chain valid: False` — CN=SMS Issuing cert missing from CA store

- The CN=SMS Issuing certificate (issuer of CN=SCCM.lab.local) was missing from `Cert:\LocalMachine\CA`
- SCCM had the private key but not the public cert stored anywhere accessible
- Multiple extraction attempts failed (no AIA extension, no enterprise CA store entry)
- A new self-signed CN=SMS Issuing cert was generated and placed in CA store, but with a different key than what SCCM uses internally
- **Status:** SCCM continues to regenerate CN=SCCM.lab.local using its own internal SMS Issuing key — chain still invalid but MP now works over HTTP so this is no longer blocking

**Key cert details found:**
- Leaf cert thumbprint: `094776B2E458E671B05B39DA989B238B07259A4D`
- Issuer key container: `{022D6A27-1D62-4308-894D-9294227ACC9B}`
- Leaf cert contains `szOID_IntuneTenantOid` and `szOID_IntuneAgentOid` (Intune hybrid join artifacts)

---

### 3. IIS Still Enforcing SSL on SCCM Virtual Directories ❌ → Fixed ✅
**Symptom:** MP crashing with `WINHTTP_CALLBACK_STATUS_FLAG_INVALID_CA`, `error 12175`

- Even after changing SSLState, IIS virtual directories still required SSL
- **Fix:** Removed SSL flags from all SMS IIS virtual directories and removed HTTPS binding

```powershell
Set-WebConfigurationProperty -Filter "system.webServer/security/access" -PSPath "IIS:\Sites\Default Web Site\SMS_MP" -Name "sslFlags" -Value 0
Set-WebConfigurationProperty -Filter "system.webServer/security/access" -PSPath "IIS:\Sites\Default Web Site\CCM_Incoming" -Name "sslFlags" -Value 0
Set-WebConfigurationProperty -Filter "system.webServer/security/access" -PSPath "IIS:\Sites\Default Web Site\CCM_System" -Name "sslFlags" -Value 0
Set-WebConfigurationProperty -Filter "system.webServer/security/access" -PSPath "IIS:\Sites\Default Web Site\CCM_System_WindowsAuth" -Name "sslFlags" -Value 0
Remove-WebBinding -Name "Default Web Site" -Protocol "https" -Port 443
iisreset /restart
```

**Result:** MP initialized successfully — `Http test request succeeded`, `Initialization successfully completed`

---

### 4. Boundary Group Had No Site System Servers ❌ → Fixed ✅
**Symptom:** `"no content locations were found for this program"` in SCClient.log

- "Lab Boundary Group" covered the correct IP range (192.168.10.1–254)
- CLIENT01 IP (192.168.10.100) was in range
- But `SMS_BoundaryGroupSiteSystems` returned **empty** — no DP was linked to the boundary group
- **Fix:** Added `\\SCCM.lab.local` to the boundary group's site system servers via console

**Verification:**
```powershell
Get-WmiObject -Namespace "root\SMS\site_PS1" -Class "SMS_BoundaryGroupSiteSystems" |
    Select-Object GroupID, ServerNALPath
```

---

### 5. AAD Token Errors (0xcaa20009) — Non-Blocking ⚠️
**Symptom:** `Failed to get AAD token. Error 0xcaa20009` in PolicyAgent.log

- Occurs in elevated (admin) PowerShell context — `WamDefaultSet: ERROR 0x80070520`
- Expected behavior: WAM cannot access user tokens in SYSTEM/elevated context
- CLIENT01 is properly hybrid joined: `AzureAdJoined: YES`, `DomainJoined: YES`
- **Status:** Not blocking SCCM functionality — normal for this configuration

---

### 6. PS10000A Task Sequence Package Never Distributed to DP ❌ → IN PROGRESS
**Symptom:** `404 Not Found` when client tries to download from `http://SCCM.lab.local/SMS_DP_SMSPKG$/PS10000A`

- Content library at `C:\SCCMContentLib` confirmed present
- PkgLib contains: PS100001–PS100007, PS100009 — **PS10000A is missing**
- DataLib contains 7-Zip content entries — application content is fine
- "Add Distribution Points" wizard shows no available DPs (console display bug related to cert state)
- distmgr.log shows no distribution activity for PS10000A
- **Fix in progress:** Distributing PS10000A via direct WMI SMS_DistributionPoint class

---

## Current Status of Each Component

| Component | Status | Notes |
|-----------|--------|-------|
| Management Point (HTTP) | ✅ Working | Port 80, initialized successfully |
| Client → MP Communication | ✅ Working | Policy downloading, no HTTPS errors |
| Boundary Group | ✅ Fixed | SCCM.lab.local added as site system |
| 7-Zip Application Content | ✅ Distributed | PS100009 on DP, State 0 |
| 7-Zip Deployment Type | ✅ Correct | Content path: `\\SCCM.lab.local\Sources\Applications\7-Zip\` |
| TS Deployment Targeting | ✅ Correct | PS10000A → Collection PS100015 "Test Lab Computers", CLIENT01 is member |
| Software Center | ✅ Showing TS | "Installing Application" visible, Install button available |
| PS10000A TS Package on DP | ❌ Missing | Not in content library — distribution in progress |
| SMS Cert Chain | ❌ Still broken | MP works over HTTP so not currently blocking |
| HTTPS Binding | ✅ Removed | IIS no longer serving port 443 |

---

## Next Steps to Complete

### Immediate — Distribute PS10000A to DP
```powershell
# Option 1: Direct WMI (run from SCCM server)
$script = @'
$nalPath = '["Display=\\SCCM.lab.local\"]MSWNET:["SMS_SITE=PS1"]\\SCCM.lab.local\'
$mc = [WmiClass]"\\SCCM\root\SMS\site_PS1:SMS_DistributionPoint"
$newObj = $mc.CreateInstance()
$newObj.PackageID = "PS10000A"
$newObj.SiteCode = "PS1"
$newObj.ServerNALPath = $nalPath
$result = $newObj.Put()
Write-Host "Result: $result"
'@
$script | Out-File "C:\temp\distrib.ps1" -Encoding ASCII
PowerShell -ExecutionPolicy Bypass -File "C:\temp\distrib.ps1"

# Then verify in distmgr.log
Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\distmgr.log" -Tail 10 -Wait
```

### After PS10000A Distributes
1. Verify PS10000A.INI appears in `C:\SCCMContentLib\PkgLib\`
2. Test HTTP access: `http://SCCM.lab.local/SMS_DP_SMSPKG$/PS10000A` should return 200
3. On CLIENT01, trigger policy refresh and retry TS from Software Center
4. Monitor `C:\Windows\CCM\Logs\smsts.log` for successful execution

### Optional Cleanup (Post-TS Success)
- Remove the fake CN=SMS Issuing cert from CA store (`f77ab433...`)
- Consider re-enabling EHTTP properly with "Use Configuration Manager-generated certificates"
- Clean up duplicate 7-Zip deployment type entries if still present in WMI
- Prevent HTTPS binding from regenerating (SMS_EXECUTIVE recreates it on restart)

---

## Key Log Files Reference

| Log | Location | Purpose |
|-----|----------|---------|
| smsts.log | `C:\Windows\CCM\Logs\smsts.log` | Task Sequence execution |
| SCClient.log | `C:\Windows\CCM\Logs\SCClient_LAB@sccmadmin_1.log` | Software Center activity |
| CcmMessaging.log | `C:\Windows\CCM\Logs\CcmMessaging.log` | Client→MP communication |
| LocationServices.log | `C:\Windows\CCM\Logs\LocationServices.log` | DP/MP location lookup |
| ContentTransferManager.log | `C:\Windows\CCM\Logs\ContentTransferManager.log` | Content download |
| PolicyAgent.log | `C:\Windows\CCM\Logs\PolicyAgent.log` | Policy download |
| distmgr.log | `C:\Program Files\Microsoft Configuration Manager\Logs\distmgr.log` | DP content distribution |
| mpcontrol.log | `C:\Program Files\Microsoft Configuration Manager\Logs\mpcontrol.log` | MP health |

---

## Key Error Codes Encountered

| Code | Meaning | Resolved? |
|------|---------|-----------|
| 0x87d00215 | No token auth / client lacks PKI cert for SSL | ✅ Yes — HTTP mode |
| 0x87d00231 | HTTP post to MP failed | ✅ Yes — HTTP mode |
| 0x80070002 | File/path not found (TS local data path) | Pending — need PS10000A on DP |
| 0x80004005 | Non-fatal — MP name not in environment variable | Minor |
| 0xcaa20009 | AAD token unavailable in elevated context | ⚠️ Non-blocking |
| 12175 | WinHTTP cert validation failure | ✅ Yes — HTTPS removed |
| WINHTTP_CALLBACK_STATUS_FLAG_INVALID_CA | Invalid CA cert | ✅ Yes — HTTPS removed |
