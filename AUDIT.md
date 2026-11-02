# Security / correctness audit — claude-usage-touchbar

- Audited commit: `7c7eca6` (felixdivo/claude-usage-touchbar main = tpklo/claude-usage-touchbar `52b933a` + 8 commits). Fixes are in the commits that follow it on this branch.
- Date: 2026-11-02. Target: an Intel 16″ MacBook Pro with a Touch Bar, macOS 26.
- Scope:
  - Read in full: `claude-touchbar.sh`, `Makefile`, `tools/extract-assets.py`, `local.claude-touchbar.plist`, `Info.plist` (plain: LSUIElement, no entitlements), CI and `SECURITY.md`.
  - `main.m`: the app delegate, NSTask, parser, clip selection and palette path were read in full. The drawing code and the `--render/--film` dev paths were scanned for I/O, exec and network calls, not read line by line.
- Verification:
  - `tools/test.sh` passes 23/23. That includes new cache-injection tests and a negative control against the original code.
  - The sandbox was exercised with a local hostile JS fixture.
  - `make assets` ran under the sandbox: all 13 clips extracted. The generated `clawd_presets.h` matched a line-by-line whitelist (numeric data only).
  - `make` builds with no warnings.
  - The app was installed with `install-script` and `install-agent`, then ran under launchd. It polled on its own, held no power assertion and logged no errors.
  - **Not exercised:** the live token-refresh path (`claude -p hi`), and shellcheck.
- **[verified]** marks findings reproduced with a standalone test.

## Findings

| # | Sev | Where | Finding | Status |
|---|-----|-------|---------|--------|
| 1 | **High** (build time) | `tools/extract-assets.py` | `make assets` downloads `creature-engine.js` and 13 HTML pages from claudepix.vercel.app (third party, no licence, no pinning). It runs them with `(0, eval)` in a full-privilege Node process. That would allow arbitrary code execution as the user (`process.mainModule.require("child_process")` is reachable), including reading the Claude Code token via `/usr/bin/security`. No proof of concept was run against the live site. **Second vector:** the eval output's `hold` and grid values were written unvalidated into `clawd_presets.h`, so a JSON string became compiled C source. | **Fixed.** Node runs under `--permission` (fs reads only in the temp dir, no writes, child_process, workers or addons), inside `sandbox-exec` (no network, no exec, writes only in the temp dir), with a bare environment. Every output value is type- and range-checked, and all 13 clips are required. Hostile fixture: child_process, writing to the home dir, reading `~/.ssh`, dlopen, a worker and a TCP connect were all blocked, and the injected `hold` was rejected. Each layer blocks independently. |
| 2 | Medium (multi-user Mac) | `claude-touchbar.sh` cache `/tmp/.claude-usage-$UID` | A predictable path in world-writable `/tmp`. Another local uid can pre-create the file, and its fields reach bash arithmetic. **[verified]** on `/bin/bash 3.2.57`: `STAMP='a[$(cmd)]'` runs `cmd`, which is code execution as the victim every 30 s. It also allows symlink clobbering and spoofed numbers. | **Fixed.** The cache and the refresh marker moved to `~/Library/Caches/local.claude-touchbar/`: 0700, refuses a symlink or a foreign owner, `umask 077`. Every field is integer-checked before arithmetic, and the model name is sanitised. |
| 3 | **Medium** (functional) | `main.m` `NSActivityUserInitiated` | This option includes `NSActivityIdleSystemSleepDisabled` (**[verified]** in the SDK header). While the app runs, **the Mac never idle-sleeps.** The intent was only to defeat App Nap. | **Fixed:** `NSActivityUserInitiatedAllowingIdleSystemSleep`. The frame timer also pauses while the display sleeps. |
| 4 | Medium | `claude-touchbar.sh` `try_refresh` | Once the token reads as expired, it relaunches `claude … -p hi` every 180 s indefinitely. Any Keychain or JSON read failure also counted as "expired", so a broken read with a working CLI meant a real request every 3 min, forever. Each refresh also loads the user's CLAUDE.md, settings and hooks, and probably opens a new 5-hour usage window (unverified). | **Fixed.** One launch per expiry episode. A read failure is `stale`, not `expired`. The CLI runs from the cache dir with `--setting-sources project,local` (unverified for the refresh path; `--bare` is not an option because it disables OAuth). Kill switch: `defaults write local.claude-touchbar AutoRefresh -bool NO`. |
| 5 | Low–Medium (availability) | `main.m` `ClipForMood` + `extract-assets.py` | Clip indices 0–12 are hard-coded, but a failed page was skipped silently. A partial fetch therefore indexed past `kClawdClips` (out-of-bounds read, then a crash loop under `KeepAlive`). | **Fixed.** All 13 clips are required, and indices are bounds-checked. |
| 6 | Low | `claude-touchbar.sh` TTL 60 s | It polls the undocumented `api.anthropic.com/api/oauth/usage` up to once a minute, with no 429 backoff. A sibling project reports throttling below about 180 s (unverified). | Open. |
| 7 | Low | `main.m` `poll` | `readDataToEndOfFile` has no timeout, so a Keychain unlock dialog freezes the readout until it is answered. | Open. |
| 8 | Info | `main.m` `present` | It re-asserts itself over the Touch Bar on every app switch, displacing app-specific controls. | By design. A collapse button exists. |
| 9 | Info | `main.m` `sendEscape:` | `CGEventPost` may need Accessibility/PostEvent TCC (SECURITY.md says so). | — |
| 10 | Info | `/bin/bash -lc` | A login shell sources the user's bash profile on every poll. | Open. |
| 11 | Info | docs | Stale text: field count in SECURITY.md (fixed), and CI and `main.m` comments still mention Claude.app as the asset source. | Partly fixed. |

Also fixed: `security -w` returns the whole credential JSON, `refreshToken` included, into `$cred`, which was only `unset` on one branch. It is now unset on every branch. Shorthand `#rgb` palette colours, previously drawn black, are now expanded.

## What checks out

- The token reaches python and curl over stdin only (`-H @-`), never through argv, env, disk or logs. The only runtime host is `api.anthropic.com` over HTTPS.
- The GUI binary has no networking or Keychain code. Its only subprocess is the script.
- Palette indexing is bounds-checked (`v < palCount`, `palCount ≤ 16`).
- The launchd plist has `KeepAlive` only on crash, `ThrottleInterval 30`, and the Aqua session only.
- There is no telemetry. Usage numbers are cached locally as percentages, a model name and a timestamp.

Note: on a default setup, `/usr/bin/security` is on the ACL of the `Claude Code-credentials` Keychain item. So any process running as the user can read the Claude Code OAuth token without a prompt. That is how Claude Code stores it, not something this project adds, but it is why any build- or run-time code execution here matters.
