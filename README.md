# flutter-windows-setup

An [Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) — a plain Markdown
instruction file an AI coding agent loads on demand — for building or repairing a **Flutter Android development
environment on Windows without administrator rights**.

It exists because the usual advice does not survive a locked-down machine: `winget` hangs on a UAC prompt it will
never show, the Visual Studio installer refuses to run, Google and Maven endpoints are unreachable, and `flutter
doctor` reports four unrelated failures at once. This skill records the portable-zip route that works anyway, plus
the specific gotchas that cost hours to rediscover.

## What it covers

| Area | Details |
| --- | --- |
| Install from zips | Flutter SDK, MinGit, OpenJDK 17, Android command-line tools, platform-tools, Gradle |
| Environment variables | `JAVA_HOME`, `ANDROID_HOME`, `CHROME_EXECUTABLE`, `PATH` — written to the `HKCU:\Environment` registry key when the GUI setter is blocked |
| Android SDK | `sdkmanager` + license acceptance, the "hand-written license files are ignored" trap |
| Gradle | Pinning the wrapper to a reachable mirror, offline dependency resolution |
| Visual Studio Build Tools | Unattended `vs_BuildTools.exe` install, and the leave-a-`.bat` fallback when elevation is impossible |
| Relocating off `C:` | Junction vs. registry rewrite, and why a direct registry write does not broadcast `WM_SETTINGCHANGE` |
| Recovery | Ordered repair steps for a half-extracted toolchain |
| Blocked networks | Mirrors for Flutter, Git, JDK, Maven and Google endpoints, and how to test reachability when `ping` is filtered |

## Files

```
SKILL.md                      English, the canonical version
translations/SKILL.zh-Hans.md 简体中文
translations/SKILL.zh-Hant.md 繁體中文
translations/SKILL.ja.md      日本語
```

`SKILL.md` is what the agent reads. The translations are for humans reviewing the procedure; they are kept in sync
with the English file, which stays authoritative if they ever diverge.

## Install

Copy the `flutter-windows-setup` folder into your agent's skills directory:

```
~/.claude/skills/flutter-windows-setup/      Claude Code
~/.qoder/skills/flutter-windows-setup/       Qoder
$CODEX_HOME/skills/flutter-windows-setup/    Codex
```

Restart the session (or reload skills) and the agent picks it up automatically when a task matches — "set up Flutter
on this Windows machine", "flutter doctor says Java is missing", "pub get is stuck". You can also invoke it directly:

```
/flutter-windows-setup
```

## Use it without an agent

The Markdown is a readable runbook on its own. Two cautions before running anything by hand:

- Every path in it (`C:\src`, `C:\tools`, `C:\Android`, `D:\VS`) is an **example**. Substitute your own, and keep them
  free of spaces and non-ASCII characters — that alone causes a class of Gradle failures.
- Check free disk space first. Visual Studio Build Tools needs roughly 10 GB, and the skill assumes nothing writes to
  the system drive unless you decided otherwise.

## Requirements

Windows 10 or 11, a normal user account (no admin), ~10 GB free on a non-system drive, and any of the download
mirrors reachable.

## License

MIT — see [LICENSE](LICENSE).
