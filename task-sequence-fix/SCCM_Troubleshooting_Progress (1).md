# SCCM Lab Troubleshooting Progress
**Goal:** Deploy 7-Zip via Software Center from SCCM.lab.local to CLIENT01  
**Date:** 2026-05-29 / 2026-05-30  
**Site Code:** PS1 (Primary Site 1)  
**Environment:** Lab — no proper PKI, self-signed certs, eHTTP mode

---

## Current Status: 🔴 Blocked — Broken SMS Certificate Chain

The root cause is a missing `CN=SMS Issuing` intermediate CA certificate. The DP's SSL cert (`CN=SCCM.lab.local`) was issued by `CN=SMS Issuing`, but that issuing CA cert is missing from all Windows certificate stores on the SCCM server. This makes the cert chain unverifiable on any client.

---

## Root Cause Chain (How We Got Here)

```
eHTTP enabled at site level
    → SCCM generates self-signed SMS PKI hierarchy (Root → Issuing → Leaf)
    → CN=SMS Issuing cert was lost from certificate stores at some point
    → CN=SCCM.lab.local leaf cert remains but chain cannot be built
    → CLIENT01 cannot trust HTTPS connection to DP
    → Content download fails with SSL trust error
    → "Software could not be found on any servers at this time"
```

---

## What Was Tried & Findings

### Phase 1 — Policy & HTTP/HTTPS Switching
| Action | Result |
|---|---|
| Forced machine policy download via TriggerSchedule | ✅ Worked |
| CcmHttpsState changed from 1216 → empty | ✅ Policy landed |
| Deleted SMS cert from CLIENT01 to stop HTTPS forcing | ❌ CcmExec regenerated certs on restart |
| Removed HTTPS 443 binding from Default Web Site | ✅ Client switched to HTTP port 80 |
| Client switched to HTTP but got 405 Method Not Allowed | ❌ WebDAV issue |

### Phase 2 — WebDAV & IIS
| Action | Result |
|---|---|
| Confirmed Web-DAV-Publishing installed | ✅ |
| Enabled WebDAV via IIS Manager | Partial — triggered iisreset |
| iisreset caused HTTPS 443 binding to regenerate | ❌ SCCM auto-recreates 443 binding on iisreset |
| Confirmed all IIS App Pools running | ✅ All Started |
| Confirmed Anonymous Auth enabled on CCMTOKENAUTH vdir | ✅ |

### Phase 3 — Token Auth & Content Location
| Action | Result |
|---|---|
| Checked MP_Token.log | ✅ MP issuing DPAuth tokens (PackageID empty = wildcard, normal) |
| Checked ContentTransferManager.log | ❌ Client always picks HTTPS (Order 0) over HTTP (Order 1) |
| Confirmed SSLState = 63 in MP registry | ❌ This is why client always prefers HTTPS |
| Confirmed site set to "HTTPS or EHTTP" | ❌ Root cause of SSLState 63 |

### Phase 4 — Certificate Trust
| Action | Result |
|---|---|
| Identified DP cert: CN=SCCM.lab.local, issued by CN=SMS Issuing | ✅ |
| Exported CN=SMS, CN=SCCM root cert and imported to CLIENT01 Trusted Root | ✅ Imported successfully |
| Tested HTTPS: still failed with SSL trust error | ❌ Chain broken — SMS Issuing cert missing |
| Cleared stuck DTS jobs (D6F7F0798, 656FA0B3) from WMI | ✅ Cleared via CCM_DTS_JobEx + CCM_CTM_JobStateEx4 classes |
| New download job started — reached "Downloading 0%" | Partial progress |
| Failed with "software could not be found on any servers" | ❌ Location request failing |

### Phase 5 — SMS Issuing CA Recovery Attempts
| Action | Result |
|---|---|
| Searched all cert stores (SMS, My, CA, Root, enterprise) | ❌ CN=SMS Issuing not found anywhere |
| Ran certutil -store SMS | ✅ Confirmed private key container exists: `{022D6A27-1D62-4308-894D-9294227ACC9B}` |
| Built X509Chain on leaf cert | ❌ Chain valid: False — only 1 element returned |
| Checked SCCM DB via sqlcmd | ❌ Connection failed (SSL error on sqlcmd itself) |
| Attempted certreq to regenerate issuing cert using existing key | ❌ "Object already exists" (0x8009000f) |
| Toggled eHTTP off/on via site properties | ❌ Did not regenerate SMS Issuing cert |
| Deleted CN=SCCM.lab.local and restarted SMS_EXECUTIVE | ❌ Regenerated same broken chain |
| Changed site to "HTTPS only" — waiting for HMAN propagation | ⏳ SSLState still 63, HMAN in 3600s wait loop |

---

## Current Environment State

| Component | State |
|---|---|
| Site communication mode | HTTPS only (set in console, not yet propagated) |
| SSLState in registry | 63 (should be 32 for HTTPS-only) |
| HTTPS 443 binding on Default Web Site | Present (SCCM auto-recreates) |
| CN=SMS, CN=SCCM root cert on CLIENT01 | ✅ Imported to Trusted Root |
| CN=SMS Issuing cert | ❌ Missing from all stores |
| CN=SCCM.lab.local leaf cert | ✅ Present in SMS store |
| X509 chain validity | ❌ False — chain broken |
| All IIS App Pools | ✅ Running |
| MP health (mplist/mpcert) | ✅ Responding on HTTP port 80 |
| DTS stuck jobs | ✅ Cleared |
| Boundary group | ✅ CLIENT01 (192.168.10.100) in Lab Range boundary |
| TempShare on SCCM | ✅ Active at C:\temp |

---

## Key Technical Details

- **CLIENT01 IP:** 192.168.10.100
- **Boundary:** 192.168.10.1–192.168.10.254 (Lab Range) ✅
- **DP SSL cert thumbprint:** `094776B2E458E671B05B39DA989B238B07259A4D`
- **SMS Issuing private key container:** `{022D6A27-1D62-4308-894D-9294227ACC9B}`
- **SMS root cert thumbprint on CLIENT01:** `E2E408987D1E422980A11E98B8F99D885106B76C`
- **WMI classes for job cleanup:** `root\ccm\DataTransferService::CCM_DTS_JobEx` and `root\ccm\ContentTransferManager::CCM_CTM_JobStateEx4`
- **SQL instance:** MSSQLSERVER (default instance, server name: SCCM)
- **SCCM DB:** CM_PS1

---

## Next Steps (In Order)

### Option A — Recreate SMS Issuing cert with new key (current attempt)
Generate a new self-signed CA cert named `CN=SMS Issuing`, install in the CA store on both SCCM and CLIENT01, then verify the chain builds correctly.

```powershell
# On SCCM server - generate new issuing cert (new key, not reusing existing container)
$inf = @"
[Version]
Signature="`$Windows NT`$"
[NewRequest]
Subject = "CN=SMS Issuing"
MachineKeySet = TRUE
RequestType = Cert
KeyLength = 2048
KeyUsage = 0x86
[Extensions]
2.5.29.19 = "{text}ca=1&pathlength=0"
"@
$inf | Out-File "C:\temp\smsissuing2.inf" -Encoding ASCII
certreq -new "C:\temp\smsissuing2.inf" "C:\temp\smsissuing2.cer"
certutil -addstore "CA" "C:\temp\smsissuing2.cer"
```

Then on CLIENT01:
```powershell
Copy-Item "\\SCCM\TempShare\smsissuing2.cer" "C:\temp\smsissuing2.cer"
certutil -addstore "CA" "C:\temp\smsissuing2.cer"
certutil -addstore "Root" "C:\temp\smsissuing2.cer"
```

> ⚠️ Note: This new cert won't actually validate the existing leaf cert's signature (wrong key), but may satisfy chain-building enough for the SSL handshake in a lab context.

### Option B — Wait for HMAN to propagate HTTPS-only setting
SSLState should change from 63 → 32 once HMAN processes the site control change (up to 3600s wait). Once it does, restart SMS_EXECUTIVE and check mplist for updated SSLState.

### Option C — Nuclear reset of eHTTP PKI
1. Switch site to "HTTPS or EHTTP" → Apply → wait 5 min
2. Delete ALL certs from `Cert:\LocalMachine\SMS`
3. Restart SMS_EXECUTIVE
4. Switch back to "HTTPS only" → Apply
5. Check if SMS Issuing cert is recreated properly this time

### Option D — Force HTTP-only via scheduled task (quickest lab workaround)
Create a startup scheduled task on SCCM that removes the 443 IIS binding on every boot, preventing SCCM from advertising HTTPS. Combined with forcing CcmHttpsState=0 on CLIENT01, this bypasses the cert issue entirely.

---

## Files on SCCM Server (C:\temp)
- `SCCMRoot.cer` — CN=SMS, CN=SCCM root cert (imported on CLIENT01)
- `SCCMDPCert.cer` — CN=SCCM.lab.local leaf cert (old, ignore)
- `smsissuing.inf` / `smsissuing2.inf` — certreq INF attempts
- `certinfo.txt` — certutil dump of Certificate 6

## TempShare
`\\SCCM\TempShare` → `C:\temp` (FullAccess: Everyone) — use this to transfer files to CLIENT01
