# SCCM Lab — Full Troubleshooting Report & Fix Commands
**Date:** May 31, 2026  
**Environment:** VirtualBox lab — DC VM, SCCM VM (SCCM.lab.local), CLIENT01 VM  
**Site Code:** PS1  
**Original Goal:** Task Sequence "Installing Application" (PS10000A) deploying 7-Zip to CLIENT01 via Software Center

---

## Environment Summary

| Item | Value |
|------|-------|
| SCCM Server | SCCM.LAB.LOCAL (MP + DP + Site Server) |
| Client | CLIENT01 (192.168.10.100) |
| Domain | LAB.local |
| SQL Instance | MSSQLSERVER (default) — DB: CM_PS1 |
| Boundary | Lab Range 192.168.10.1–192.168.10.254 |
| Entra Join | Hybrid (Intune co-management enabled) |

---

## Key Package IDs

| Package | ID | Notes |
|---------|-----|-------|
| Task Sequence | PS10000A | "Installing Application" |
| 7-Zip Application | PS100009 | Distributed, 100% compliance |
| CM Client Package | PS100001 | Distributed |
| 7-Zip Content Hash | Content_eefec8cc-a785-4d88-80a8-259b472eb54a.1 | In DataLib |

---

## Key File/Path Reference

| Item | Path |
|------|------|
| SCCM Logs | C:\Program Files\Microsoft Configuration Manager\Logs\ |
| DP Provider Log | C:\SMS_DP$\sms\logs\smsdpprov.log |
| Content Library | C:\SCCMContentLib\ |
| PkgLib | C:\SCCMContentLib\PkgLib\ |
| DataLib | C:\SCCMContentLib\DataLib\ |
| 7-Zip Source | C:\Sources\Applications\7-Zip\7z2601-x64.msi |
| TS Source (created) | C:\Sources\TaskSequence\InstallingApplication\ |
| IIS appcmd | C:\Windows\System32\inetsrv\appcmd.exe |
| SQL DB | CM_PS1 on localhost (MSSQLSERVER) |

---

## Root Causes Summary

| # | Root Cause | Symptom | Status |
|---|-----------|---------|--------|
| 1 | Site set to HTTPS-only with no PKI | MP crashes, client 0x87D00215 auth errors | ✅ Fixed |
| 2 | Boundary group had no site systems assigned | "No content locations found" on client | ✅ Fixed |
| 3 | DP HTTP provider not registered in IIS | HTTP 404 on SMS_DP_SMSPKG$, download cancels | ✅ Fixed |

---

## Root Cause 1: Site Communication Mode Set to HTTPS-Only (No PKI)

**Symptom:** MP crashes with WINHTTP_CALLBACK_STATUS_FLAG_INVALID_CA; CcmMessaging.log shows 0x87D00215 (CCM_E_NO_TOKEN_AUTH); client cannot authenticate to MP.

**Why it happened:** Site was configured for "HTTPS only" (SSLState=63) but no valid PKI certificate chain existed. The SMS cert chain was broken (CN=SMS Issuing missing from CA store), causing the MP to crash on every HTTPS attempt.

### Step 1 — Force MP SSLState to 0 via WMI
```powershell
# Run on SCCM server as Administrator
$mp = Get-WmiObject -Namespace "root\SMS\site_PS1" -Class "SMS_SCI_SysResUse" |
    Where-Object { $_.RoleName -eq "SMS Management Point" }
($mp.Props | Where-Object { $_.PropertyName -eq "SslState" }).Value = 0
$mp.Put()
Write-Host "MP SSLState updated"
```

### Step 2 — Force DP SSLState to 0 via WMI
```powershell
$dp = Get-WmiObject -Namespace "root\SMS\site_PS1" -Class "SMS_SCI_SysResUse" |
    Where-Object { $_.RoleName -eq "SMS Distribution Point" }
($dp.Props | Where-Object { $_.PropertyName -eq "SslState" }).Value = 0
$dp.Put()
Write-Host "DP SSLState updated"
```

### Step 3 — Verify both are now 0
```powershell
Get-WmiObject -Namespace "root\SMS\site_PS1" -Class "SMS_SCI_SysResUse" |
    Where-Object { $_.RoleName -match "Management Point|Distribution Point" } |
    ForEach-Object {
        $ssl = $_.Props | Where-Object { $_.PropertyName -eq "SslState" }
        Write-Host "$($_.RoleName): SslState = $($ssl.Value)"
    }
# Expected: SslState = 0 for both
```

### Step 4 — Remove SSL requirement from IIS virtual directories
```powershell
cd C:\Windows\System32\inetsrv

.\appcmd set config "Default Web Site/SMS_MP" /section:access /sslFlags:None /commit:apphost
.\appcmd set config "Default Web Site/CCM_Incoming" /section:access /sslFlags:None /commit:apphost
.\appcmd set config "Default Web Site/CCM_System" /section:access /sslFlags:None /commit:apphost
.\appcmd set config "Default Web Site/CCM_System_WindowsAuth" /section:access /sslFlags:None /commit:apphost
```

### Step 5 — Remove HTTPS binding from IIS
```powershell
.\appcmd set site "Default Web Site" /-bindings.[protocol='https',bindingInformation='*:443:']
```

### Step 6 — Restart IIS and SMS_EXECUTIVE
```powershell
iisreset /restart
Start-Sleep 30
Restart-Service SMS_EXECUTIVE -Force
Start-Sleep 90
```

### Step 7 — Verify MP is healthy
```powershell
Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\mpcontrol.log" -Tail 20
# Look for: "Http test request succeeded" and "Initialization successfully completed"
```

### Step 8 — Fix client communication mode on CLIENT01
```powershell
# Run on CLIENT01 as Administrator
reg add "HKLM\SOFTWARE\Microsoft\CCM" /v "CcmHTTPSState" /t REG_DWORD /d 0 /f
reg add "HKLM\SOFTWARE\Microsoft\CCM" /v "CcmHTTPPort" /t REG_DWORD /d 80 /f
reg add "HKLM\SOFTWARE\Microsoft\CCM" /v "SMSMP" /t REG_SZ /d "SCCM.lab.local" /f
reg add "HKLM\SOFTWARE\Microsoft\CCM" /v "ClientAlwaysOnInternet" /t REG_DWORD /d 0 /f

Stop-Service CcmExec -Force
Start-Sleep 10
Start-Service CcmExec
Start-Sleep 45
```

### Step 9 — Verify client can reach MP
```powershell
# Run on CLIENT01
Invoke-WebRequest -Uri "http://SCCM.lab.local/sms_mp/.sms_aut?MPList" -UseBasicParsing |
    Select-Object StatusCode, Content
# Expected: StatusCode 200
```

---

## Root Cause 2: Boundary Group Has No Site Systems Assigned

**Symptom:** CLIENT01 is in the correct IP range but receives "no content locations found"; LocationServices.log shows "MP error threshold reached, moving to next MP"; TS download cancels immediately.

**Why it happened:** The boundary existed and covered the correct IP range, but no DP or MP was added to the boundary group's site systems list. Without this link, SCCM never offers the DP to the client.

### Step 1 — Confirm boundary group exists and CLIENT01 is in range
```powershell
# Run on SCCM server
Get-WmiObject -Namespace "root\SMS\site_PS1" -Class "SMS_BoundaryGroup" |
    Select-Object Name, GroupID | Format-Table -AutoSize

Get-WmiObject -Namespace "root\SMS\site_PS1" -Class "SMS_Boundary" |
    Select-Object DisplayName, BoundaryType, Value | Format-Table -AutoSize

# Check CLIENT01 IP
# Run on CLIENT01:
(Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.IPAddress -notmatch "127|169" }).IPAddress
```

### Step 2 — Confirm no site systems are assigned (the problem)
```powershell
Get-WmiObject -Namespace "root\SMS\site_PS1" -Class "SMS_BoundaryGroupSiteSystems" |
    Select-Object GroupID, ServerNALPath | Format-Table -AutoSize
# If this returns nothing — the boundary group has no DP/MP assigned
```

### Step 3 — Fix via SCCM Console
```
Administration
→ Hierarchy Configuration
→ Boundary Groups
→ double-click your boundary group (e.g. "Lab Boundary Group")
→ References tab
→ Under "Site systems" click Add
→ select \\SCCM.lab.local
→ OK → Apply → OK
```

### Step 4 — Verify fix
```powershell
Get-WmiObject -Namespace "root\SMS\site_PS1" -Class "SMS_BoundaryGroupSiteSystems" |
    Select-Object GroupID, ServerNALPath | Format-Table -AutoSize
# Should now show the DP NAL path for your boundary group
```

### Step 5 — Force client to refresh location
```powershell
# Run on CLIENT01
([wmiclass]"root\ccm:SMS_Client").TriggerSchedule("{00000000-0000-0000-0000-000000000021}")
Start-Sleep 30
([wmiclass]"root\ccm:SMS_Client").TriggerSchedule("{00000000-0000-0000-0000-000000000032}")
Start-Sleep 30

Get-Content "C:\Windows\CCM\Logs\LocationServices.log" -Tail 20
# Look for: DP being offered, no more "MP error threshold reached"
```

---

## Root Cause 3: DP HTTP Provider Not Registered in IIS

**Symptom:** HTTP 404 on `http://SCCM.lab.local/SMS_DP_SMSPKG$/...`; smsdpprov.log does not exist; ContentTransferManager.log shows download canceled with 0x87D00215; TS bounces back to "Available" in Software Center immediately after clicking Install.

**Why it happened:** The original HTTPS-only misconfiguration prevented the DP role from properly initializing its IIS components. The smsfileisapi.dll ISAPI handler was never registered, and the SMS_DP_SMSPKG$ virtual directory was either missing or pointing to the wrong path (C:\SMSPKGC$ instead of C:\SCCMContentLib).

### Step 1 — Confirm the problem
```powershell
# Run on SCCM server
Test-Path "C:\Program Files\Microsoft Configuration Manager\Logs\smsdpprov.log"
# False = DP provider never initialized

Test-Path "C:\SMS_DP$\sms\logs\smsdpprov.log"
# False = same problem, different log location

& "$env:SystemRoot\system32\inetsrv\appcmd.exe" list vdir | Select-String "SMS_DP"
# Missing SMS_DP_SMSPKG$ or pointing to wrong path = problem confirmed

Invoke-WebRequest -Uri "http://localhost/SMS_DP_SMSPKG$/PS100009" `
    -UseBasicParsing -ErrorAction SilentlyContinue | Select-Object StatusCode
# 404 = DP not serving content
```

### Step 2 — Remove and re-add the Distribution Point role via console
```
Administration
→ Site Configuration
→ Servers and Site System Roles
→ SCCM.lab.local
→ right-click "Distribution Point" → Remove Role
→ wait 2-3 minutes for removal to complete

→ right-click SCCM.lab.local → Add Site System Roles
→ check "Distribution Point" → Next
→ Communication tab:
    • select "EHTTP or HTTPS"
    • check "Allow clients to connect anonymously"
    • select "Create self-signed certificate"
→ Drive Settings: Automatic → Next
→ Boundary Groups: add "Lab Boundary Group" → Next
→ Complete wizard
```

### Step 3 — Wait for DP to install and verify
```powershell
# Watch sitecomp for DP installation
Get-Content "C:\Program Files\Microsoft Configuration Manager\Logs\sitecomp.log" -Tail 5 -Wait
# Look for: "Detected a change to sitectrl.box" then DP-related install messages

# Poll for smsdpprov.log to appear (may take 2-5 minutes)
while (-not (Test-Path "C:\SMS_DP$\sms\logs\smsdpprov.log")) {
    Write-Host "Waiting for DP provider... $(Get-Date -Format 'HH:mm:ss')"
    Start-Sleep 15
}
Write-Host "DP provider initialized!"
Get-Content "C:\SMS_DP$\sms\logs\smsdpprov.log" -Tail 20
```

### Step 4 — Verify correct IIS virtual directories were created
```powershell
& "$env:SystemRoot\system32\inetsrv\appcmd.exe" list vdir | Select-String "SMS_DP"
# Expected:
# VDIR "Default Web Site/SMS_DP_SMSPKG$/"  (physicalPath:C:\SCCMContentLib)
# VDIR "Default Web Site/CCMTOKENAUTH_SMS_DP_SMSSIG$/"  (physicalPath:C:\SMSSIG$)
```

### Step 5 — Remove any incorrectly created virtual directories
```powershell
# If a bad vdir exists pointing to C:\SMSPKGC$ instead of C:\SCCMContentLib, remove it
& "$env:SystemRoot\system32\inetsrv\appcmd.exe" delete vdir "Default Web Site/SMS_DP_SMSPKG%"
# Adjust the name to match whatever incorrect vdir exists (% vs $)
```

### Step 6 — Verify smsfileisapi.dll is allowed and registered
```powershell
# Check ISAPI restrictions
& "$env:SystemRoot\system32\inetsrv\appcmd.exe" list config /section:isapiCgiRestriction |
    Select-String "sms"
# Should show: allowed="true" for smsfileisapi.dll with groupId="SMS_DP"

# Check handler registered on SMS_DP_SMSPKG$ vdir
& "$env:SystemRoot\system32\inetsrv\appcmd.exe" list config "Default Web Site/SMS_DP_SMSPKG$" /section:handlers |
    Select-String "smsfile"
# Should show: scriptProcessor="...smsfileisapi.dll"
```

### Step 7 — Test content delivery end-to-end
```powershell
# Get a content hash from a known distributed package's INI file
Get-Content "C:\SCCMContentLib\PkgLib\PS100009.INI"
# Note the Content_xxxx value under [Packages]

# Test HTTP delivery using that hash (replace with your actual hash)
Invoke-WebRequest -Uri "http://localhost/SMS_DP_SMSPKG$/Content_eefec8cc-a785-4d88-80a8-259b472eb54a.1" `
    -UseBasicParsing | Select-Object StatusCode
# Expected: 200
```

### Step 8 — Restart services and trigger client retry
```powershell
# On SCCM server
Restart-Service SMS_EXECUTIVE -Force
Start-Sleep 60

# On CLIENT01
Stop-Service CcmExec -Force
Start-Sleep 10
Start-Service CcmExec
Start-Sleep 60

([wmiclass]"root\ccm:SMS_Client").TriggerSchedule("{00000000-0000-0000-0000-000000000021}")
Start-Sleep 30
```

### Step 9 — Retry Task Sequence and monitor
```powershell
# On CLIENT01 — open Software Center and click Install on the Task Sequence
# Monitor these logs simultaneously:

Get-Content "C:\Windows\CCM\Logs\ContentTransferManager.log" -Tail 10 -Wait
# Look for: successful download, no 0x87D00215 errors

Get-Content "C:\Windows\CCM\Logs\smsts.log" -Tail 20 -Wait
# Look for: new entries with today's date, "Installing application 7-Zip", "Task sequence completed"
```

---

## Full Verification Checklist (Run After All Three Fixes)

```powershell
# Run on SCCM server — confirms everything is in order

Write-Host "=== MP SSLState (expect 0) ==="
Get-WmiObject -Namespace "root\SMS\site_PS1" -Class "SMS_SCI_SysResUse" |
    Where-Object { $_.RoleName -eq "SMS Management Point" } |
    ForEach-Object { ($_.Props | Where-Object { $_.PropertyName -eq "SslState" }).Value }

Write-Host "=== DP SSLState (expect 0) ==="
Get-WmiObject -Namespace "root\SMS\site_PS1" -Class "SMS_SCI_SysResUse" |
    Where-Object { $_.RoleName -eq "SMS Distribution Point" } |
    ForEach-Object { ($_.Props | Where-Object { $_.PropertyName -eq "SslState" }).Value }

Write-Host "=== Boundary Group Site Systems (expect DP NAL path) ==="
Get-WmiObject -Namespace "root\SMS\site_PS1" -Class "SMS_BoundaryGroupSiteSystems" |
    Select-Object GroupID, ServerNALPath

Write-Host "=== MP HTTP Test (expect 200) ==="
Invoke-WebRequest -Uri "http://SCCM.lab.local/sms_mp/.sms_aut?MPList" -UseBasicParsing |
    Select-Object StatusCode

Write-Host "=== DP Content Test (expect 200) ==="
Invoke-WebRequest -Uri "http://localhost/SMS_DP_SMSPKG$/Content_eefec8cc-a785-4d88-80a8-259b472eb54a.1" `
    -UseBasicParsing -ErrorAction SilentlyContinue | Select-Object StatusCode

Write-Host "=== smsdpprov.log exists (expect True) ==="
Test-Path "C:\SMS_DP$\sms\logs\smsdpprov.log"

Write-Host "=== PkgLib contents ==="
Get-ChildItem "C:\SCCMContentLib\PkgLib\" | Select-Object Name

Write-Host "=== IIS SMS_DP virtual directories ==="
& "$env:SystemRoot\system32\inetsrv\appcmd.exe" list vdir | Select-String "SMS_DP"
```

---

## Notes & Lessons Learned

- **Set EHTTP from day one.** In a lab without PKI, always configure the site to "HTTPS or EHTTP" during initial setup under Administration → Site Configuration → Sites → PS1 → Properties → Communication Security. This prevents the entire cascade of issues encountered here.
- **Boundary group site systems are mandatory.** A boundary covering the right IP range does nothing unless a DP is explicitly added to the boundary group's References tab.
- **The DP role must be removed and re-added** to cleanly re-register smsfileisapi.dll in IIS after a communication mode change. Simply restarting SMS_EXECUTIVE is not sufficient.
- **smsdpprov.log lives at C:\SMS_DP$\sms\logs\** not in the main SCCM logs folder.
- **SMS cert chain issues** (CN=SMS Issuing missing) are a symptom of HTTPS misconfiguration in a lab, not a root cause. Switching to EHTTP/HTTP makes them irrelevant.
- **Application-based Task Sequences** (using Install Application step) do not require a PkgSourcePath on the TS package itself. The referenced application's content (7-Zip, PS100009) is what must be distributed to the DP.
