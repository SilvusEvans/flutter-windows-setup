---
name: flutter-windows-setup-zh-Hans
description: 在 Windows 上配置或修复 Flutter 开发环境（免管理员权限）。涵盖安装/重装 Flutter SDK、Dart、JDK、Android SDK、Gradle 与 Visual Studio，以及排查 flutter doctor 报错、flutter 命令卡死、flutter 或 pub get 卡住不动、Android licenses 未接受、sdkmanager 找不到 Java，和无法连接 Google 服务器时的网络故障。
agent_created: true
---

# Windows Flutter 环境配置

目标：从一台干净的 Windows 机器，配到 `flutter build apk` 能跑通。全程不需要管理员权限，都用便携 zip 安装。
路径统一用英文：`C:\src`（Flutter）、`C:\tools`（Git/JDK）、`C:\Android`（SDK）。

> 如果所在网络连不上 Google，下面多个步骤会同时失败（SDK 下载、`pub get`、Gradle）。
> 见文末《无法连接 Google 服务器时》一节，里面有报错清单、原因分析和逐步处理办法。

## 语言版本

本文件是简体中文版。其他语言在 `translations/`，与本文件一一对应
（章节、命令、路径完全相同，只有说明文字被翻译）：

| 语言 | 文件 |
|---|---|
| English（默认） | `SKILL.md` |
| 简体中文 | `translations/SKILL.zh-Hans.md` |
| 繁體中文 | `translations/SKILL.zh-Hant.md` |
| 日本語 | `translations/SKILL.ja.md` |

按用户使用的语言选用对应版本。四个版本需同步修改——改动任一版本时，其余三处要一起更新。

## 先判断：要不要装 Visual Studio

VS 只影响 **Windows 桌面端**开发。只做 Android / iOS / Web 的话可以完全跳过，
`flutter build apk` 和 `flutter build web` 不碰 MSVC。别为 doctor 的一个 `[X]` 白装 10 GB。

## Step 0 · 探测（别急着装）

把日志写进文件再读回来——受限环境里控制台输出经常丢。

必查项：已有 flutter/dart/git/java、`winget --version`、磁盘剩余（VS 需要约 10 GB）、是否管理员，
以及 **`flutter\bin` 是否真的存在**（目录看着在，`bin` 可能已经没了）。

**非管理员时 winget 会弹 UAC 卡死十几分钟** → 放弃包管理器，全程用便携 zip。

## Step 1 · 下载源

优先用官方地址。如果你的网络访问不到，镜像列可用——怎么判断见《无法连接 Google 服务器时》。

| 组件 | 官方 | 镜像（连不上 Google 时） |
|---|---|---|
| Flutter SDK | `https://storage.googleapis.com/flutter_infra_release/releases/stable/windows/flutter_windows_<ver>-stable.zip` | `https://storage.flutter-io.cn/flutter_infra_release/releases/stable/windows/flutter_windows_<ver>-stable.zip` |
| 版本清单 | `https://storage.googleapis.com/flutter_infra_release/releases/releases_windows.json` | `https://storage.flutter-io.cn/flutter_infra_release/releases/releases_windows.json` |
| Git | `https://github.com/git-for-windows/git/releases/...` | `https://registry.npmmirror.com/-/binary/git-for-windows/<ver>/MinGit-<ver>-64-bit.zip` |
| JDK 17 | `https://download.java.net/openjdk/jdk17/...` | `https://repo.huaweicloud.com/openjdk/17/openjdk-17_windows-x64_bin.zip` |
| Android cmdline-tools | `https://dl.google.com/android/repository/commandlinetools-win-13114758_latest.zip` | 同一域名——通常即使 `storage.googleapis.com` 不通它也是通的 |
| Android 组件 | `https://dl.google.com/android/repository/platform-tools_r37.0.1-win.zip`、`platform-36_r02.zip`、`build-tools_r36_windows.zip` | 同一域名 |
| Gradle | `https://services.gradle.org/distributions/gradle-<ver>-all.zip` | `https://mirrors.cloud.tencent.com/gradle/gradle-<ver>-all.zip` |
| VS Build Tools | `https://aka.ms/vs/17/release/vs_BuildTools.exe` | 同一域名 |

**下载用 Python 的 `urllib`，不要用 `curl`。** Windows 上 Git Bash 的 curl 经常报
`curl: (6) getaddrinfo() thread failed to start`，跟网络好不好没关系；Python 走系统网络栈，正常可用
（1.8 GB 的 Flutter 包约 167 秒下完）。

```python
import urllib.request
UA = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'}
req = urllib.request.Request(url, headers=UA)   # User-Agent 必带，部分镜像缺了会 403
with urllib.request.urlopen(req, timeout=120) as r, open(dest, 'wb') as f:
    while True:
        b = r.read(1024 * 1024)
        if not b:
            break
        f.write(b)
```

版本号和压缩包名从清单里取，别猜：

```python
import json, urllib.request
d = json.loads(urllib.request.urlopen(
    'https://storage.flutter-io.cn/flutter_infra_release/releases/releases_windows.json', timeout=30).read())
for r in d['releases']:
    if r['hash'] == d['current_release']['stable'] and r['channel'] == 'stable':
        print(d['base_url'] + '/' + r['archive'], r['version'], r.get('dart_sdk_version'))
```

如果有 PowerShell，`System.Net.WebClient` 同样可用，也**必须带 User-Agent**。
`robocopy` 和 `Start-Process -Verb RunAs` 在受限环境里可能被拦——拷贝改用 Bash 的 `cp -a`。

## Step 2 · 解压

Windows 自带的 `tar.exe` 解 Flutter SDK 会失败，`.NET ZipFile::ExtractToDirectory` 也可能被策略拦。
用 Python 的 `zipfile`：

```python
import zipfile, os, sys
z = zipfile.ZipFile(sys.argv[1])
for i in z.infolist():
    if not i.filename.endswith('/'):
        p = os.path.join(sys.argv[2], i.filename)
        os.makedirs(os.path.dirname(p), exist_ok=True)
        open(p, 'wb').write(z.read(i))
```

千万别为了腾地方去删一个 6 万文件的目录——又慢又可能被拦。**改名成 `*_broken` 绕开就行。**

## Step 3 · 环境变量（用户级）

`[Environment]::SetEnvironmentVariable($k, $v, 'User')`，新开终端才生效。

```
JAVA_HOME                       = C:\tools\jdk\jdk-17
ANDROID_HOME / ANDROID_SDK_ROOT = C:\Android
CHROME_EXECUTABLE               = C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe  (没 Chrome 时用 Edge 顶替)
PATH += C:\src\flutter\bin; C:\tools\git\cmd; C:\tools\jdk\jdk-17\bin; C:\Android\platform-tools
```

**只有网络连不上 Google 时**，才额外设置镜像（详见《无法连接 Google 服务器时》）：

```
PUB_HOSTED_URL           = https://pub.flutter-io.cn
FLUTTER_STORAGE_BASE_URL = https://storage.flutter-io.cn
```

**正常网络下不要设这两个变量**——它们是绕行方案而不是默认配置，设了会把你的包流量导到第三方。

## Step 4 · Android SDK 与 licenses

用 `sdkmanager` 装 platform / build-tools / platform-tools，然后**必须真正交互接受一遍 licenses**：

```bash
yes | head -30 | sdkmanager.bat --licenses --sdk_root=C:/Android
# 看到 "All SDK package licenses accepted" 才算完成
```

手写 license hash 不被认可（会停在 "5 of 7 not accepted"）。成功后 `C:\Android\licenses` 有 7 个文件，
doctor 的 Android toolchain 才转绿。

## Step 5 · Gradle 发行版换源

`services.gradle.org` 经常很慢或不可达。改 Flutter 模板，之后新建项目自动生效：

`C:\src\flutter\packages\flutter_tools\templates\app\android.tmpl\gradle\wrapper\gradle-wrapper.properties.tmpl`
```
distributionUrl=https\://mirrors.cloud.tencent.com/gradle/gradle-{{gradleVersion}}-all.zip
```

已存在的项目改各自的 `android/gradle/wrapper/gradle-wrapper.properties`。

> **全新安装的 Flutter 会把这个文件重置回 `services.gradle.org`。** 每次重装或修复 SDK 后都必须重新打这个补丁，
> 否则下一次 `flutter build apk` 会卡在下载 Gradle 上。这是最容易漏掉的一步。

## Step 6 · Visual Studio（仅 Windows 桌面端需要）

```powershell
Start-Process 'D:\tools\vs\vs_BuildTools.exe' -ArgumentList `
  '--installPath','D:\VS','--quiet','--wait','--norestart','--nocache','--add','Microsoft.VisualStudio.Workload.VCTools','--includeRecommended' -Wait
```

把安装器本身下载到 `D:\tools\vs` 并保留 `--installPath D:\VS`，这样约 10 GB 的负载永远不碰 C 盘。VS Code 同理：安装时把位置设为 `D:\VSCode`。

**非管理员必然失败**（退出码 1，且不弹 UAC）。不要反复重试，留一个 `.bat` 让用户右键"以管理员身份运行"。

## Step 7 · 验证

```bash
flutter --version          # 3 秒内返回
flutter doctor             # Flutter / Android toolchain / Network resources 应为 √
flutter build apk --debug  # 端到端
```

## 重装 / 修复

以下症状说明是 **SDK 本身坏了，不是配置问题**：

- `flutter.bat: No such file or directory` → `flutter\bin` 没了。`bin` 里还有 `cache\dart-sdk` 和
  `cache\flutter_tools.snapshot`，丢了等于整个 Dart SDK 都没了。
- `java -version` 报 `Failed setting boot class path` → `jdk-17\lib\modules` 缺失（正常约 125 MB）。

这两种都没法就地恢复，本机也没有 `flutter*.zip` 缓存可回退。重新下载：

1. 从清单里取版本号（Step 1）。
2. 把坏目录 `os.rename` 成 `*_broken`——先别删。
3. 用 Python 下载并解压（Step 1–2）。
4. **先确认** `flutter\bin\flutter.bat` 和 `jdk-17\bin\java.exe` 存在，再往下走。
5. 重新打 Gradle 补丁（Step 5）——全新安装会清掉它。
6. 跑 `flutter --version`。首次运行会下载依赖，约 1 分钟，属正常。
7. 确认无误后再清掉 `*_broken` 残留。

上面 1–7 步就是恢复顺序，照顺序做，不要合并成「删掉重装」一把梭；第 4 步才是区分工具链装好与解压残缺的关键。

## 把工具链搬到非系统盘

C 盘快满时，把工具链挪到 D:/E:。两种做法，选一种。

### 做法 A — junction（保留原路径有效）

在**原路径留一个 junction**，这样 PATH 和 Flutter 内部所有绝对路径都还有效，而真实数据在别的盘。

1. **先复制、验证、再删除。** 用 Bash 的 `cp -a`（`robocopy` 常被拦且会静默失败）。
2. 动源目录之前先比对文件数：
   ```bash
   echo "src=$(find /c/src -type f | wc -l)  dst=$(find /d/src -type f | wc -l)"
   ```
   完全一致才可以继续。
3. 删几万个文件可能被批量删除保护拦住（有些环境限制每会话 50 个文件）。这时**直接给用户一个 `.bat`**——
   `rmdir /S /Q` + `mklink /J`——别在上面反复耗。
4. 改名不算删除：要把目录挪开先用 `mv` 改成 `_junk_*`，清理由 `.bat` 一并做。

`mklink /J <旧路径> <新路径>` 不需要管理员权限。**永远不要搬 agent 自己正在用的工作/缓存目录**（会话会断）。

### 做法 B — 只改指向 PATH 与工具链变量（不复制）

只改 PATH 和工具链变量的指向——不复制任何数据，且在批量拷贝 / 批量删除被拒时同样可行。以下变量要**全部**设置，否则构建会把几个 GB 写回 C 盘：

```powershell
# PUB_CACHE / GRADLE_USER_HOME / ANDROID_USER_HOME 默认都在 C:\Users\<u>\...
$k = [Microsoft.Win32.Registry]::CurrentUser.OpenSubKey('Environment', $true)
$p = [string]$k.GetValue('Path')                      # 不要传第三个参数，见坑位
$p = $p.Replace('C:\src\flutter\bin','D:\src\flutter\bin')
$k.SetValue('Path', $p, [Microsoft.Win32.RegistryValueKind]::ExpandString)
$k.SetValue('JAVA_HOME','D:\tools\jdk\jdk-17',[Microsoft.Win32.RegistryValueKind]::String)
$k.Close()
```

- `[Environment]::SetEnvironmentVariable($k,$v,'User')`（Step 3）**在沙箱 shell 里经常被拦**（报 `cmd.exe cannot be used from the PowerShell tool`），`Set-ItemProperty ... -Type ExpandString` 同样。直接写注册表始终可用，且不需要管理员。
- `$k.GetValue('Path','',$true)` 会抛 `MethodException` —— 第三个参数要 `RegistryValueOptions` 枚举而非 `bool`。改用 `GetValue('Path')`，或 `Get-ItemProperty HKCU:\Environment`。
- 写 `Path` 必须显式给 `RegistryValueKind::ExpandString`，否则 `%VAR%` 引用会被固化。
- **对已有 PATH 做字符串替换，不要整条重建。** 用户 PATH 里含非 ASCII 项，粗暴重写会写乱或丢掉。
- 先备份旧值到文件，回滚就是一行命令。
- **直接写注册表不会广播 `WM_SETTINGCHANGE`**——配置看起来全对，但在用户机器上照样失败。`SetValue` / `Set-ItemProperty` 会让正在运行的 `explorer.exe` 一直持有旧的环境块，于是从开始菜单新开的 CMD 仍报 `'flutter' 不是内部或外部命令`，而 `HKCU:\Environment\Path` 早已正确、重建环境后的 `where flutter` 也能解析。**再次改动 PATH 之前，先回读注册表确认**——问题出在陈旧的环境块，而不是值本身。刷新方式：注销重登、重启「Windows 资源管理器」（任务管理器 → Windows 资源管理器 → 重新启动），或 `setx <无害变量> <值>`（setx 自己会广播，且不碰 PATH）。沙箱里 `Add-Type` 的 P/Invoke `SendMessageTimeout` 通常会被拦。

### PATH 坑位（两种做法都适用）

- **PATH 看起来配好了，`flutter` 却找不到。** PATH 里配了条目，不等于 SDK 真在那儿——解压中断或半途废弃的安装会留下一棵**没有 `bin\` 的目录树**，条目指向空气；而 `flutter doctor` 仍能跑，因为你是用绝对路径启动它的。务必验证真实文件（`Test-Path ...\bin\flutter.bat`、`...\bin\cache\dart-sdk\bin\dart.exe`），并留意 PATH 上出现两份 `platform-tools`（doctor 会报 `Multiple adb binaries found`）。
- **doctor 说 "not on your path"，但 `where` 明明解析正常。** 如果进程环境里**同时存在 `Path` 和 `PATH`**（两个只差大小写的条目），Windows 大小写不敏感的查找命中一个，而 Dart 的 `Platform.environment['PATH']` 命中另一个。于是 doctor 报 `The flutter binary is not on your path`，可 `where.exe flutter` 打印的路径完全正确。用两行 Dart 探针打印 `containsKey("PATH")` 与 `containsKey("Path")`；两者皆为 true 就说明该告警是调用方 shell 的产物——请到**全新终端**里验证后再动手。

## 无法连接 Google 服务器时

Flutter 的默认下载端——`storage.googleapis.com`、`pub.dev`、`dl.google.com`——都在 Google 的基础设施后面。
网络屏蔽 Google 时，会有好几个步骤同时失败：SDK 下载、`pub get`、Gradle 发行版下载。
按下面的顺序从上往下排查。

### 1. 会看到哪些报错

| 出现位置 | 报错信息 | 实际含义 |
|---|---|---|
| `flutter doctor` | `Network resources` 每行都是 `X`，提示 "A network error occurred while checking https://pub.dev" | Flutter 连不上自己的包仓库 |
| `flutter pub get` | `Got socket error trying to find package <名称> at https://pub.dev` | pub 端点不可达（DNS 或 TCP 层） |
| 任何下载 | `SocketException: Failed host lookup: 'storage.googleapis.com'` | DNS 解析失败——域名根本没解析出 IP |
| 任何下载 | `curl: (6) getaddrinfo() thread failed to start` | 这台机器上 Git Bash 的 curl 本身坏了，**是工具问题，不是网络问题** |
| 首次 `flutter --version` | 卡在 `Downloading Dart SDK...`，或 `Unable to update Dart SDK. Retrying...` | 从 Google 存储拉 Dart SDK 失败 |
| `flutter build apk` | 卡在 `Downloading https://services.gradle.org/distributions/...` | Gradle 自己的发行版不可达 |
| Android SDK 工具 | `Warning: Failed to download ...` 之后 `Done`，但包不全 | `dl.google.com` 被部分屏蔽 |
| VS Build Tools | 安装器卡在 0% | 微软 CDN 慢，**不是** Google 的问题 |

### 2. 常见原因（五类）

**A. DNS 解析失败。** `storage.googleapis.com`、`pub.dev` 解析不出来，或解析到错误的／被黑洞的 IP。
特征是报 "Failed host lookup" 而不是单纯超时。

**B. 直连被阻断。** 域名能解析，但 TCP :443 建不起来。下载停在 0%，最后超时。

**C. 代理配置不对。** 要么机器在代理后面但从没为这些工具配过，要么设了代理但代理本身服务不了这些请求。
Flutter 的工具认 `HTTP_PROXY`/`HTTPS_PROXY`，但不是所有工具都认——所以浏览器能上网并不代表工具链能出去。

**D. 安全软件 / 企业防火墙。** 杀毒软件、"网络加速"类工具或企业网关会检查并断开 TLS 连接。
通常表现为时好时坏，而且只对某些域名发作。

**E. 镜像地址失效。** 已经换到镜像了，但那个镜像挂了、限速了，或返回 403
（清华镜像对大文件 GET 会返回 403，这是已知的）。

### 3. 按原因处理

**如果是 A（DNS 失败）** —— 别再试图解析 Google 域名，直接换镜像：

1. 先确认确实是 DNS：
   `python -c "import socket;print(socket.gethostbyname('storage.googleapis.com'))"` —— 如果这里就抛异常，那就是 DNS 问题。
2. 设置镜像环境变量，然后**新开一个终端**：
   ```
   PUB_HOSTED_URL           = https://pub.flutter-io.cn
   FLUTTER_STORAGE_BASE_URL = https://storage.flutter-io.cn
   ```
3. 从 `storage.flutter-io.cn` 下载 Flutter SDK（见 Step 1 表格）。
4. 重新跑 `flutter doctor`。

**如果是 B（直连被阻断）** —— 光换镜像可能不够，因为 `pub get` 仍然需要一条通路：

1. 用短超时测连通性，例如 `curl --max-time 10 -I https://pub.flutter-io.cn`
   （Git Bash 的 curl 坏了就用 PowerShell）。
2. 设置与 A 相同的镜像环境变量。
3. 如果镜像**也**被阻断，那就必须走代理——见 C。

**如果是 C（代理）** —— 先搞清楚有没有代理、代理通不通：

1. 查看当前设置：
   ```
   env | grep -i proxy
   netsh winhttp show proxy
   ```
2. 如果代理存在且可用，为这些工具设置：
   ```
   HTTPS_PROXY = http://<主机>:<端口>
   HTTP_PROXY  = http://<主机>:<端口>
   ```
3. 如果设了代理但代理**已经失效**，排查前先取消它——残留的代理会让 `flutter doctor` 报出跟 Google 毫无关系的网络错误：
   ```
   unset HTTP_PROXY HTTPS_PROXY
   ```
4. 如果代理是唯一的出口，`pub get` 时保留它，但跑 `flutter doctor` 时取消。

**如果是 D（安全软件）** —— 最难发现，因为是间歇性的：

1. 临时关掉杀毒软件／"加速"类工具，重试单次下载。
2. 如果这时能成，就把 `storage.flutter-io.cn`、`pub.flutter-io.cn`、`dl.google.com`
   以及 Flutter/JDK 安装目录加入白名单。
3. 尽量先下载到文件再从本地文件安装，这样传输中途断开不会留下装了一半的 SDK。

**如果是 E（镜像失效）** —— 换之前先验证：

1. 用小请求测镜像（`-I`、`--max-time 10`），别直接开一个 1.8 GB 的下载。
2. 已知可用的替代：Flutter → `storage.flutter-io.cn`；Gradle → `mirrors.cloud.tencent.com`；
   Git → `registry.npmmirror.com`；JDK → `repo.huaweicloud.com`。
   Maven Central 和 Google Maven 往往在其它 Google 域名不通时仍然可用，所以**不要**无谓地换 Maven 仓库。
3. 大文件避开清华镜像——GET 会返回 403。

### 4. 镜像速查

| 工具 | 设置位置 | 值 |
|---|---|---|
| Dart / pub | 环境变量 `PUB_HOSTED_URL` | `https://pub.flutter-io.cn` |
| Flutter SDK | 环境变量 `FLUTTER_STORAGE_BASE_URL` | `https://storage.flutter-io.cn` |
| Gradle | `gradle-wrapper.properties` 的 `distributionUrl` | `https://mirrors.cloud.tencent.com/gradle/gradle-<ver>-all.zip` |
| Android SDK | `ANDROID_HOME` + 仓库配置 | `dl.google.com` 通常可用，不需要镜像 |

### 5. 验证是否修好

```bash
flutter doctor      # Network resources 不应再报错
flutter pub get     # 应能解析依赖，不再抛 socket error
flutter build apk --debug
```

如果镜像也设了、代理也对了，`pub get` 还是失败，说明到镜像的通路是真的被阻断——别再换姿势试了，
如实说明情况，不要反复跑同样的命令。

## 诊断速查表

| 症状 | 根因 | 解法 |
|---|---|---|
| `flutter.bat: No such file or directory` | `flutter\bin` 被删，Dart SDK 一并丢失 | 重装 SDK（见《重装 / 修复》） |
| `java -version` 报 `Failed setting boot class path` | `jdk-17\lib\modules` 缺失 | 重装 JDK |
| `flutter` 任何命令卡死十几分钟 | `bin/cache/flutter_tools.snapshot` 被改名成 `.old1`（重编译被中断） | 把 `.old1` 复制回 `flutter_tools.snapshot`，删掉 `flutter.bat.lock`。之后 `--version` 约 3 秒返回 |
| `flutter build apk` 卡在下载 Gradle | SDK 重装后模板被重置回 `services.gradle.org` | 重新打镜像补丁（Step 5） |
| sdkmanager 报 "Java version 17 or higher is required" | 华为 OpenJDK 17 输出 `openjdk version "17"`，脚本用 `tokens=1,2 delims=.` 解析成 `17` < `170` | 在 `sdkmanager.bat` 解析循环里补 `if "%%c" == "" set versionWithQ=%%b0`，就不再需要 `SKIP_JDK_VERSION_CHECK` |
| licenses 停在 "5 of 7 not accepted" | 手写 hash 不被认可 | 真正交互接受一遍（Step 4） |
| `pub get` 报 `Got socket error trying to find package ...` | 到 pub 没有通路 | 见《无法连接 Google 服务器时》第 3 节。把 `PUB_CACHE`/`GRADLE_USER_HOME` 挪到别处没用 |
| doctor 三项显示 crashed | `reg.exe` / `wmic.exe` 被策略拦 | 真实机器上不会出现，不是故障 |
| doctor 的 Network resources 全 X | 代理或通路被阻断 | 先取消失效代理，再按《无法连接 Google 服务器时》排查 |

## 环境干扰（不是真实故障）

- `reg.exe` / `wmic.exe` 被拦 → doctor 的 "Connected device"、"Windows Version"、"Visual Studio" 三项崩溃，
  报 `Flutter failed to run "reg query ..."`。无害。
- `adb` 守护进程起不来（`could not read ok from ADB Server`）→ 没接设备时很常见，不影响 `flutter build apk`。
- 受限环境里 `pub get` / `build apk` 可能整条跑不通。直接说明，让用户在自己机器上跑这一步，不要反复换姿势重试。

## 没有网络时如何验证工具链

- **`flutter create` / `pub get` / `build apk` 第一道关卡是工具自举，不是你的项目。**
  `bin\internal\shared.bat` 会把 `bin\cache\flutter_tools.stamp` 与基于 `git rev-parse HEAD` 算出的 compile key 比较，
  不一致就在 `packages\flutter_tools` 里重跑 `dart pub upgrade`，打印 `Building flutter tool...`，
  然后卡在 `Running pub upgrade...`。刚解压的 SDK 若 stamp 里是空 revision（文件内容 `":"`，字节
  `22 3A 22 0D 0A`），就**必须先连上 pub 镜像一次**，`create` / `pub get` 才可能工作。这是网络要求，不是配置错误。
  - 手写一个「正确」的 stamp **不能**可靠地跳过重建：`shared.bat` 第 75 行通过嵌套 `FOR /f` 子 shell 取 revision，
    任何禁止嵌套 `cmd.exe` 的策略都会让 revision 恒为空，手写的值永远匹配不上。若做过实验，
    **务必把 stamp 按原始字节还原**——不要在别人的 SDK 上留下未经验证的改动。
- **用 `--no-pub` 在无网络下验证整条工具链。** `flutter create --no-pub <名称>` 会把整个工程骨架
  （`lib/`、`android/`、`web/`、`pubspec.yaml`）生成出来，完全不碰依赖解析。随后读 `android\local.properties`
  确认 SDK 路径、读 `android\gradle\wrapper\gradle-wrapper.properties` 确认镜像。pub 不通时这是最快的真实验证。
- **只验 Dart 一侧：** `dart compile exe hello.dart -o hello.exe` 再运行产物。完全离线，独立于 pub 与 Gradle。
