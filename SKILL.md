---
name: flutter-windows-setup
description: Set up or repair a Flutter dev environment on Windows (no admin rights needed). Covers installing/reinstalling the Flutter SDK, Dart, JDK, Android SDK, Gradle and Visual Studio, plus troubleshooting flutter doctor errors, a hanging flutter command, a stuck flutter or pub get, unaccepted Android licenses, sdkmanager not finding Java, and network failures when Google servers are unreachable.
agent_created: true
---

# Flutter Environment Setup on Windows

Goal: get from a bare Windows box to a working `flutter build apk`. No admin rights required — everything installs
from portable zips. Use English-only paths: `C:\src` (Flutter), `C:\tools` (Git/JDK), `C:\Android` (SDK).

> On a network that cannot reach Google, several steps below fail at once (SDK download, `pub get`, Gradle).
> See **When Google is unreachable** near the end for symptoms, causes and step-by-step fixes.

## Language versions

This file is the English version (default). Other languages live in `translations/` and correspond one-to-one with this file
(identical sections, commands and paths — only the prose is translated):

| Language | File |
|---|---|
| English (default) | `SKILL.md` |
| 简体中文 | `translations/SKILL.zh-Hans.md` |
| 繁體中文 | `translations/SKILL.zh-Hant.md` |
| 日本語 | `translations/SKILL.ja.md` |

Pick the version matching the language the user is writing in. All four must stay in sync — whenever you edit one,
apply the same change to the other three.

## Decide first: do you even need Visual Studio?

VS only affects **Windows desktop** development. If you target Android / iOS / Web only, skip it entirely —
`flutter build apk` and `flutter build web` never touch MSVC. Don't install 10 GB just to clear one `[X]` in doctor.

## Step 0 · Probe first (don't install yet)

Write logs to a file and read the file back — console output is often lost in restricted environments.

Check: existing flutter/dart/git/java, `winget --version`, free disk space (VS needs ~10 GB), whether you are admin,
and **whether `flutter\bin` actually exists** (a directory that looks present can be missing its `bin`).

**Without admin rights, winget hangs on a UAC prompt for ten-plus minutes** → abandon package managers, use portable zips.

## Step 1 · Download sources

Prefer the official URLs. If they are unreachable from your network, the mirror column works — see
**When Google is unreachable** for how to diagnose that.

| Component | Official | Mirror (if Google is unreachable) |
|---|---|---|
| Flutter SDK | `https://storage.googleapis.com/flutter_infra_release/releases/stable/windows/flutter_windows_<ver>-stable.zip` | `https://storage.flutter-io.cn/flutter_infra_release/releases/stable/windows/flutter_windows_<ver>-stable.zip` |
| Version manifest | `https://storage.googleapis.com/flutter_infra_release/releases/releases_windows.json` | `https://storage.flutter-io.cn/flutter_infra_release/releases/releases_windows.json` |
| Git | `https://github.com/git-for-windows/git/releases/...` | `https://registry.npmmirror.com/-/binary/git-for-windows/<ver>/MinGit-<ver>-64-bit.zip` |
| JDK 17 | `https://download.java.net/openjdk/jdk17/...` | `https://repo.huaweicloud.com/openjdk/17/openjdk-17_windows-x64_bin.zip` |
| Android cmdline-tools | `https://dl.google.com/android/repository/commandlinetools-win-13114758_latest.zip` | same host — usually reachable even when `storage.googleapis.com` is not |
| Android components | `https://dl.google.com/android/repository/platform-tools_r37.0.1-win.zip`, `platform-36_r02.zip`, `build-tools_r36_windows.zip` | same host |
| Gradle | `https://services.gradle.org/distributions/gradle-<ver>-all.zip` | `https://mirrors.cloud.tencent.com/gradle/gradle-<ver>-all.zip` |
| VS Build Tools | `https://aka.ms/vs/17/release/vs_BuildTools.exe` | same host |

**Download with Python's `urllib`, not `curl`.** On Windows, Git Bash's curl frequently fails with
`curl: (6) getaddrinfo() thread failed to start` regardless of connectivity; Python uses the OS network stack and works
(a 1.8 GB Flutter zip completes in ~167 s).

```python
import urllib.request
UA = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'}
req = urllib.request.Request(url, headers=UA)   # User-Agent is mandatory: some mirrors 403 without it
with urllib.request.urlopen(req, timeout=120) as r, open(dest, 'wb') as f:
    while True:
        b = r.read(1024 * 1024)
        if not b:
            break
        f.write(b)
```

Resolve the exact Flutter version and archive name from the manifest rather than guessing:

```python
import json, urllib.request
d = json.loads(urllib.request.urlopen(
    'https://storage.flutter-io.cn/flutter_infra_release/releases/releases_windows.json', timeout=30).read())
for r in d['releases']:
    if r['hash'] == d['current_release']['stable'] and r['channel'] == 'stable':
        print(d['base_url'] + '/' + r['archive'], r['version'], r.get('dart_sdk_version'))
```

If you have PowerShell, `System.Net.WebClient` also works and equally needs a User-Agent. `robocopy` and
`Start-Process -Verb RunAs` may be blocked in restricted environments — fall back to Bash `cp -a` for copies.

## Step 2 · Extract

Windows' built-in `tar.exe` fails on the Flutter SDK and `.NET ZipFile::ExtractToDirectory` may be blocked by policy.
Python `zipfile` is the reliable path:

```python
import zipfile, os, sys
z = zipfile.ZipFile(sys.argv[1])
for i in z.infolist():
    if not i.filename.endswith('/'):
        p = os.path.join(sys.argv[2], i.filename)
        os.makedirs(os.path.dirname(p), exist_ok=True)
        open(p, 'wb').write(z.read(i))
```

Never `Remove-Item` a 60k-file directory to make room — it is slow and may be blocked. **Rename it to `*_broken` and move on.**

## Step 3 · Environment variables (user level)

`[Environment]::SetEnvironmentVariable($k, $v, 'User')` — takes effect in a new terminal.

```
JAVA_HOME                       = C:\tools\jdk\jdk-17
ANDROID_HOME / ANDROID_SDK_ROOT = C:\Android
CHROME_EXECUTABLE               = C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe  (substitute Edge when Chrome is absent)
PATH += C:\src\flutter\bin; C:\tools\git\cmd; C:\tools\jdk\jdk-17\bin; C:\Android\platform-tools
```

Only if your network cannot reach Google, also set the mirrors (see **When Google is unreachable**):

```
PUB_HOSTED_URL           = https://pub.flutter-io.cn
FLUTTER_STORAGE_BASE_URL = https://storage.flutter-io.cn
```

**Leave the mirror variables unset on a normal network** — they are a workaround, not a default, and they route your
package traffic through a third party.

## Step 4 · Android SDK and licenses

Use `sdkmanager` for platform / build-tools / platform-tools, then **accept the licenses genuinely and interactively**:

```bash
yes | head -30 | sdkmanager.bat --licenses --sdk_root=C:/Android
# Only done when you see "All SDK package licenses accepted"
```

Hand-written license hash files are not honored (it stalls at "5 of 7 not accepted"). On success `C:\Android\licenses`
holds 7 files and doctor's Android toolchain turns green.

## Step 5 · Gradle distribution source

`services.gradle.org` is often slow or unreachable. Patch the Flutter template so every new project picks it up:

`C:\src\flutter\packages\flutter_tools\templates\app\android.tmpl\gradle\wrapper\gradle-wrapper.properties.tmpl`
```
distributionUrl=https\://mirrors.cloud.tencent.com/gradle/gradle-{{gradleVersion}}-all.zip
```

Existing projects: edit their own `android/gradle/wrapper/gradle-wrapper.properties`.

> **A fresh Flutter install resets this file back to `services.gradle.org`.** Any reinstall or SDK repair must re-apply
> this patch, or the next `flutter build apk` hangs fetching Gradle. This is the single most commonly forgotten step.

## Step 6 · Visual Studio (only for Windows desktop targets)

```powershell
Start-Process 'D:\tools\vs\vs_BuildTools.exe' -ArgumentList `
  '--installPath','D:\VS','--quiet','--wait','--norestart','--nocache','--add','Microsoft.VisualStudio.Workload.VCTools','--includeRecommended' -Wait
```

**Without admin rights it always fails** (exit code 1, and no UAC prompt appears). Don't retry in a loop — leave a `.bat`
behind for the user to right-click → "Run as administrator".

Download the installer itself to `D:\tools\vs` and keep the `--installPath D:\VS` flag so the ~10 GB payload never
touches C:. The same rule applies to VS Code: run its installer with the install location set to `D:\VSCode`. See
**Relocating the toolchain off the system drive** if you later need to move an existing install.

## Step 7 · Verify

```bash
flutter --version          # should return within 3 seconds
flutter doctor             # Flutter / Android toolchain / Network resources should be checkmarks
flutter build apk --debug  # end to end
```

## Reinstall / repair

Symptoms that mean the SDK itself is damaged, not misconfigured:

- `flutter.bat: No such file or directory` → `flutter\bin` is gone. `bin` also holds `cache\dart-sdk` and
  `cache\flutter_tools.snapshot`, so losing it costs the whole Dart SDK.
- `java -version` → `Failed setting boot class path` → `jdk-17\lib\modules` is missing (normally ~125 MB).

Both are unrecoverable in place; there is no `flutter*.zip` cache to fall back on. Re-download:

1. Resolve the version from the manifest (Step 1).
2. `os.rename` the broken dir to `*_broken` — do not delete it yet.
3. Download with Python and extract (Steps 1–2).
4. Confirm `flutter\bin\flutter.bat` and `jdk-17\bin\java.exe` exist **before** trusting anything else.
5. Re-apply the Gradle patch (Step 5) — a fresh install wipes it.
6. Run `flutter --version`. First run downloads packages and takes ~1 min; that is normal.
7. Only then remove the `*_broken` leftovers.

Steps 1–7 are the recovery order — follow them as written rather than collapsing them into a single
delete-and-reinstall; step 4 is what separates a working toolchain from a half-extracted one.

## Relocating the toolchain off the system drive

When C: is nearly full, move the toolchain to D:/E:. Two methods — pick one.

### Method A — junction (keeps the original path valid)

Leave a **junction** at the original path so PATH and every absolute path inside Flutter stay valid while the real data lives elsewhere.

1. **Copy first, verify, then delete.** Use Bash `cp -a` (`robocopy` is often blocked and fails silently).
2. Compare file counts before touching the source:
   ```bash
   echo "src=$(find /c/src -type f | wc -l)  dst=$(find /d/src -type f | wc -l)"
   ```
   Only proceed when they match exactly.
3. Deleting tens of thousands of files may be blocked by bulk-delete guards (some environments cap it at 50 files per session). If so, **hand the user a `.bat`** — `rmdir /S /Q` + `mklink /J` — instead of burning turns on it.
4. Renaming is not deleting: to move a directory out of the way, `mv` it to `_junk_*` and clean it up in the `.bat`.

`mklink /J <old> <new>` needs no admin rights. Never relocate the agent's own working/cache directory while a session is live.

### Method B — repoint PATH and toolchain variables (no copy)

Simply repoint PATH and the toolchain variables — nothing is copied, and it works when a bulk copy / bulk delete is refused. Set **all** of these or a build writes gigabytes back to C:

```powershell
# PUB_CACHE / GRADLE_USER_HOME / ANDROID_USER_HOME default to C:\Users\<u>\...
$k = [Microsoft.Win32.Registry]::CurrentUser.OpenSubKey('Environment', $true)
$p = [string]$k.GetValue('Path')                      # no 3rd arg - see gotchas
$p = $p.Replace('C:\src\flutter\bin','D:\src\flutter\bin')
$k.SetValue('Path', $p, [Microsoft.Win32.RegistryValueKind]::ExpandString)
$k.SetValue('JAVA_HOME','D:\tools\jdk\jdk-17',[Microsoft.Win32.RegistryValueKind]::String)
$k.Close()
```

- `[Environment]::SetEnvironmentVariable($k,$v,'User')` (Step 3) is **often blocked in a sandboxed shell** (`cmd.exe cannot be used from the PowerShell tool`), as is `Set-ItemProperty ... -Type ExpandString`. Writing the registry directly always works and needs no admin.
- `$k.GetValue('Path','',$true)` throws `MethodException` — the 3rd parameter wants `RegistryValueOptions`, not `bool`. Use `GetValue('Path')`, or `Get-ItemProperty HKCU:\Environment`.
- Always pass `RegistryValueKind::ExpandString` for `Path`, or `%VAR%` references get frozen.
- **String-replace the existing PATH; never rebuild it.** User PATHs hold non-ASCII entries a naive rewrite mangles.
- Back up the old values to a file first; rollback becomes a one-liner.
- **A direct registry write does not broadcast `WM_SETTINGCHANGE`** — the change looks complete and still fails on the user's machine. `SetValue` / `Set-ItemProperty` leave the running `explorer.exe` holding its old environment block, so a CMD opened from the Start menu keeps reporting `'flutter' is not recognized` while `HKCU:\Environment\Path` is already correct and `where flutter` resolves fine in a rebuilt shell. **Read the registry back to confirm before touching PATH again** — the fault is the stale environment, not the value. Refresh with a logoff/logon, a restart of Windows Explorer (Task Manager → Windows Explorer → Restart), or `setx <harmless-var> <value>` (setx itself broadcasts; it leaves PATH alone). P/Invoke `SendMessageTimeout` through `Add-Type` is normally blocked in a sandbox.

### PATH pitfalls (both methods)

- **`flutter` "not found" while PATH looks correct.** A configured PATH entry is not proof the SDK is there — a partially extracted or abandoned install leaves a tree with **no `bin\` in it**, the entry points at nothing, yet `flutter doctor` still runs because you launched it by absolute path. Test the real files (`Test-Path ...\bin\flutter.bat`, `...\bin\cache\dart-sdk\bin\dart.exe`), and watch for two `platform-tools` on PATH (doctor: `Multiple adb binaries found`).
- **doctor says "not on your path" while `where` resolves fine.** If the process environment holds **both `Path` and `PATH`** (two entries differing only in case), Windows' case-insensitive lookup hits one while Dart's `Platform.environment['PATH']` hits the other. doctor then warns `The flutter binary is not on your path` although `where.exe flutter` prints the right path. Confirm with a two-line Dart probe printing `containsKey("PATH")` and `containsKey("Path")`; both true means the warning is an artifact of the invoking shell — verify in a brand-new terminal before changing anything.

## When Google is unreachable

Flutter's default endpoints — `storage.googleapis.com`, `pub.dev`, `dl.google.com` — all sit behind Google
infrastructure, so a network that blocks Google breaks several steps at once: the SDK download, `pub get`, and the
Gradle distribution fetch. Work through this section top to bottom.

### 1. What you will see

| Where | Message | What it actually means |
|---|---|---|
| `flutter doctor` | `Network resources` → every line `X`, "A network error occurred while checking https://pub.dev" | Flutter cannot reach its own package registry |
| `flutter pub get` | `Got socket error trying to find package <name> at https://pub.dev` | The pub endpoint is unreachable (DNS or TCP) |
| any download | `SocketException: Failed host lookup: 'storage.googleapis.com'` | DNS resolution failed — the name never resolved to an IP |
| any download | `curl: (6) getaddrinfo() thread failed to start` | Git Bash's curl is broken on this machine — **a tool problem, not a network one** |
| first `flutter --version` | `Downloading Dart SDK...` then hangs, or `Unable to update Dart SDK. Retrying...` | Fetching the Dart SDK from Google storage failed |
| `flutter build apk` | hangs on `Downloading https://services.gradle.org/distributions/...` | Gradle's own distribution is unreachable |
| Android SDK tools | `Warning: Failed to download ...` then `Done` with missing packages | `dl.google.com` partially blocked |
| VS Build Tools | installer stalls at 0% | Microsoft's CDN is slow — **not** a Google issue |

### 2. Why it fails — the five usual causes

**A. DNS resolution fails.** `storage.googleapis.com` and `pub.dev` do not resolve, or resolve to a wrong/blackholed IP.
The giveaway is "Failed host lookup" rather than a plain timeout.

**B. Direct connection blocked.** The name resolves, but TCP :443 never completes. Downloads sit at 0% and eventually
time out.

**C. Proxy misconfigured.** Either the machine sits behind a proxy that was never configured for these tools, or a proxy
env var is set that the proxy itself cannot serve. Flutter's tools honour `HTTP_PROXY`/`HTTPS_PROXY`, but not every tool
does — so a working browser does not prove the toolchain can get out.

**D. Security software / corporate firewall.** Antivirus, "network acceleration" utilities, or an enterprise gateway
inspects and drops the TLS connection. Often intermittent, and often only for certain hosts.

**E. Wrong or dead mirror.** You already switched to a mirror, but that mirror is down, rate-limited, or returns 403
(the Tsinghua mirror is known to 403 on large-file GETs).

### 3. What to do, per cause

**If A (DNS fails)** — stop trying to resolve Google names and switch to mirrors:

1. Confirm it is DNS:
   `python -c "import socket;print(socket.gethostbyname('storage.googleapis.com'))"` — if this raises, DNS is the cause.
2. Set the mirror env vars, then open a **new** terminal:
   ```
   PUB_HOSTED_URL           = https://pub.flutter-io.cn
   FLUTTER_STORAGE_BASE_URL = https://storage.flutter-io.cn
   ```
3. Download the Flutter SDK from `storage.flutter-io.cn` (Step 1 table).
4. Re-run `flutter doctor`.

**If B (connection blocked)** — mirrors alone may not be enough, because `pub get` still needs a route:

1. Test reachability with a short timeout, e.g. `curl --max-time 10 -I https://pub.flutter-io.cn`
   (use PowerShell if Git Bash curl is broken).
2. Apply the same mirror env vars as in A.
3. If the mirror is *also* blocked, you must go through a proxy — see C.

**If C (proxy)** — find out whether a proxy exists and whether it works:

1. Inspect the current settings:
   ```
   env | grep -i proxy
   netsh winhttp show proxy
   ```
2. If a proxy exists and works, set it for the tools:
   ```
   HTTPS_PROXY = http://<host>:<port>
   HTTP_PROXY  = http://<host>:<port>
   ```
3. If a proxy is set but is *dead*, unset it before diagnosing — a stale proxy makes `flutter doctor` report a network
   failure that has nothing to do with Google:
   ```
   unset HTTP_PROXY HTTPS_PROXY
   ```
4. Keep the proxy for `pub get` if it is the only route out, but unset it when running `flutter doctor`.

**If D (security software)** — the hardest to spot, because it is intermittent:

1. Temporarily disable antivirus / "acceleration" tools and retry a single download.
2. If it then succeeds, whitelist `storage.flutter-io.cn`, `pub.flutter-io.cn`, `dl.google.com` and the Flutter/JDK
   install directories.
3. Prefer downloading to a file first and installing from the local file, so a mid-transfer drop cannot leave a
   half-installed SDK.

**If E (dead mirror)** — verify before switching again:

1. Test the mirror URL with a small request (`-I`, `--max-time 10`) instead of starting a 1.8 GB download.
2. Known-good alternates: Flutter → `storage.flutter-io.cn`; Gradle → `mirrors.cloud.tencent.com`; Git →
   `registry.npmmirror.com`; JDK → `repo.huaweicloud.com`. Maven Central and Google Maven tend to work even when other
   Google hosts do not, so **do not** swap Maven repositories unnecessarily.
3. Avoid the Tsinghua mirror for large files — it returns 403 on GETs.

### 4. Mirror quick reference

| Tool | Where to set it | Value |
|---|---|---|
| Dart / pub | env `PUB_HOSTED_URL` | `https://pub.flutter-io.cn` |
| Flutter SDK | env `FLUTTER_STORAGE_BASE_URL` | `https://storage.flutter-io.cn` |
| Gradle | `gradle-wrapper.properties` → `distributionUrl` | `https://mirrors.cloud.tencent.com/gradle/gradle-<ver>-all.zip` |
| Android SDK | `ANDROID_HOME` + repo config | `dl.google.com` usually works; no mirror needed |

### 5. Confirm the fix

```bash
flutter doctor      # Network resources should stop reporting errors
flutter pub get     # should resolve packages, not raise a socket error
flutter build apk --debug
```

If `pub get` still fails after the mirrors are set and the proxy is correct, the route to the mirror is genuinely
blocked — stop trying variations and say so plainly, rather than looping on the same commands.

## Diagnostic quick reference

| Symptom | Root cause | Fix |
|---|---|---|
| `flutter.bat: No such file or directory` | `flutter\bin` deleted; takes the Dart SDK with it | Reinstall the SDK (see *Reinstall / repair*) |
| `java -version` → `Failed setting boot class path` | `jdk-17\lib\modules` missing | Reinstall the JDK |
| Any `flutter` command hangs for ten-plus minutes | `bin/cache/flutter_tools.snapshot` renamed to `.old1` (interrupted recompile) | Copy `.old1` back to `flutter_tools.snapshot`, delete `flutter.bat.lock`. `--version` returns in ~3 s afterwards |
| `flutter build apk` hangs downloading Gradle | SDK was reinstalled and the template reverted to `services.gradle.org` | Re-apply the mirror (Step 5) |
| sdkmanager says "Java version 17 or higher is required" | Huawei OpenJDK 17 prints `openjdk version "17"`; the script's `tokens=1,2 delims=.` parse yields `17` < `170` | Add `if "%%c" == "" set versionWithQ=%%b0` in the parse loop of `sdkmanager.bat`; `SKIP_JDK_VERSION_CHECK` then becomes unnecessary |
| Licenses stall at "5 of 7 not accepted" | Hand-written hashes aren't honored | Accept them interactively (Step 4) |
| `pub get` → `Got socket error trying to find package ...` | No route to pub | See **When Google is unreachable** §3. Relocating `PUB_CACHE`/`GRADLE_USER_HOME` does not help |
| doctor reports three checks crashed | `reg.exe` / `wmic.exe` blocked by policy | Not a real failure on a normal machine |
| doctor's Network resources all fail | Proxy or blocked route | Unset a stale proxy first; then see **When Google is unreachable** |

## Environment interference (not real failures)

- `reg.exe` / `wmic.exe` blocked → doctor's "Connected device", "Windows Version" and "Visual Studio" checks crash with
  `Flutter failed to run "reg query ..."`. Harmless.
- `adb` daemon won't start (`could not read ok from ADB Server`) → common with no device attached; does not affect
  `flutter build apk`.
- In a restricted environment `pub get` / `build apk` may be impossible end to end. Say so plainly and let the user run
  that step on their own machine rather than retrying variations.

## Verifying a toolchain without network

- **`flutter create` / `pub get` / `build apk` first stop at the tool bootstrap, not at your project.**
  `bin\internal\shared.bat` compares `bin\cache\flutter_tools.stamp` against a compile key built from
  `git rev-parse HEAD` and re-runs `dart pub upgrade` inside `packages\flutter_tools` whenever the two differ,
  printing `Building flutter tool...` and then hanging on `Running pub upgrade...`. A freshly extracted SDK whose
  stamp holds an empty revision (file content `":"`, bytes `22 3A 22 0D 0A`) **must reach the pub mirror once**
  before `create` / `pub get` can work at all. That is a network requirement, not a configuration fault.
  - Hand-writing a "correct" stamp does **not** reliably skip the rebuild: line 75 of `shared.bat` obtains the revision
    through a nested `FOR /f` sub-shell, so any policy that blocks nested `cmd.exe` leaves the revision empty and no
    hand-written value can ever match. If you experiment, **restore the stamp byte-exact** — never leave an unverified
    edit on someone's SDK.
- **Use `--no-pub` to prove the toolchain end to end with no network.** `flutter create --no-pub <name>` scaffolds the
  whole project (`lib/`, `android/`, `web/`, `pubspec.yaml`) and skips dependency resolution entirely. Then read
  `android\local.properties` to confirm the SDK path and `android\gradle\wrapper\gradle-wrapper.properties` to confirm
  the mirror. Fastest real check when pub is unreachable.
- **For Dart alone:** `dart compile exe hello.dart -o hello.exe` then run the binary. Fully offline, and it proves the
  compiler + VM independent of pub and Gradle.
