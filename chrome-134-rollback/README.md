# Chrome 134 rollback log

> **Final state (2026-04-15 19:45)**: recovered. Chrome 134 running normally, all 73 tabs (8 windows) restored, GoogleUpdate locked down.

---

## Final fix (reproducible)

If GoogleUpdate silently upgrades Chrome again, follow these steps to recover.

### Preconditions
- Clean 134 backup at `C:\Users\Public\Documents\tmp\134-safe-backup\` (chrome.exe + the 134.0.6998.89 folder + clean User Data)
- Original session file at `User Data.147bak\Default\Sessions\Session_13420681320090187`

### Step 1: lock down every Google update mechanism (elevated terminal, PowerShell)

**All of these must be run; none can be skipped.** Google has several parallel update mechanisms.

```powershell
# --- 1a. Kill the updater executables (rename them) ---
# New-style updater (the one that actually does the work; most important!)
ren "C:\Program Files (x86)\Google\GoogleUpdater\148.0.7730.0\updater.exe" updater.exe.bak
# Legacy updater (may already have been cleaned up by the new one; errors can be ignored)
ren "C:\Program Files (x86)\Google\Update\GoogleUpdate.exe" GoogleUpdate.exe.bak

# --- 1b. Disable all Google services ---
# Note: in PowerShell this must be sc.exe, not sc (that is the Set-Content alias)
sc.exe config GoogleUpdaterService148.0.7730.0 start= disabled
sc.exe config GoogleUpdaterInternalService148.0.7730.0 start= disabled
sc.exe config GoogleChromeElevationService start= disabled
# Legacy services (do not exist on this machine; errors can be ignored)
sc.exe config gupdate start= disabled
sc.exe config gupdatem start= disabled

# --- 1c. Registry policy as a backstop ---
reg add "HKLM\SOFTWARE\Policies\Google\Update" /v UpdateDefault /t REG_DWORD /d 0 /f
```

**Troubleshooting notes:**
- `sc` in PowerShell is the `Set-Content` alias → must be written `sc.exe`
- `gupdate` / `gupdatem` do not exist on this machine; error 1060 is normal
- The scheduled-task folder `\GoogleSystem` is empty; no schtasks command needed
- The GoogleUpdater version number may change; check the actual one with `dir "C:\Program Files (x86)\Google\GoogleUpdater"`

### Step 2: deploy the 134 binaries (elevated terminal)
```powershell
# Back up the current version
ren "C:\Program Files\Google\Chrome\Application\chrome.exe" chrome.exe.XXXbak
ren "C:\Program Files\Google\Chrome\Application\<new version number>" <new version number>.bak

# Deploy 134
copy "C:\Users\Public\Documents\tmp\134-safe-backup\chrome.exe" "C:\Program Files\Google\Chrome\Application\chrome.exe"
xcopy "C:\Users\Public\Documents\tmp\134-safe-backup\134.0.6998.89(chrome)" "C:\Program Files\Google\Chrome\Application\134.0.6998.89\" /E /I /H
```
**Critical**: the target folder name must be `134.0.6998.89` (without the `(chrome)` suffix); otherwise the launcher cannot find its sub-folder and raises an SxS error.

### Step 3: replace User Data (a normal terminal is enough)
```
# Back up the polluted User Data
ren "User Data" "User Data.XXXpolluted-bak"

# Replace with the clean backup
xcopy "C:\Users\Public\Documents\tmp\134-safe-backup\User Data" "User Data\" /E /I /H
```
**Why it cannot be repaired and must be replaced**: a higher Chrome version irreversibly upgrades the SQLite database schemas (Web Data, Cookies, Login Data, etc.) and the version fields in Preferences (`extensions.last_chrome_version` etc.). When 134 detects a version downgrade it crashes and exits. Fixing the Preferences fields one by one is not enough, and SQLite cannot be downgraded by hand.

### Step 4: restore Session + History
```python
# 1. Empty the Sessions directory, put in the 134-format session file
#    Key file: Session_13420681320090187 (2:29AM, natively written by 134, 2.1MB)
# 2. Copy the History file from 147bak (keeps recent history)
# 3. Edit Preferences:
#      profile.exit_type = "Crashed"       ← triggers the restore prompt
#      session.restore_on_startup = 5      ← restore the last session
#    All other version fields must stay at 134 (already the case in the clean backup; no change needed)
```

### Step 5: restore the theme (Turquoise-Green)
```
# Extract the theme extension from the zip or config-backup into the Extensions directory
xcopy "config-backup-0415\theme-extension" "User Data\Default\Extensions\<theme-extension-id>\" /E /I /H

# Edit the theme paths in Preferences (with python or by hand):
#   extensions.theme.id = "<theme-extension-id>"
#   extensions.theme.pack = "C:\Users\<user>\...\Extensions\<theme-extension-id>\1.6_0"
```
**Note**: the user name inside the theme path must be the current Windows user name, not the older user name from the earlier backup.

### Step 6: start Chrome
- A sync prompt appears on startup → **decline / skip** (the state on Google's servers is the higher version; syncing it down would overwrite everything)
- Chrome restores all tabs automatically (0 clicks)
- Hide the bookmarks bar: `Ctrl+Shift+B`

---

## Approaches tried and failed (Attempt Log)

### Attempt 1: restore the session directly on the 147-polluted User Data
- **Operation**: copied Session_13420681320090187 from 147bak into the current Sessions directory, set exit_type=Crashed
- **Result**: Chrome 134 started but the tabs were not restored
- **Cause**: User Data had already been written by 147; fields such as `extensions.last_chrome_version` in Preferences were 147. Chrome 134 could still start (only the session restore failed)
- **Lesson**: session restore needs not only the session file but also a consistent Preferences and configuration state

### Attempt 2: fix the version fields in Preferences, then restore the session
- **Operation**: changed every 147 reference in Preferences and Local State back to 134, kept exit_type=Crashed
- **Result**: Chrome 134 started and **successfully restored all 73 tabs**! But within 1 minute GoogleUpdate upgraded it again, to 147.0.7727.102
- **Cause**: forgot to lock GoogleUpdate first
- **Lesson**: **GoogleUpdate must be locked before any restore operation**

### Attempt 3: lock GoogleUpdate, redeploy 134 + fix Preferences
- **Operation**: disable GoogleUpdate → deploy the 134 binaries → fix the Preferences version fields → session restore
- **Result**: Chrome 134 "crashed on sign-in" and could not run normally
- **Cause**: the second run of 147.0.7727.102 had deeply polluted User Data (irreversible SQLite schema upgrade); fixing the Preferences JSON fields alone was not enough
- **Lesson**: **User Data polluted by a higher version cannot be repaired by editing fields; it must be replaced as a whole with a clean backup**

### Attempt 4 (final, successful): replace User Data as a whole with the clean copy
- **Operation**: lock GoogleUpdate → deploy the 134 binaries → replace User Data as a whole with the clean safe-backup copy → put in the session file + copy History → set exit_type=Crashed
- **Result**: Chrome 134 starts normally, all 73 tabs restored automatically, no manual clicks needed

---

## Version-stamp investigation (verified 2026-04-15)

Confirmed **no intermediate version was swapped in between 134 and 147**. Version timeline:
1. **133.0.6943.127**: the version the profile was created with (2025-02-21); also the shortcut_migration version
2. **134.0.6998.89**: built 2025-03-07, `stats_version=134.0.6998.89-64`, ran stably until 2026-04-15
3. **147.0.7727.56**: silently upgraded on the afternoon of 2026-04-15, later pulled up to 147.0.7727.102

Key evidence: `stats_version` in the Local State of 147bak still read `134.0.6998.89-64`; any upgrade in between would have overwritten this value.

---

## Current file layout (final state, 2026-04-15)

### Application directory
```
C:\Program Files\Google\Chrome\Application\
├── 134.0.6998.89\              ← in use (copied from safe-backup, (chrome) suffix removed)
├── 147.0.7727.102.bak\         ← backup of the second 147 upgrade
├── 147.0.7727.56.bak\          ← backup of the first 147 upgrade
├── chrome.exe                  ← 134 version (3.38MB, 2025/3/7)
├── chrome.exe.147-102bak       ← backup of the 147.102 chrome.exe
├── chrome.exe.txt              ← another 147 chrome.exe backup
├── chrome_proxy.exe            ← still the 147 version (does not affect function)
└── ...
```

### User Data directory
```
C:\Users\<user>\AppData\Local\Google\Chrome\
├── User Data\                          ← in use (from the clean 134 safe-backup)
│   └── Default\
│       ├── Sessions\
│       │   └── Session_13420681320090187   ← 2:29AM 134-format session (restored successfully)
│       ├── History                         ← copied from 147bak (keeps 3 months of history)
│       └── Preferences                    ← clean 134 state
├── User Data.147bak\                   ← full backup after the first 147 upgrade
├── User Data.147polluted-bak\          ← User Data polluted twice by 147
└── session_recovery_73tabs.html        ← HTML fallback (8 windows, 73 tab URLs)
```

### Google update mechanism lockdown status (5 layers)
```
1. Executables (renamed):
   C:\Program Files (x86)\Google\GoogleUpdater\148.0.7730.0\updater.exe.bak  ← new-style updater (main threat)
   C:\Program Files (x86)\Google\Update\GoogleUpdate.exe.bak                 ← legacy (already cleaned up by the new one; directory empty)

2. Services (DISABLED):
   GoogleUpdaterService148.0.7730.0         ← DISABLED
   GoogleUpdaterInternalService148.0.7730.0 ← DISABLED
   GoogleChromeElevationService             ← DISABLED (its binary path points into the renamed 147.bak directory: doubly dead)

3. Registry policy:
   HKLM\SOFTWARE\Policies\Google\Update\UpdateDefault = 0

4. Scheduled tasks:
   \GoogleSystem folder exists but is empty

5. Legacy services gupdate/gupdatem: not installed on this machine
```

### Clean backup (do not delete)
```
C:\Users\Public\Documents\tmp\134-safe-backup\
├── chrome.exe                              ← 134 launcher (3.38MB, 2025/3/7)
├── 134.0.6998.89(chrome)\                  ← full 134 files (remove the suffix when deploying)
├── User Data\                              ← clean 134 User Data (never touched by 147)
├── Session_13420681320090187.original      ← the original 2:29AM session (gold standard)
├── session_recovery_73tabs.html            ← HTML fallback (8 windows, 73 tab URLs)
├── Sessions-running-bak-0415\              ← session files backed up while running
│   ├── Session_13420681320090187
│   ├── Session_13420770485155286
│   ├── Session_13420770488091709
│   └── Tabs_13420770221180802
├── config-backup-0415\                     ← configuration snapshot (Preferences + theme + bookmarks + LocalState)
│   ├── Preferences                         ← clean 134 + correct theme path + restore settings
│   ├── LocalState
│   ├── Bookmarks
│   └── theme-extension\                    ← Turquoise-Green theme
└── chrome134-config-backup-20260415.zip    ← zip of the above config + session + HTML (12MB)
```
**Why it lives in `C:\Users\Public\Documents\tmp\`**: this directory is outside Chrome's User Data path and is not touched by Chrome's upgrade / reset / clean-up mechanisms. GoogleUpdater only operates on files under `Program Files` and `AppData\Local\Google`.

---

## To do (not urgent)

- [ ] Restore Bookmarks from 147bak (147bak has 236KB vs 232KB in the current safe-backup)
- [ ] Fix UI theme / appearance settings (currently from the 2025-03 backup, somewhat old)
- [ ] Consider whether to restore Login Data, Cookies, etc. from 147bak (possible SQLite compatibility problems)
- [ ] Set up a mechanism that checks whether GoogleUpdate has come back after Windows updates

---

## Environment

- OS: Windows 11 (10.0.26200)
- User name: `<user>`
- Target stable Chrome version: 134.0.6998.89
- Upgraded versions: 147.0.7727.56 → 147.0.7727.102
- Python: 3.12.10
- GoogleUpdate path: `C:\Program Files (x86)\Google\Update\` (note: x86)
- Clean backup: `C:\Users\Public\Documents\tmp\134-safe-backup\`
