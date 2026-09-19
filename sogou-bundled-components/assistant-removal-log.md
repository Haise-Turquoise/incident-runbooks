# Blocking bundled adware: the Wangzai AI assistant — handling log

> **Final state (2026-05-11)**: the three located SGSmartAssistant / SOGOUSmartAssistant executables have been replaced with empty files and set read-only. The fourth, the actual running path (located through Task Manager; the operation completed but the path was not recorded), is pending verification. The input method's core function and its online word suggestions are kept; the Wangzai AI process can no longer start normally.

---

## Core architecture analysis

### Process topology of Wangzai AI

Wangzai AI ("AI汪仔") is **an independent child process outside the input method's main process**, not an embedded module. The relationship:

```
Sogou input method main process (SogouPY.exe etc.)
    └── launches SGSmartAssistant.exe (independent Wangzai AI process)
             └── data / cache directory: C:\Users\<user>\AppData\Roaming\ichat\
```

Key properties:
- The process shows in Task Manager as **"AI汪仔"** or **"搜狗输入法 AI汪仔"**
- Actual exe name: `SGSmartAssistant.exe` (15.x series) / `SOGOUSmartAssistant.exe` (IChat component)
- The data directory (`ichat\`) has **a completely different name** from the exe directory; this is a deliberate design that breaks the name → path mapping
- The install directory is a non-standard path (under Downloads rather than Program Files), which evades the usual software-path scans

### Multiple launch paths (why it pops up again after being switched off in settings)

Sogou Wangzai has at least the following parallel launch paths; the **auto-start switch** in settings cuts only one of them:

1. The input method's main process launches it directly (the main one)
2. Registry Run key auto-start
3. Forced configuration reset on version upgrade (**upgrade → config reset → Wangzai comes back**; this is the most frequent revival path)
4. Hotkey trigger (`=` key / `Alt+Space`)

A version upgrade **not only resets the configuration, it may also drop a new exe over the blank replacement file**; this is why read-only protection is needed.

---

## Final handling (reproducible)

### Preconditions
- The exe paths have been located (Task Manager "Details" tab + findstr)
- Replacement: replace the exe with an empty txt file of the same name (empty content), keeping the `.exe` extension
- Read-only protection: right-click → Properties → tick **Read-only**

### Step 1: locate every Wangzai exe (CMD, elevated terminal)

**Standard locating command** (for later re-checks or when a new version appears):

```cmd
dir /s /b C:\Users\<user>\ 2>nul | findstr /i "SmartAssistant.exe"
```

**Notes**:
- The user name must be typed exactly; on the first run it was off by one letter, which scanned a non-existent directory and returned nothing
- Scan from the user's root directory so that AppData and Downloads are both covered
- The Wangzai process-name prefix is `SG` (15.x main version) or `SOGOU` (IChat sub-component)

**Actual findstr output on this machine**:

```
C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Recent\SGSmartAssistant.exe.txt.lnk
C:\Users\<user>\Downloads\sogou input\SogouInput\15.6.0.2047\SGSmartAssistant.exe
C:\Users\<user>\Downloads\sogou input\SogouInput\15.7.0.2170\SGSmartAssistant.exe
C:\Users\<user>\Downloads\sogou input\SogouInput\Components\IChat\1.0.2.2909\SOGOUSmartAssistant.exe
```

**The `.lnk` entry in the Recent directory** (`SGSmartAssistant.exe.txt.lnk`) shows that the fourth, actually running exe was already replaced during the Task Manager operation; Windows kept a shortcut record of the access, but the lnk itself needs no handling.

### Step 2: replace with empty files + read-only protection

**The three paths already handled (read-only set)**:

```
C:\Users\<user>\Downloads\sogou input\SogouInput\15.6.0.2047\SGSmartAssistant.exe

C:\Users\<user>\Downloads\sogou input\SogouInput\15.7.0.2170\SGSmartAssistant.exe

C:\Users\<user>\Downloads\sogou input\SogouInput\Components\IChat\1.0.2.2909\SOGOUSmartAssistant.exe
```

**The fourth path, pending verification**: located through the Task Manager "Details" tab and already replaced, but the absolute path was not recorded during the operation. How to verify:

```cmd
dir /s /b C:\Users\<user>\ 2>nul | findstr /i "SmartAssistant"
```

If the fourth path is already an empty read-only file, findstr will still find it (size 0 bytes). If the file has been overwritten by a new version, repeat the replace + read-only operation.

### Step 3: hardening the read-only protection (optional; guards against an administrator-level overwrite)

If there is concern that the updater writes with administrator rights (bypassing read-only), tighten the file permissions further:

```
Right-click the exe → Properties → Security → Edit →
find the current user → untick "Write"
keep Full control only for Administrators (but if you are yourself an Administrator, do not leave write enabled)
```

From the sample of reports on Zhihu, the Sogou updater rarely performs a forced administrator-level overwrite; read-only protection is generally enough.

---

## Key path of the locating process (working notes)

### Wrong paths ruled out

| Initial guess | Reality | Conclusion |
|---|---|---|
| `C:\Users\<user>\AppData\Local\SogouInput\16.3\` | Does not exist | 16.3 was a remembered version number; there is no SogouInput directory under AppData\Local |
| `C:\Users\<user>\AppData\Roaming\ichat\` | Exists, but only as a data directory | ichat is the runtime cache / config directory; the exe is not there |
| `C:\Program Files (x86)\SogouInput\` | Does not exist | Sogou's install path on this machine is non-standard; it was never installed into Program Files |

### The path that worked

1. Task Manager "Processes" tab (search `sogou`) → right-click the collapsed parent node → **"Open file location" is greyed out** (because the right-click was on the parent)
2. Switch to the Task Manager **"Details"** tab → search `sogou` or `assistant` → find `SGSmartAssistant.exe` → right-click → "Open file location" → **located**
3. Supplementary scan: `dir /s /b C:\Users\<user>\ 2>nul | findstr /i "SmartAssistant.exe"` → locates the remaining three versions

### The user-name typo during locating

The first findstr run had the user name in the path off by one letter, so it scanned a non-existent directory and returned nothing. Once noticed and corrected, it hit.

---

## Key directory layout of Wangzai AI

### exe locations (unpacked installer tree, non-standard path)

```
C:\Users\<user>\Downloads\sogou input\SogouInput\
├── 15.6.0.2047\
│   └── SGSmartAssistant.exe          ← replaced with empty read-only file
├── 15.7.0.2170\
│   └── SGSmartAssistant.exe          ← replaced with empty read-only file
└── Components\
    └── IChat\
        └── 1.0.2.2909\
            └── SOGOUSmartAssistant.exe  ← replaced with empty read-only file
```

**Note**: the unpacked installer tree sits under Downloads rather than Program Files; this is a non-standard layout Sogou uses deliberately to evade the usual software-path scans. The version number `1.0.2.2909` (IChat sub-component) ≠ the input method's main version (15.x); two version-number schemes run in parallel.

### Runtime data directory (no exe)

```
C:\Users\<user>\AppData\Roaming\ichat\
    └── CefLocalStorage\
        └── Code Cache\js\   ← browser rendering engine cache (Wangzai renders its UI with a CEF core)
```

### Windows Recent trace (no handling needed)

```
C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Recent\
    └── SGSmartAssistant.exe.txt.lnk  ← shortcut left by the operation; just an access trace
```

---

## Attempt Log

### Attempt 1: switch off auto-start in the Sogou settings
- **Operation**: S icon → gear / settings → More settings → turn off "Auto-start AI Wangzai"; in Advanced settings turn off skin recommendations / bubble recommendations / desktop bottom-right recommendations
- **Result**: effective in the short term; came back automatically after a version upgrade
- **Cause**: the upgrade resets the configuration, and the settings layer cuts only one launch path
- **Conclusion**: switching it off in settings is a temporary measure; it does not persist

### Attempt 2: locate the exe from the Task Manager "Processes" tab
- **Operation**: search sogou → right-click the "搜狗输入法 AI汪仔" parent node → "Open file location"
- **Result**: the option was greyed out
- **Cause**: the right-click was on the collapsed parent node, not the actual process row; Windows does not allow file location on a parent node
- **Conclusion**: switch to the Task Manager "Details" tab and act on the specific process row

### Attempt 3: findstr with the version number 1.0.3 across AppData
- **Operation**: `dir /s /b C:\Users\<user>\AppData 2>nul | findstr "1.0.3"`
- **Result**: the first run produced nothing (user name mistyped); after correcting it, the second run produced a flood of unrelated Chrome / Office cache files and no Sogou hit
- **Cause**: the version number 1.0.3 was not precise enough (it is actually 1.0.2), and Chrome Code Cache file names contain many random hex strings that match findstr by accident
- **Conclusion**: a version number is too low-signal as a findstr keyword; the exe name is more accurate

### Attempt 4 (final, successful): findstr on the exe name across the whole user directory
- **Operation**: `dir /s /b C:\Users\<user>\ 2>nul | findstr /i "SmartAssistant.exe"`
- **Result**: exactly four hits (the Recent lnk trace + three actual exes)
- **Conclusion**: the exe name as the keyword, combined with a full scan from the user's root directory, is the lowest-friction locating path

---

## Characteristics of the adware, from an engineering viewpoint

Design features observed during the handling. These are not engineering oversights; they are deliberate layers that raise the cost of removal by the user:

1. **The settings switch cuts only some launch paths**: multiple parallel paths, a single switch is ineffective
2. **The collapsed Task Manager node blocks direct locating**: collapsing the parent greys out "Open file location"; the most intuitive path is closed
3. **Process name and data directory name do not match** (`SGSmartAssistant` vs `ichat`): breaks the direct name → path mapping
4. **Non-standard install path** (Downloads rather than Program Files): evades the usual software-path scanning habits
5. **Multiple versions coexist**: dilutes the target; a single clean-up is incomplete
6. **Version upgrades force a configuration reset**: overwrites what the user has switched off, and may drop a new exe over the blank replacement file
7. **The official help centre hides the disable instructions**: the method exists (visible on the uninstall retention page) but cannot be reached from the normal help entry

---

## Current protection status

```
Layer 1: exe replaced with empty files (done):
  C:\Users\<user>\Downloads\sogou input\SogouInput\15.6.0.2047\SGSmartAssistant.exe    ← empty, read-only
  C:\Users\<user>\Downloads\sogou input\SogouInput\15.7.0.2170\SGSmartAssistant.exe    ← empty, read-only
  C:\Users\<user>\Downloads\sogou input\SogouInput\Components\IChat\1.0.2.2909\SOGOUSmartAssistant.exe  ← empty, read-only
  Fourth (the actual running path located via Task Manager)                             ← replaced, path not recorded

Layer 2: read-only attribute (done):
  The three recorded paths above are all read-only
  Read-only status of the fourth path pending verification

Layer 3: network permission (kept):
  The input method's core online functions are kept; cloud word suggestions unaffected
  No outbound firewall rules or hosts-file domain blocking were applied in this session
```

---

## To do (not urgent)

- [ ] Verify the read-only status of the fourth exe path (findstr scan; check whether it is still 0 bytes)
- [ ] After the next Sogou version upgrade, re-run the findstr scan to check whether a new exe has been dropped over the empty files
- [ ] If a new-version exe appears, repeat the replace + read-only operation for the new path
- [ ] Optional: set every exe under `C:\Users\<user>\Downloads\sogou input\SogouInput\` read-only in one batch, to cover future version sub-directories

---

## Environment

- OS: Windows 11 (10.0.26200.8037)
- User name: `<user>`
- Sogou input method versions: 15.6 / 15.7 (multiple versions coexist)
- Wangzai AI component version: IChat 1.0.2.2909
- exe naming: main version uses `SGSmartAssistant.exe`, IChat sub-component uses `SOGOUSmartAssistant.exe`
- Runtime data directory: `C:\Users\<user>\AppData\Roaming\ichat\`
- Unpacked installer root: `C:\Users\<user>\Downloads\sogou input\SogouInput\`
