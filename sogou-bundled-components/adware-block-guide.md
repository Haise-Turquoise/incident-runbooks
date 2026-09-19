# Sogou input method: blocking the advertising components

> **Final state (updated 2026-04-17)**: outbound firewall blocking is in effect, but a "Change wallpaper" entry still lingers in the Win11 right-click menu. Cause: biz_shellext64.dll is injected directly into explorer.exe through COM registration and does not use the network, so the firewall has no effect on it. Fix pending.

---

## Quick fix (if Wangzai / the wallpaper package / a new component shows up again)

### 1. Check whether the firewall rules are still there (elevated PowerShell)
```powershell
netsh advfirewall firewall show rule name=all dir=out | findstr "Block Sogou"
netsh advfirewall firewall show rule name=all dir=out | findstr "Block kwallpaper"
```

### 2. If the rules were wiped, redeploy (elevated PowerShell, paste in one go)
```powershell
$sogou = "C:\Users\<user>\Downloads\sogou input\SogouInput"
$kwall = "C:\Users\<user>\AppData\Local\kwallpaper_sogou"
netsh advfirewall firewall add rule name="Block Sogou SogouSvc" dir=out action=block program="$sogou\SogouExe\SogouSvc.exe"
netsh advfirewall firewall add rule name="Block Sogou SogouExe" dir=out action=block program="$sogou\SogouExe\SogouExe.exe"
netsh advfirewall firewall add rule name="Block Sogou ComMgr" dir=out action=block program="$sogou\Components\SogouComMgr.exe"
netsh advfirewall firewall add rule name="Block Sogou SGDownload" dir=out action=block program="$sogou\15.7.0.2170\SGDownload.exe"
netsh advfirewall firewall add rule name="Block Sogou SmartAssistant" dir=out action=block program="$sogou\15.7.0.2170\SGSmartAssistant.exe"
netsh advfirewall firewall add rule name="Block Sogou Wangzai" dir=out action=block program="$sogou\15.7.0.2170\SGWangzai.exe"
netsh advfirewall firewall add rule name="Block Sogou BizLauncher" dir=out action=block program="$sogou\15.7.0.2170\SGBizLauncher.exe"
netsh advfirewall firewall add rule name="Block Sogou PinyinUp" dir=out action=block program="$sogou\15.7.0.2170\PinyinUp.exe"
netsh advfirewall firewall add rule name="Block Sogou NetSchedule" dir=out action=block program="$sogou\15.7.0.2170\userNetSchedule.exe"
netsh advfirewall firewall add rule name="Block Sogou Feedback" dir=out action=block program="$sogou\15.7.0.2170\sgfeedbackhelper.exe"
netsh advfirewall firewall add rule name="Block Sogou Guide" dir=out action=block program="$sogou\15.7.0.2170\SGIGuideHelper.exe"
netsh advfirewall firewall add rule name="Block Sogou CrashRpt" dir=out action=block program="$sogou\Components\crashrpt.exe"
Get-ChildItem "$kwall\*.exe" | ForEach-Object { netsh advfirewall firewall add rule name="Block kwallpaper $($_.BaseName)" dir=out action=block program="$($_.FullName)" }
```

### 3. If everything needs to be rolled back
```powershell
netsh advfirewall firewall show rule name=all dir=out | findstr "Block Sogou Block kwallpaper"
# After confirming, delete:
netsh advfirewall firewall delete rule name=all dir=out program="$sogou\SogouExe\SogouSvc.exe"
# ... or delete one by one by name
```

---

## How it works

### Classification of Sogou processes

```
Keep (cloud candidates = the second candidate with the cloud icon while typing):
  SogouCloud.exe          ← cloud candidates; sends pinyin to the server and returns smart suggestions
  SogouImeBroker.exe      ← broker for the input method's main process

Blocked (ads / downloads / component management):
  SogouSvc.exe            ← background service; checks for and downloads new components (whack-a-mole hole #1)
  SogouExe.exe            ← Sogou main controller (hole #2)
  SogouComMgr.exe         ← Component Manager; installs new plug-ins (hole #3)
  SGDownload.exe          ← downloader
  SGBizLauncher.exe       ← commercial content launcher
  PinyinUp.exe            ← pinyin update (triggers version upgrades)
  SGSmartAssistant.exe    ← Wangzai itself
  SGWangzai.exe           ← another Wangzai entry point
  userNetSchedule.exe     ← network scheduled tasks
  sgfeedbackhelper.exe    ← feedback reporting
  SGIGuideHelper.exe      ← guide pop-ups
  crashrpt.exe            ← crash reporting
  kwallpaper*.exe (17)    ← the whole Sogou wallpaper family
```

### Why block outbound traffic instead of deleting files

- Delete / rename the file → SogouSvc detects it is missing and re-downloads it (whack-a-mole)
- Block outbound network → the downloader cannot reach the network, cannot pull new components, and cannot self-repair
- Local typing does not depend on the network, so blocking has no effect on it

### Sogou's three-layer self-recovery mechanism

1. **sogouService / SogouSvc.exe**: resident service, periodically checks component integrity, re-downloads anything deleted
2. **ComponentConfig.ini (encrypted)**: the server pushes a "which components to install" configuration; it cannot be read or modified locally
3. **Version updates**: a silent Sogou update re-enables every component by default

The firewall rules cut the network channel of all three layers at once.

---

## File paths

```
Sogou input method main directory:
  C:\Users\<user>\Downloads\sogou input\SogouInput\
  ├── 15.6.0.2047\          ← old version
  ├── 15.7.0.2170\          ← current version (the exes are here)
  ├── Components\            ← plug-in directory (IChat = Wangzai, biz_center = ads, etc.)
  └── SogouExe\              ← service + main controller

Sogou wallpaper (separate directory):
  C:\Users\<user>\AppData\Local\kwallpaper_sogou\   ← 17 exes
```

---

## Privacy assessment

- Classification: adware; Malwarebytes detection signature Adware.Sogou
- Citizen Lab (2023) found a vulnerability in the keystroke encryption implementation (an attacker on the same Wi-Fi could intercept input)
- Practical risk: Sogou collects aggregated statistics (hot words, word frequency); it does not inspect individual inputs
- Usage advice: switch to the system input method for passwords and other sensitive input; keep using Sogou for everyday typing

---

## 2026-04-17 investigation: the "Change wallpaper" entry lingering in the Win11 right-click menu

### Symptom

- Win11 desktop right-click → the simplified menu shows a **Change wallpaper** item
- Click "Show more options" to expand the classic menu → the item disappears
- Confirmed: all outbound firewall blocks in place, kwallpaper.exe replaced with a 0KB read-only empty file
- The puzzle: the holes are plugged, so why is the menu still there

### Root cause

**This is not a network attack surface; it is a local COM DLL injection attack surface. The firewall blocked a different, independent "hole".**

```
Attack surface #1 (closed): network download channel
  exe outbound → firewall → ❌ blocked
  exe file → replaced with empty txt → ❌ cannot execute
  ✅ Wangzai / the wallpaper package cannot be downloaded back

Attack surface #2 (not handled): local Shell Extension registration
  explorer.exe starts
    → reads the registry key shellex\ContextMenuHandlers\sgshellext2
    → loads biz_shellext64.dll into the explorer process
    → on right-click the DLL draws the menu item directly
    → entirely local, no network; the firewall cannot see it at all
```

**Why the simplified menu shows it and the expanded menu does not:**
- Win11 simplified menu: "draw first, ask later": the DLL says draw, it draws
- Classic menu: "ask first, draw later": the DLL tries to pull ad content to fill the menu, the firewall blocks it, it returns empty, so the classic menu shows nothing

### How it was located

```
1. reg query shellex\ContextMenuHandlers → found "    sgshellext2" (note the leading spaces: deliberately hidden)
2. Read that key's value → CLSID {7BCE96FA-77AF-4288-9E16-2388A50EC807}
3. Look up the CLSID's InprocServer32 → points to biz_shellext64.dll / biz_shellext.dll
4. Confirm the DLL path: ...\Components\biz_center\1.0.0.3115\
5. biz_center = business center = Sogou's ad distribution centre
```

### Files and registry keys involved

```
Registry (3 places):
  HKLM\SOFTWARE\Classes\Directory\Background\shellex\ContextMenuHandlers\    sgshellext2
    → default value = {7BCE96FA-77AF-4288-9E16-2388A50EC807}
  HKCR\CLSID\{7BCE96FA-77AF-4288-9E16-2388A50EC807}\InprocServer32
    → biz_shellext64.dll (64-bit)
  HKCR\WOW6432Node\CLSID\{7BCE96FA-77AF-4288-9E16-2388A50EC807}\InprocServer32
    → biz_shellext.dll (32-bit)

DLL files (2):
  C:\Users\<user>\Downloads\sogou input\SogouInput\Components\biz_center\1.0.0.3115\biz_shellext64.dll  (3.2MB)
  C:\Users\<user>\Downloads\sogou input\SogouInput\Components\biz_center\1.0.0.3115\biz_shellext.dll   (2.6MB)

The biz_center directory also contains (potentially ad-related):
  biz_helper.exe (7.0MB), biz_notify.exe (1.8MB), biz_render.exe (235KB)
  biz_bundle.dll, browser_host.dll, ginkgo.exe
  pdf-related files (pdftoolsdk*.dll): probably a PDF-conversion promotion
  qimei.dll: Tencent device-fingerprint SDK (introduced after Tencent acquired Sogou)
  4 version directories in total: 1.0.0.2869 / 1.0.0.2880 / 1.0.0.2922 / 1.0.0.3115 (currently active)
```

### Review of strategies tried

| Strategy | Against network download | Against the right-click menu | Conclusion |
|------|-----------|-----------|------|
| Firewall: block exe outbound | ✅ effective | ❌ no effect (the DLL lives inside explorer, not a separate process) | necessary but not sufficient |
| Rename exe to empty txt + read-only | ✅ effective | ❌ no effect (a DLL is not an exe) | necessary but not sufficient |
| Delete registry keys + rename DLL | not relevant | ✅ should work | pending |

### Pending fix (≤5 steps, elevated PowerShell, paste in one go)

```powershell
# Step 1-3: delete the registry keys (right-click menu entry + COM registration)
reg delete "HKLM\SOFTWARE\Classes\Directory\Background\shellex\ContextMenuHandlers\    sgshellext2" /f
reg delete "HKCR\CLSID\{7BCE96FA-77AF-4288-9E16-2388A50EC807}" /f
reg delete "HKCR\WOW6432Node\CLSID\{7BCE96FA-77AF-4288-9E16-2388A50EC807}" /f

# Step 4: rename the DLLs so they cannot be re-registered
ren "C:\Users\<user>\Downloads\sogou input\SogouInput\Components\biz_center\1.0.0.3115\biz_shellext64.dll" biz_shellext64.dll.bak
ren "C:\Users\<user>\Downloads\sogou input\SogouInput\Components\biz_center\1.0.0.3115\biz_shellext.dll" biz_shellext.dll.bak

# Step 5: restart Explorer to apply (or sign out and back in)
taskkill /f /im explorer.exe & start explorer.exe
```

### Extra suggestion to reduce recurrence within a week

If a Sogou component update bumps biz_center to a new version number (e.g. 1.0.0.3200), it may re-register the shell extension.
As a precaution the whole biz_center directory can be set to deny writes:
```powershell
icacls "C:\Users\<user>\Downloads\sogou input\SogouInput\Components\biz_center" /deny <user>:(W) /T
```
Then even if SogouSvc tries to update that directory it is stopped by NTFS permissions.
To revert the permission:
```powershell
icacls "C:\Users\<user>\Downloads\sogou input\SogouInput\Components\biz_center" /remove:d <user> /T
```
