# Incident logs: two Windows-level failures, April to May 2026

Four working documents plus the prompt log behind them, from two incidents on my own machine, translated from the Chinese originals section by section. Nothing was rewritten for presentation: the attempt logs, the wrong guesses, the pending items and the file-tree snapshots are as they were at the time. The Chinese originals are kept locally and available on request; the only edits are the removal of the account name and a theme-extension id.

## The four files

| File | Date | What it is |
|---|---|---|
| [`archive/chrome-134-mid-incident-state.md`](archive/chrome-134-mid-incident-state.md) | 2026-04-15, ~T+3h | State document kept while the incident was open: objective function, known facts, operations done, unresolved, to-do |
| [`chrome-134-rollback/README.md`](chrome-134-rollback/README.md) | 2026-04-15, 19:45 | Final log: reproducible 6-step fix, attempt log (4), version-stamp verification, final layout, 5-layer update lockdown, clean-backup inventory |
| [`sogou-bundled-components/adware-block-guide.md`](sogou-bundled-components/adware-block-guide.md) | 2026-04-17 | Outbound firewall block of 29 executables with the dictionary sync kept alive, the reasoning for blocking instead of deleting, and the root cause of a context-menu entry the firewall could not explain |
| [`sogou-bundled-components/assistant-removal-log.md`](sogou-bundled-components/assistant-removal-log.md) | 2026-05-11 | Locating and neutralizing the bundled AI assistant: wrong paths ruled out, attempt log (4), the adware's design features seen from the engineering side |
| [`prompt-log.md`](prompt-log.md) | 04-15, 04-17, 05-11 | My own inputs to Claude Code during the three sessions, in order, translated; omissions marked. The one file here the assistant did not draft. |

The two Sogou documents were written in separate sessions three weeks apart. The later one's "network permission" section says no firewall rules were applied; the earlier guide shows they were. The 05-11 log was reconstructed in a later session before the firewall state was re-checked (see the 05:57 and 06:00 prompts in the prompt log); the 04-17 guide is the authoritative record of the rules. Both documents are left exactly as written. The state was re-checked on the machine on 2026-09-19, before publication: the 31 outbound rules are in place, the context-menu shell-extension key that the 04-17 guide lists as pending is gone, and the three assistant binaries are still 0-byte files.

## What happened, in one line each

- **Chrome.** A Windows restart woke Google's updater, which jumped Chrome from 134 to 147 and rewrote the profile. Eight windows of organized tabs were gone. Recovered all 73 tabs; the fix that finally held was replacing the whole profile with a clean copy after locking five update mechanisms.
- **Sogou.** An input method kept re-installing an AI assistant, a wallpaper package and a right-click menu entry after every update. Cut the download channel at the firewall, neutralized the binaries, then found that the menu entry came from a COM shell extension loaded into `explorer.exe`, which no firewall rule can see.

Time to close, from the prompt timestamps in `prompt-log.md`: the Chrome profile was recovered 69 minutes after the first prompt, on the fourth attempt, with the rest of the evening spent on theme, backups and the write-up; the context-menu root cause on 04-17 took 15 minutes from symptom to registry key; the frozen-system investigation on 05-11 reached its cause in an hour. None of the three procedures was copied from a guide; each was derived from probes on the machine, with web searches used only for background on the vendors' update mechanisms.

Durability, checked 2026-09-19, five months after the Chrome incident and four after the last Sogou entry: Chrome is still on 134.0.6998.89, the three updater services are still disabled and the updater executable still renamed, the firewall rules are still present, the shell-extension key has not returned, and none of the neutralized binaries has been restored. Nothing has needed a second visit.

## How the work was divided

I did the locating, chose the probe points, wrote the queries, and set the risk and quality controls: what had to survive, what could be sacrificed, what to check before and after each change, when to stop and back up. Claude Code executed commands, refined the plans into runnable steps, and drafted the write-ups from the session. Concretely, from the logs:

| Mine | Claude Code's |
|---|---|
| Objective function with priorities (restore 134 → history → tab groups → lock the updater) written before any command ran | Turning the six steps into pasteable command blocks with the failure notes (`sc` vs `sc.exe`, error 1060, the `(chrome)` suffix) |
| The probes: `stats_version` in `Local State` to prove no intermediate build; `reg query` on `ContextMenuHandlers` when the firewall did not explain the menu; `findstr` on the exe name after the version-number scan produced noise | Walking CLSID → `InprocServer32` → DLL path and listing the rest of the `biz_center` directory |
| Constraints: keep `SogouCloud.exe` and `PinyinUp.exe` online, block everything else; decline cloud sync on first launch; replace the profile rather than patch it after attempt 3 | Enumerating the 29 executables and generating the rule set with its rollback |
| Stop-and-back-up points: the HTML fallback of 73 URLs before touching session files; `.bak` renames instead of deletes; empty read-only files instead of uninstalling | Drafting the `icacls` deny-write prevention and its reversal |
| Reading the greyed-out Task Manager option as "wrong node", and catching the one-letter typo that made a scan return nothing | Keeping the attempt log and the file-tree snapshots current during the session |

Each row can be checked against the documents and against `prompt-log.md`, where every prompt carries a function tag: 37 of 68 set a constraint, ask for a check, or put forward a hypothesis; 6 say go; the longer prompts carry a median of 3 separately answerable items each. Ten prompts from that log, with what each one changed:

| When | What I typed (short) | What it changed |
|---|---|---|
| 04-15 18:40 | "Some files may have been silently updated long ago; check the version stamps in the bak folders. Discuss first, do not execute." | Set the first probe before any fix: the `stats_version` check that later proved no intermediate build had run |
| 18:51 | "Anything renamed or replaced: keep a bak first and record which bak maps to what." | The `.bak`-instead-of-delete rule that runs through every step of the runbook |
| 19:14 | "Do I need to reopen once before locking the updater, or in what order?" | Surfaced the ordering problem right after attempt 2 was undone; became Step 1 |
| 19:25 | "Check that the safe-backup timestamps are from before 2026; if so, they are clean." | Established the clean copy by evidence before it was used for the whole-profile replacement |
| 19:49 | "Keep the attempted approaches and the final fix in separate sections." | The attempt-log / runbook split that the four documents share |
| 19:51 | "Check that everything lockable is locked; estimate recurrence within a week; is the md enough to fix it next time?" | Turned the fix into a verified state and a replayable procedure |
| 23:35 | "Explain in 10 to 20 lines how you found the source. And verify that the second candidate with the cloud icon really is the cloud-dictionary process." | Forced the keep-list to be checked before 29 executables were blocked |
| 23:38 | "If I block these outbound, can the input method break? Is unblocking cheap?" | Reversibility checked before the change; the rollback block in the guide |
| 04-17 01:01 | "All outbound is blocked and the menu entry still comes back. Is this a local modification?" | Pointed the investigation away from the network and at the COM shell extension |
| 05-11 06:00 | "Before acting, confirm the last three steps in each md; some may already be done and the md not updated." | Caught that the 05-11 log's firewall section was stale; the 04-17 guide is authoritative |

## Why the logs look the way they do

Both incidents follow the same document shape, which I keep for anything that touches system state: final status on top, a reproducible fix, an attempt log with cause and lesson for each failure, a rollback next to every destructive step, a snapshot of the file layout, open items, environment. The shape exists so that the next occurrence is a paste, not an investigation.

## A note on the browser pin

Freezing a browser twelve months behind and disabling its updater is a security cost. It was a deliberate, temporary decision on a personal machine to get session state back, made with that cost in mind. It is documented here as an incident, not as a recommendation.

## Scrubbing

`<user>` replaces the Windows account name; `<theme-extension-id>` replaces one extension id. Product names, install paths, registry keys and file sizes are kept, since the documents are about them.
