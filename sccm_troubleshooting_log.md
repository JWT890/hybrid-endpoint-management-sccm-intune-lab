# SCCM Lab Troubleshooting Log
**Environment:** SCCM Primary Site PS1 — SCCM.lab.local  
**Version:** 5.00.9141.1000 (ConfigMgr Current Branch)  
**OS:** Windows Server 2022 Standard Evaluation  
**SQL:** SQL Server 2019 (MSSQL17.MSSQLSERVER)  
**Account:** LAB\sccmadmin (hybrid identity — synced to sccmadmin@jonlab2026.onmicrosoft.com)

---

## Problem Statement

Distribution Point on SCCM.lab.local appeared stuck with:
- `IsActive: False`
- `HealthCheckEnabled: False`
- `LastHealthCheckTime: (blank)`

SCCM console opened but displayed a blank navigation tree (no workspaces loaded).

---

## Investigation Timeline

### Phase 1 — DP Activation Attempts
**Symptoms observed:**
- `IsActive: False`, `HealthCheckEnabled: False` in WMI
- sitecomp.log looping: *"Detected a change to sitectrl.box / A new site control file was not available"*
- distmgr.log repeating: *"DP monitoring task is disabled / No need to initialize monitoring task"*

**Actions taken:**
- Restarted SMS_SITE_COMPONENT_MANAGER and SMS_EXECUTIVE
- Ran BranchCache toggle to force SCF commit → sitecomp processed cleanly but DP still inactive
- Ran Pull DP toggle → failed silently (SourceDistributionPoint not set)
- Set ContentValidation schedule → processed but no change
- Confirmed boundary group `Lab Boundary Group` had `\\SCCM.lab.local` assigned
- Distributed 7-Zip (PS10000B) → STATMSG 2301 success, content on DP → still `IsActive: False`

**Conclusion at end of Phase 1:**  
All standard DP activation paths exhausted. DP appeared installed and content distributed successfully, but activation state never flipped. Suspected WMI data unreliable.

---

### Phase 2 — Console Blank / SMS Provider Failure
**Symptoms observed:**
- Console connects (title bar shows PS1) but navigation tree never renders
- Event log: `ProviderLoadFailure — SELECT * FROM SMS_Site WHERE SiteCode = 'PS1'`

**Actions taken:**
- Cleared console cache (`%APPDATA%\Microsoft\MMC\`, `%LOCALAPPDATA%\Microsoft\ConfigMgr10`)
- Checked smsprov.log → found: `Using User : GUEST`
- Verified Guest account already disabled — made no difference
- Ran `winmgmt /verifyrepository` → **consistent** (WMI repository not corrupted)
- Ran `mofcomp smsprov.mof -autorecover` → compiled successfully ("Done!") but `__Win32Provider` query returned empty
- Full server reboot + chkdsk ran on C: drive automatically
- Post-reboot: WMI healthy, Kerberos working, but GUEST fallback persisted

**Key finding:** `smsprov.mof` declares `HostingModel = "LocalSystemHost:CMProv"` and registers the provider under `root\SMS\site_PS1` — but the `__Win32Provider` instance was never being written to the repository despite successful mofcomp.

---

### Phase 3 — Stale SID in MOF Security Descriptor
**Symptoms observed:**
- MOF contains `NamespaceSecuritySDDL` with SID `S-1-5-21-2460854176-365065659-1558237133-1015`
- This SID resolved to **SCCM\SMS Admins** — a different domain (`SCCM`) than the current environment (`LAB`)
- LAB\sccmadmin SID: `S-1-5-21-1582941424-275186334-2922663849-1105` — completely different domain prefix

**Root cause identified (partial):**  
SCCM was originally installed in a domain called `SCCM`. The environment was rebuilt with a new domain `LAB`, but the MOF still contained the old domain's SID. Every connection attempt failed security context resolution and fell back to GUEST.

**Actions taken:**
- Added `LAB\sccmadmin` to local `SMS Admins` group
- Confirmed local `SMS Admins` group membership: `LAB\sccm$` and `LAB\sccmadmin`
- Updated WMI namespace ACL on `root\SMS\site_PS1` via wmimgmt.msc to add `LAB\sccmadmin`
- Removed stale `SCCM\SMS Admins` entry from namespace ACL
- Set `LocalAccountTokenFilterPolicy = 1` in registry + rebooted
- Fixed DCOM Default Impersonation Level from **Identify** → **Impersonate**
- Verified DCOM Launch and Activation permissions (Administrators, SYSTEM, NETWORK SERVICE all correct)

**Result:** GUEST fallback persisted. SMS Provider still crashing with `DLL_PROCESS_DETACH` immediately after loading.

---

### Phase 4 — SQL Server Not Running (Root Cause)
**Discovery:**  
Attempted `Invoke-Sqlcmd -ServerInstance "SCCM.lab.local" -Database "CM_PS1"` →  
**"The server was not found or was not accessible. TCP Provider, error: 0 — The remote computer refused the network connection."**

Checked services:
```
MSSQLSERVER     Stopped    SQL Server (MSSQLSERVER)
```

**Event log errors (repeating since at least 9:34 PM):**
```
Unable to load user-specified certificate [Cert Hash(sha1) "D8A05EE8BF0FE5F28B988CEE024794D6A415D2E2"]
TDSSNIClient initialization failed — Unable to initialize SSL support. Cannot find object or property.
Could not start the network library because of an internal error in the network library.
SQL Server could not spawn FRunCommunicationsManager thread.
```

**Registry confirmed:**
```
HKLM\SOFTWARE\Microsoft\Microsoft SQL Server\MSSQL17.MSSQLSERVER\MSSQLServer\SuperSocketNetLib
Certificate = d8a05ee8bf0fe5f28b988cee024794d6a415d2e2
```

The certificate with that thumbprint **no longer existed in the Windows certificate store**. SQL Server was configured to use it for encrypted connections, couldn't load it on startup, and crashed before accepting any connections.

---

## Root Cause

**A SQL Server SSL certificate was deleted from the Windows certificate store while SQL Server still had its thumbprint bound in the registry.**

On every startup attempt, SQL Server tried to load the certificate, failed, and terminated before accepting connections. This caused the following cascade:

```
Missing SSL cert
  → SQL Server crashes on startup (MSSQLSERVER: Stopped)
    → SMS Provider cannot connect to CM_PS1 database
      → smsprov.dll loads then immediately unloads (DLL_PROCESS_DETACH)
        → WMI falls back to anonymous/GUEST context
          → All WMI queries against root\SMS\site_PS1 fail
            → SCCM console navigation tree never loads (blank)
              → All DP status data (IsActive, HealthCheckEnabled) unreliable
                → Hours of DP troubleshooting on false data
```

---

## Fix Applied

### Step 1 — Identify SQL instance registry path
```powershell
reg query "HKLM\SOFTWARE\Microsoft\Microsoft SQL Server\Instance Names\SQL"
# Result: MSSQLSERVER = MSSQL17.MSSQLSERVER
```

### Step 2 — Clear the broken certificate binding
```powershell
Set-ItemProperty `
  -Path "HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server\MSSQL17.MSSQLSERVER\MSSQLServer\SuperSocketNetLib" `
  -Name "Certificate" -Value "" -Type String

# Verify
Get-ItemProperty `
  -Path "HKLM:\SOFTWARE\Microsoft\Microsoft SQL Server\MSSQL17.MSSQLSERVER\MSSQLServer\SuperSocketNetLib" `
  -Name "Certificate" | Select-Object Certificate
# Result: Certificate = (empty)
```

### Step 3 — Start SQL Server
```powershell
Start-Service MSSQLSERVER
Get-Service MSSQLSERVER | Select-Object Name, Status
# Result: MSSQLSERVER = Running
```

### Step 4 — Restart SMS_EXECUTIVE
```powershell
Restart-Service SMS_EXECUTIVE -Force
Start-Sleep -Seconds 60
```

---

## Verification

### SMS Provider — Working
```
smsprov.log:
  CSspClassManager::Initialize
  Reading SQL db info from registry for sitecode PS1
  Set SQL Machine SCCM.lab.local from registry
  Execute WQL = select * from SMS_Site~
  Results returned: 1 of 2~
  ExecQueryAsync: COMPLETE
  CExtUserContext::LeaveThread : Releasing IWbemContextPtr
```
No more `Using User : GUEST`. No more `DLL_PROCESS_DETACH`.

### WMI Query — Working
```powershell
Get-WmiObject -Namespace "root\SMS\site_PS1" -Class "SMS_Site" |
    Select-Object SiteCode, SiteName, Version

SiteCode : PS1
SiteName : Primary Site 1
Version  : 5.00.9141.1000
```

### SQL Direct Query — Working
```powershell
Invoke-Sqlcmd -ServerInstance "localhost" -Database "CM_PS1" `
    -Query "SELECT SiteCode, SiteName, Version FROM Sites"

SiteCode : PS1
SiteName : Primary Site 1
Version  : 5.00.9141.1000
```

### SCCM Console — Working
Console loads fully. Connected to PS1, Primary Site 1 — SCCM.lab.local.  
Navigation tree renders all workspaces: Assets and Compliance, Software Library, Monitoring, Administration.

### Distribution Point — Working
**Monitoring → Distribution Status → Distribution Point Configuration Status:**
- SCCM.LAB.LOCAL: **green circle**
- Status Type: **Success**
- Success: 3 | In Progress: 0 | Failed: 0

distmgr.log confirms:
- `GetDPUsableDrives` ✓
- `GetContentLibLocation` ✓
- `~Starting the DP cert monitoring thread` ✓
- `STATMSG 2300/2301` (content distribution success) ✓

---

## Key Lessons Learned

1. **`IsActive: False` in WMI does not mean the DP is broken.** The console's Distribution Point Configuration Status is the authoritative source — not WMI properties.

2. **Validate the data source before trusting diagnostic data.** All WMI queries throughout this session were returning unreliable data because the SMS Provider was crashing. Hours were spent troubleshooting DP symptoms that didn't exist.

3. **SQL Server must be running before any SCCM diagnostics are meaningful.** The SMS Provider depends entirely on SQL connectivity. A stopped SQL Server produces cascading failures that look like dozens of different unrelated problems.

4. **`Using User : GUEST` in smsprov.log is a symptom, not a root cause.** It indicates the provider's security context collapsed — which can happen for many upstream reasons including SQL connectivity failure.

5. **Check foundational services first.** Before debugging SCCM-specific components, verify: SQL Server running → SMS Provider responding → WMI queries returning valid data → then investigate SCCM behavior.

---

## Outstanding Items

- The original SSL certificate (`D8A05EE8BF0FE5F28B988CEE024794D6A415D2E2`) that SQL was configured to use is missing from the certificate store. SQL is currently running without SSL encryption on connections. For a production environment this should be resolved by either:
  - Generating a new certificate and binding it to SQL Server via SQL Server Configuration Manager
  - Or confirming HTTP/unencrypted SQL connections are acceptable for the lab environment

- The MOF's `NamespaceSecuritySDDL` still references the old `SCCM` domain SID. This is cosmetic in the current working state but should be noted if SCCM is ever reinstalled or the provider re-registered.
