# SCCM Distribution Point Recovery — Progress Log

**Site:** PS1  
**Server:** SCCM.lab.local  
**Goal:** Distribute 7-Zip application (PS10000B) to DP and recover Task Sequence deployment to CLIENT01  
**Status:** 🔄 In Progress

---

## Problem Summary

The SCCM Distribution Point on the co-located site server (`SCCM.lab.local`) became broken. The root symptom was CLIENT01 receiving error `0x87d00215` (content not found) when running the Task Sequence containing a 7-Zip install step.

---

## Root Causes Identified (in order of discovery)

| # | Finding | Impact |
|---|---------|--------|
| 1 | 7-Zip app had no backing content package (PackageType=8 missing from WMI/DB) | `Start-CMContentDistribution` silently did nothing |
| 2 | Content package PS10000B created but distmgr fired instant STATMSG 2301 with AID0=400 | Phantom "already distributed" state blocked all transfers |
| 3 | DP role `IsActive=False` — DP marked inactive in database | MP excluded DP from client content location responses |
| 4 | IIS virtual directories (`SMS_DP_SMSPKG$`, `SMS_DP_SMSSIG$`) missing after DP reinstall | HTTP 404 on all DP content endpoints |
| 5 | IIS returned 403.18 on `SMS_DP_SMSPKG$` | ISAPI `smsfileisapi.dll` doing cross-app-pool `ExecuteURL` blocked by IIS 10 |
| 6 | `applicationHost.config` location block contained full inherited ASP.NET handler list with `ExtensionlessUrlHandler` (`path=".*"`) catching all requests before SMS ISAPI handler | SMS ISAPI never reached |
| 7 | `SMS_DP_SMSPKG$` location block missing entirely after edits | HTTP 404 |
| 8 | `IsActive` being reset to `False` by SCCM health evaluation cycle after each SQL/WMI fix | Distribution target rejected by `Start-CMContentDistribution` |

---

## Actions Taken

---

### Phase 1 — Diagnose Missing Content Package

**Problem:** `Start-CMContentDistribution` completed silently with no distmgr activity.

**Diagnosis commands:**

```powershell
# Check if content package exists
Get-WmiObject -Namespace "root\SMS\site_PS1" `
    -Class SMS_PackageBaseclass `
    -Filter "PackageType=8" |
    Select-Object PackageID, Name, SourceVersion, StoredVersion |
    Format-Table -AutoSize

# Confirm app object exists
Set-Location "PS1:"
$app = Get-CMApplication -Name "7-Zip"
Write-Host "CI_ID: $($app.CI_ID)"
Write-Host "IsEnabled: $($app.IsEnabled)"
Write-Host "NumberOfDeploymentTypes: $($app.NumberOfDeploymentTypes)"
```

**Finding:** No PackageType=8 object existed — deployment type had been created but SCCM never generated the backing content package.

**Fix:**

```powershell
Set-Location "PS1:"

# Force CI reprocessing by updating the deployment type
$dt = Get-CMDeploymentType -ApplicationName "7-Zip"

Set-CMMsiDeploymentType `
    -ApplicationName "7-Zip" `
    -DeploymentTypeName $dt.LocalizedDisplayName `
    -ContentLocation "\\SCCM.lab.local\Sources\Applications\7-Zip\7z2601-x64.msi" `
    -AddDetectionClause (New-CMDetectionClauseFile `
        -FileName "7z.exe" `
        -Path "%ProgramFiles%\7-Zip" `
        -Existence) `
    -Force

Start-Sleep -Seconds 20

# Confirm PS10000B now exists
Get-WmiObject -Namespace "root\SMS\site_PS1" `
    -Class SMS_PackageBaseclass `
    -Filter "PackageType=8" |
    Select-Object PackageID, Name, SourceVersion |
    Format-Table
```

**Result:** Content package **PS10000B** created, SourceVersion=2, ContentId=`ab81dd81-7626-42ec-96e2-8c0f0ed44c0a`, 2 source files confirmed.

---

### Phase 2 — Confirm DP Is Broken

```powershell
# Check DP active state
Get-CMDistributionPointInfo -SiteSystemServerName "SCCM.lab.local" |
    Select-Object Name, IsActive, Communication | Format-List

# Test IIS endpoints
Invoke-WebRequest http://SCCM.lab.local/SMS_DP_SMSSIG$ -UseBasicParsing
Invoke-WebRequest http://SCCM.lab.local/SMS_DP_SMSPKG$ -UseBasicParsing

# Check IIS virtual directories
Import-Module WebAdministration
Get-WebVirtualDirectory -Site "Default Web Site" |
    Select-Object Name, PhysicalPath | Format-Table -AutoSize

# Check IIS web applications
Get-WebApplication -Site "Default Web Site" |
    Select-Object Name, ApplicationPool, PhysicalPath | Format-Table -AutoSize

# Check IIS site and services
Get-WebSite | Select-Object Name, State, PhysicalPath | Format-List
Get-Service W3SVC, WAS | Select-Object Name, Status | Format-Table
```

**Result:** `IsActive=False`, HTTP 404 on both DP endpoints, virtual directories absent.

---

### Phase 3 — DP Role Removal and Reinstall

**Via SCCM Console:**
- Administration → Site Configuration → Servers and Site System Roles
- SCCM.lab.local → Distribution Point → Remove Role
- After removal: Add Site System Roles → Distribution Point (HTTP, no PXE, no schedule)

**Monitor removal:**

```powershell
Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\distmgr.log" `
    -Tail 10 -Wait
```

**Verify IIS cleaned up after removal:**

```powershell
Import-Module WebAdministration
Get-WebVirtualDirectory -Site "Default Web Site" |
    Select-Object Name, PhysicalPath | Format-Table -AutoSize
```

**Verify reinstall activity:**

```powershell
Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\distmgr.log" `
    -Tail 10 -Wait
```

**Result:** distmgr log showed `Skipping OS configuration for distribution point SCCM.lab.local. You should install and configure IIS manually.` — co-located site server behaviour; SCCM does not auto-create IIS virtual directories on reinstall.

---

### Phase 4 — IIS Virtual Directory Reconstruction

**Create missing folder and virtual directories:**

```powershell
# Create the physical folder
New-Item -Path "C:\SMSSIG$" -ItemType Directory -Force

Import-Module WebAdministration

# Create virtual directories
New-WebVirtualDirectory `
    -Site "Default Web Site" `
    -Name "SMS_DP_SMSPKG$" `
    -PhysicalPath "C:\SCCMContentLib"

New-WebVirtualDirectory `
    -Site "Default Web Site" `
    -Name "SMS_DP_SMSSIG$" `
    -PhysicalPath "C:\SMSSIG$"

# Verify
Get-WebVirtualDirectory -Site "Default Web Site" |
    Select-Object Name, PhysicalPath | Format-Table -AutoSize
```

**After SMS_EXECUTIVE restart, SCCM rebuilt SMS_DP_SMSSIG$ itself. Verify final IIS state:**

```powershell
# Check all 4 DP applications
Get-WebApplication -Site "Default Web Site" |
    Where-Object { $_.Name -match "SMS_DP|CCMTOKEN" } |
    Select-Object Name, ApplicationPool, PhysicalPath | Format-Table -AutoSize

# Confirm from raw applicationHost.config
[xml]$config = Get-Content "C:\Windows\System32\inetsrv\config\applicationHost.config"
$site = $config.configuration.'system.applicationHost'.sites.site |
    Where-Object { $_.name -eq "Default Web Site" }
$site.application |
    Where-Object { $_.path -match "SMS_DP" } |
    ForEach-Object {
        Write-Host "App: $($_.path) Pool: $($_.applicationPool)"
        $_.virtualDirectory | ForEach-Object {
            Write-Host "  VDir: $($_.path) -> $($_.physicalPath)"
        }
    }
```

---

### Phase 5 — 403.18 Forbidden Investigation

**Test endpoints:**

```powershell
Invoke-WebRequest "http://SCCM.lab.local/SMS_DP_SMSPKG$/PS10000B" -UseBasicParsing
Invoke-WebRequest "http://SCCM.lab.local/SMS_DP_SMSSIG$" -UseBasicParsing
```

**Check ISAPI handler and restriction:**

```powershell
# Verify smsfileisapi.dll exists
Test-Path "C:\Windows\System32\inetsrv\smsfileisapi.dll"
Get-Item "C:\Windows\System32\inetsrv\smsfileisapi.dll" |
    Select-Object Name, Length, LastWriteTime

# Check ISAPI restriction
Get-WebConfiguration "system.webServer/security/isapiCgiRestriction" |
    Select-Object -ExpandProperty Collection |
    Where-Object { $_.path -match "smsfile" } |
    Select-Object path, allowed, description

# Check handler on the application
Get-WebConfiguration "system.webServer/handlers" `
    -PSPath "IIS:\Sites\Default Web Site\SMS_DP_SMSPKG$" |
    Select-Object -ExpandProperty Collection |
    Where-Object { $_.Name -match "SMS" } |
    Select-Object Name, path, verb, modules, scriptProcessor | Format-List

# Check what app pool Default Web Site root uses
Get-WebConfiguration "system.applicationHost/sites/site[@name='Default Web Site']/application[@path='/']" |
    Select-Object path, applicationPool
```

**Finding:** `smsfileisapi.dll` allowed=True, handler present. Root issue: `SMS_DP_SMSPKG$` ran in `SMS Distribution Points Pool`; Default Web Site root uses `DefaultAppPool`. IIS 10 blocks cross-pool `ExecuteURL` calls from ISAPI → 403.18.

**Check applicationHost.config handler bloat:**

```powershell
# Find the location block and check its size
$content = Get-Content "C:\Windows\System32\inetsrv\config\applicationHost.config" -Raw
$match = [regex]::Match($content,
    '<location path="Default Web Site/SMS_DP_SMSPKG[^"]*"[\s\S]*?</location>')
Write-Host "Block length: $($match.Value.Length) chars"
Write-Host $match.Value.Substring(0, 500)

# View handlers via appcmd
& "C:\Windows\System32\inetsrv\appcmd.exe" list config "Default Web Site/SMS_DP_SMSPKG$" /section:handlers
```

**Finding:** Location block was ~7500 chars — full inherited ASP.NET handler list with `<clear />` followed by `ExtensionlessUrlHandler` (`path=".*"`) catching all requests before the SMS ISAPI handler, which was listed last.

---

### Phase 6 — Fix applicationHost.config App Pool and Handler Block

**Change SMS_DP_SMSPKG$ to DefaultAppPool:**

```powershell
$configPath = "C:\Windows\System32\inetsrv\config\applicationHost.config"
$config = Get-Content $configPath -Raw

# Locate exact string first
$idx = $config.IndexOf('application path="/SMS_DP_SMSPKG')
Write-Host $config.Substring($idx, 150)

# Replace app pool
$old = 'application path="/SMS_DP_SMSPKG$" applicationPool="SMS Distribution Points Pool"'
$new = 'application path="/SMS_DP_SMSPKG$" applicationPool="DefaultAppPool"'

if ($config.Contains($old)) {
    $config = $config.Replace($old, $new)
    [System.IO.File]::WriteAllBytes($configPath, [System.Text.Encoding]::UTF8.GetBytes($config))
    Write-Host "Done"
    # Verify
    $verify = Get-Content $configPath -Raw
    $idx = $verify.IndexOf('application path="/SMS_DP_SMSPKG')
    Write-Host $verify.Substring($idx, 100)
}
```

**Replace bloated handler location block with clean minimal version:**

```powershell
$configPath = "C:\Windows\System32\inetsrv\config\applicationHost.config"
$config = Get-Content $configPath -Raw

# Get the exact current block
$idx = $config.IndexOf('<location path="Default Web Site/SMS_DP_SMSPKG$">')
$end = $config.IndexOf('</location>', $idx) + '</location>'.Length
$oldBlock = $config.Substring($idx, $end - $idx)
Write-Host "Old block length: $($oldBlock.Length)"

# Build clean replacement — security settings preserved, only SMS handler, no <clear />
$newBlock = '<location path="Default Web Site/SMS_DP_SMSPKG$">' +
'<system.webServer><security><authentication>' +
'<windowsAuthentication enabled="true" />' +
'<anonymousAuthentication enabled="true" />' +
'</authentication><access sslFlags="None" /></security>' +
'<directoryBrowse enabled="true" />' +
'<staticContent><mimeMap fileExtension=".*" mimeType="DP_ALL_FILETYPES" /></staticContent>' +
'<handlers accessPolicy="Read, Execute">' +
'<add name="SMS_DP_SMSPKG" path="*" verb="*" modules="IsapiModule" scriptProcessor="C:\Windows\system32\inetsrv\smsfileisapi.dll" requireAccess="None" responseBufferLimit="0" />' +
'</handlers></system.webServer></location>'

$newConfig = $config.Replace($oldBlock, $newBlock)

if ($newConfig.Length -ne $config.Length) {
    [System.IO.File]::WriteAllBytes($configPath, [System.Text.Encoding]::UTF8.GetBytes($newConfig))
    Write-Host "Written successfully"
} else {
    Write-Host "ERROR: length unchanged"
}
```

**Restart IIS and test:**

```powershell
iisreset /noforce
Start-Sleep -Seconds 20

Invoke-WebRequest "http://SCCM.lab.local/SMS_DP_SMSPKG$" -UseBasicParsing
Invoke-WebRequest "http://SCCM.lab.local/SMS_DP_SMSSIG$/PS100002.1.tar" -UseBasicParsing
```

**Result:**
- `SMS_DP_SMSPKG$` → **404** (ISAPI executing correctly, no content at bare path — expected ✓)
- `SMS_DP_SMSSIG$/PS100002.1.tar` → **200 OK**, 6,009,856 bytes ✓

**IIS layer confirmed working.**

---

### Phase 7 — IsActive=False Investigation

**Check IsActive from WMI and SQL:**

```powershell
# WMI check
Get-WmiObject -Namespace "root\SMS\site_PS1" `
    -Class SMS_DistributionPointInfo `
    -Filter "ServerName='SCCM.LAB.LOCAL'" |
    Select-Object ServerName, IsActive | Format-List

# SQL — find the base table
$query = "SELECT TABLE_NAME FROM INFORMATION_SCHEMA.TABLES WHERE TABLE_NAME LIKE '%DistPoint%' ORDER BY TABLE_NAME"
Invoke-Sqlcmd -ServerInstance "SCCM.LAB.LOCAL" -Database "CM_PS1" -Query $query |
    Select-Object -ExpandProperty TABLE_NAME

# SQL — read current state
$query = "SELECT DPID, ServerName, IsActive, DPCRC FROM DistributionPoints WHERE NALPath LIKE '%SCCM%'"
Invoke-Sqlcmd -ServerInstance "SCCM.LAB.LOCAL" -Database "CM_PS1" -Query $query | Format-List
```

**Finding:** `DPID=2003`, `IsActive=False`, `DPCRC=3612FE25` in SQL.

**Attempted fixes:**

```powershell
# Attempt 1 — WMI Put() — FAILED (IsActive is read-only in WMI)
$dp = Get-WmiObject -Namespace "root\SMS\site_PS1" `
    -Class SMS_DistributionPointInfo `
    -Filter "ServerName='SCCM.LAB.LOCAL'"
$dp.IsActive = $true
$dp.Put()

# Attempt 2 — Force DP config re-evaluation
Set-Location "PS1:"
Set-CMDistributionPoint -SiteSystemServerName "SCCM.lab.local" -SiteCode "PS1"
Start-Sleep -Seconds 60
Get-WmiObject -Namespace "root\SMS\site_PS1" `
    -Class SMS_DistributionPointInfo `
    -Filter "ServerName='SCCM.LAB.LOCAL'" |
    Select-Object ServerName, IsActive | Format-List

# Attempt 3 — Direct SQL update
$update = "UPDATE DistributionPoints SET IsActive=1 WHERE DPID=2003"
Invoke-Sqlcmd -ServerInstance "SCCM.LAB.LOCAL" -Database "CM_PS1" -Query $update

# Verify SQL update
$verify = "SELECT DPID, ServerName, IsActive FROM DistributionPoints WHERE DPID=2003"
Invoke-Sqlcmd -ServerInstance "SCCM.LAB.LOCAL" -Database "CM_PS1" -Query $verify | Format-List

# Restart SMS_EXECUTIVE to flush WMI cache
Restart-Service SMS_EXECUTIVE
Start-Sleep -Seconds 60

Get-WmiObject -Namespace "root\SMS\site_PS1" `
    -Class SMS_DistributionPointInfo `
    -Filter "ServerName='SCCM.LAB.LOCAL'" |
    Select-Object ServerName, IsActive | Format-List
```

**Result:** SQL update sets `IsActive=True` in DB, but SCCM health evaluation overwrites it back to `False` on the next cycle. Root cause of health evaluation failure not yet identified.

**Check distmgr for clues:**

```powershell
# Watch distmgr for DP health activity
Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\distmgr.log" -Tail 30

# Search for relevant patterns
Select-String `
    -Path "C:\Program Files\Microsoft Configuration Manager\Logs\distmgr.log" `
    -Pattern "IsActive|SCCM.LAB.LOCAL|active|health|DPCRC" |
    Select-Object -Last 15 | Format-List Line

# Check sitecomp for role evaluation
Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\sitecomp.log" -Tail 30
```

---

## Current State

| Component | Status |
|-----------|--------|
| 7-Zip app object | ✅ Exists, CI_ID=16780531, IsEnabled=True |
| Content package PS10000B | ✅ Exists, SourceVersion=2, 2 files |
| IIS W3SVC / WAS | ✅ Running |
| SMS_DP_SMSSIG$ HTTP | ✅ 200 OK — serving content |
| SMS_DP_SMSPKG$ HTTP | ✅ ISAPI executing (404 on bare path = correct) |
| All 4 IIS DP applications | ✅ Present and mapped correctly |
| DP role registered | ✅ In boundary groups, NAL path correct |
| IsActive (WMI/SQL) | ❌ False — being reset by SCCM health check each cycle |
| Start-CMContentDistribution | ❌ "No content destination found" — IsActive=False blocks it |
| distmgr snapshot for PS10000B | ❌ Not generated — AID0=400 phantom state |
| CLIENT01 content resolution | ❌ 0x87d00215 — MP returns 0 locations |

---

## Next Steps

### 1. Identify what SCCM health check is failing

```powershell
# Check all DP-related logs for health evaluation failures
Get-ChildItem "C:\Program Files\Microsoft Configuration Manager\Logs\" |
    Where-Object { $_.Name -match "dp|monitor|health" } |
    Select-Object Name, LastWriteTime | Format-Table -AutoSize

# Check distmgr post-SMS_EXECUTIVE restart
Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\distmgr.log" -Tail 30 |
    Select-String "IsActive|active|health|SCCM.LAB|ConfigureDP|DPCRC" |
    Format-List

# Check sitecomp for DP evaluation
Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\sitecomp.log" -Tail 30

# Investigate DPCRC mismatch — config checksum may be triggering inactive state
Select-String `
    -Path "C:\Program Files\Microsoft Configuration Manager\Logs\distmgr.log" `
    -Pattern "DPCRC|CRC|3612FE25" |
    Select-Object -Last 10 | Format-List Line
```

### 2. Clear PS10000B phantom distribution state

```powershell
# Check current phantom state
$q = "SELECT * FROM vSMS_DistributionDPStatus WHERE PackageID='PS10000B'"
Invoke-Sqlcmd -ServerInstance "SCCM.LAB.LOCAL" -Database "CM_PS1" -Query $q | Format-List

# If State=0 exists (phantom success), delete it
$del = "DELETE FROM PkgStatus WHERE PackageID='PS10000B' AND Type=2"
Invoke-Sqlcmd -ServerInstance "SCCM.LAB.LOCAL" -Database "CM_PS1" -Query $del
```

### 3. Distribute 7-Zip content (once IsActive=True is stable)

```powershell
Set-Location "PS1:"
Get-CMApplication -Name "7-Zip" |
    Start-CMContentDistribution -DistributionPointName "SCCM.lab.local"
Write-Host "Submitted at $(Get-Date)"
Set-Location "C:"

# Watch distmgr — must see "Taking snapshot", NOT instant 2301
Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\distmgr.log" `
    -Tail 5 -Wait

# Watch PkgXferMgr for actual file transfer
Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\PkgXferMgr.log" `
    -Tail 10 -Wait

# Verify content landed
Get-ChildItem "C:\SCCMContentLib\PkgLib\" | Where-Object { $_.Name -match "PS10000B" }
Get-ChildItem "C:\SMSSIG$\" | Select-Object Name, Length, LastWriteTime
```

### 4. Re-add 7-Zip to Task Sequence

```powershell
Set-Location "PS1:"
$freshApp = Get-CMApplication -Name "7-Zip"
Add-CMTaskSequenceStep `
    -TaskSequenceId "PS10000A" `
    -Step (New-CMTaskSequenceStepInstallApplication `
        -Name "Install 7-Zip" `
        -Application $freshApp)
Write-Host "Step added: $($freshApp.ModelName)"
Set-Location "C:"
```

### 5. Retry Task Sequence on CLIENT01

```powershell
# On CLIENT01
Stop-Service CcmExec -Force
Start-Sleep -Seconds 15
Start-Service CcmExec
Start-Sleep -Seconds 60

# Trigger machine policy
([wmiclass]"root\ccm:SMS_Client").TriggerSchedule("{00000000-0000-0000-0000-000000000021}")
Start-Sleep -Seconds 45

# Watch content resolution — expect CLSCallback LocationUpdateEx2 with 1+ locations
Get-Content "C:\Windows\CCM\Logs\ContentTransferManager.log" -Tail 10 -Wait
```

---

## Reference

### Key Package IDs

| ID | Description |
|----|-------------|
| PS10000A | Task Sequence |
| PS10000B | 7-Zip application content package (PackageType=8) |
| PS100002 | Existing package (healthy, distributing OK) |
| PS100003 | Existing package (healthy, distributing OK) |

### Key Log Files

| Log | Location | Purpose |
|-----|----------|---------|
| distmgr.log | SCCM server Logs\ | Package queuing, snapshot, transfer jobs |
| PkgXferMgr.log | SCCM server Logs\ | Actual file movement to DP |
| sitecomp.log | SCCM server Logs\ | Site component / role health evaluation |
| ContentTransferManager.log | CLIENT01 CCM\Logs\ | Client content location resolution |
| CAS.log | CLIENT01 CCM\Logs\ | MP content location responses |

### Key Paths

| Path | Purpose |
|------|---------|
| `C:\Windows\System32\inetsrv\config\applicationHost.config` | IIS configuration |
| `C:\SCCMContentLib\PkgLib\` | Content library index (.INI per package) |
| `C:\SMSSIG$\` | Signature .tar files (one per distributed package) |
| `C:\SCCMContentLib\` | Main content library |

### applicationHost.config Backups

| File | Timestamp | Notes |
|------|-----------|-------|
| `applicationHost.config.bak_20260605_220204` | 6/5/2026 9:53 PM | Earliest available |
| `applicationHost.config.bak3_20260605_225046` | 6/5/2026 9:53 PM | Same timestamp |
| `applicationHost.config.bak3_20260605_230744` | 6/5/2026 10:55 PM | Pre-location block edit |
| `applicationHost.config.bak4_20260605_233337` | 6/5/2026 10:55 PM | Last good before failed regex edit |

### STATMSG Reference

| ID | Meaning |
|----|---------|
| 2300 | Distribution to DP started |
| 2301 | Distribution to DP completed |
| 2330 | Distribution to DP failed/retrying |
| 2375 | DP install completed |
| AID0=400 | DP reported content already present (phantom state) |
| LE=0X0 | Success |
