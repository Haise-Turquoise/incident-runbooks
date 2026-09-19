# Chrome 134 rollback log (mid-incident state)

*State document kept during the incident, before the tabs were recovered. The final log is `README.md` in this folder.*

## Objective function

- Main objective: restore Chrome 134, disable GoogleUpdate so it cannot silently upgrade again
- Sub-objective 1: recover browsing history (3 months)
- Sub-objective 2: recover the tabs / tab groups organized before the upgrade
- Sub-objective 3: lock GoogleUpdate so a Windows update cannot trigger a reset

---

## Known facts

- Chrome was silently upgraded from 134 to 147 by the GoogleUpdate scheduled task
- Trigger: Windows automatic restart → scheduled task activated → GoogleUpdate pulled a 12-month backlog of versions in one go
- Turning off auto-update in Chrome's settings only affects the UI prompt, not the GoogleUpdate service itself
- After 147 started it wrote version-format changes into Preferences, Sessions and other files under User Data\Default
- A full User Data backup from 2025-03 exists on the blue external SSD

---

## File path configuration

### Application directory
```
C:\Program Files\Google\Chrome\Application\
├── 147.0.7727.56.bak\        ← 147 version folder (renamed as backup)
├── 134.0.6998.89(chrome)\    ← 134 version folder (copied in; note the suffix on the folder name)
├── chrome.exe                ← currently the 134 launcher (2025/3/7)
├── chrome_proxy.exe
├── chrome.exe.txt
├── initial_preferences
├── PlatformExperienceHelper\
└── SetupMetrics\
```

**Note**: the 134 folder is named `134.0.6998.89(chrome)`; the `(chrome)` suffix makes the launcher unable to find its sub-folder and raises an SxS error. Confirm whether it has been renamed to `134.0.6998.89`.

### User Data directory
```
C:\Users\<user>\AppData\Local\Google\Chrome\
├── User Data\                        ← currently in use (134 running)
│   └── Default\
│       ├── Sessions\
│       │   ├── Sessions.134bak\      ← Sessions backup from the 134 period
│       │   └── Session_13420681320090187   ← pre-upgrade session copied from 147bak (written 2:29 AM, native 134 format)
│       ├── History                   ← current (8.6MB, written 2026/4/15)
│       ├── History.134bak-25-3       ← History from the 2025-03 backup (15.8MB)
│       ├── Preferences               ← current (written by 147, then re-initialized by 134)
│       └── ...
└── User Data.147bak\                 ← full 147 backup
    └── Default\
        ├── Sessions\
        │   ├── Session_13420681320090187   ← 2026/4/15 2:29 (before the Windows restart, written by 134, copied)
        │   ├── Session_13420761707546840   ← 2026/4/15 17:23 (written by 147)
        │   ├── Session_13420761825088655   ← 2026/4/15 17:50 (written by 147)
        │   └── Tabs_13420761547524397      ← 2026/4/15 17:50 (written by 147)
        ├── History                         ← 8.6MB, contains the last 3 months
        ├── Bookmarks                       ← 236KB (updated 2026/4/15)
        └── ...
```

---

## Operations completed

1. Copied the 134 install folder from the blue external SSD into the Application directory
2. Renamed the 147 folder to `147.0.7727.56.bak`
3. Replaced the root `chrome.exe` with the 134 version
4. Renamed `User Data\Default` as a whole to `Default.147bak` (later confirmed to actually be `User Data.147bak`)
5. Replaced the current User Data with the 2025-03 backup from the SSD
6. Replaced `Default\History` with `History.134bak-25-3` (recovers 3 months of history)
7. Renamed the current Sessions to `Sessions.134bak`, created a new Sessions folder
8. Copied only `Session_13420681320090187` (native 134 format, written at 2:29) into the new Sessions folder
9. Chrome 134 starts normally; history recovered

---

## Unresolved

- **Tabs / tab groups not fully recovered**: the organized tab groups are still missing; after copying Session_13420681320090187 the restore prompt did not appear, or the restore was incomplete
- The session files written by 147 (17:23, 17:50) hold more tab state but are in the 147 format; compatibility unknown

---

## To do

- [ ] Use Claude Code to try parsing the Sessions files and extract the list of tab URLs
- [ ] Confirm whether the `(chrome)` suffix has been removed from the `134.0.6998.89(chrome)` folder name
- [ ] Run the four GoogleUpdate-disable commands so it cannot upgrade again:
```powershell
sc config gupdate start= disabled
sc config gupdatem start= disabled
schtasks /Change /TN "GoogleUpdateTaskMachineCore" /DISABLE
schtasks /Change /TN "GoogleUpdateTaskMachineUA" /DISABLE
```
- [ ] Set up a scheduled task that re-locks GoogleUpdate after Windows updates

---

## Environment

- OS: Windows 11
- User name: `<user>`
- Target stable Chrome version: 134.0.6998.89
- Upgraded version: 147.0.7727.56
- External SSD: blue, holds the full 2025-03 User Data backup
