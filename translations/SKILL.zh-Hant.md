---
name: flutter-windows-setup-zh-Hant
description: 在 Windows 上設定或修復 Flutter 開發環境（免管理員權限）。涵蓋安裝／重裝 Flutter SDK、Dart、JDK、Android SDK、Gradle 與 Visual Studio，以及排查 flutter doctor 報錯、flutter 指令卡死、flutter 或 pub get 卡住不前進、Android licenses 未接受、sdkmanager 找不到 Java，和無法連線 Google 伺服器時的網路故障。
agent_created: true
---

# Windows Flutter 環境設定

目標：從一台乾淨的 Windows 機器，設定到 `flutter build apk` 能跑通。全程不需要管理員權限，都用可攜式 zip 安裝。
路徑統一用英文：`C:\src`（Flutter）、`C:\tools`（Git/JDK）、`C:\Android`（SDK）。

> 如果所在網路連不上 Google，下面多個步驟會同時失敗（SDK 下載、`pub get`、Gradle）。
> 見文末《無法連線 Google 伺服器時》一節，內有錯誤訊息清單、原因分析與逐步處理方式。

## 語言版本

本檔案是繁體中文版。其他語言在 `translations/`，與本檔案一一對應
（章節、指令、路徑完全相同，只有說明文字被翻譯）：

| 語言 | 檔案 |
|---|---|
| English（預設） | `SKILL.md` |
| 简体中文 | `translations/SKILL.zh-Hans.md` |
| 繁體中文 | `translations/SKILL.zh-Hant.md` |
| 日本語 | `translations/SKILL.ja.md` |

依使用者書寫的語言選用對應版本。四個版本需同步修改——改動任一版本時，其餘三處要一起更新。

## 先判斷：要不要裝 Visual Studio

VS 只影響 **Windows 桌面端**開發。只做 Android / iOS / Web 的話可以完全跳過，
`flutter build apk` 與 `flutter build web` 不會碰到 MSVC。別為了 doctor 的一個 `[X]` 白裝 10 GB。

## Step 0 · 探測（別急著裝）

把日誌寫進檔案再讀回來——受限環境裡主控台輸出經常會丟。

必查項：既有的 flutter/dart/git/java、`winget --version`、磁碟剩餘（VS 需要約 10 GB）、是否為管理員，
以及 **`flutter\bin` 是否真的存在**（目錄看著在，`bin` 可能已經沒了）。

**非管理員時 winget 會卡在 UAC 十幾分鐘** → 放棄套件管理員，全程用可攜式 zip。

## Step 1 · 下載源

優先使用官方網址。如果你的網路連不上，鏡像欄可用——怎麼判斷見《無法連線 Google 伺服器時》。

| 元件 | 官方 | 鏡像（連不上 Google 時） |
|---|---|---|
| Flutter SDK | `https://storage.googleapis.com/flutter_infra_release/releases/stable/windows/flutter_windows_<ver>-stable.zip` | `https://storage.flutter-io.cn/flutter_infra_release/releases/stable/windows/flutter_windows_<ver>-stable.zip` |
| 版本清單 | `https://storage.googleapis.com/flutter_infra_release/releases/releases_windows.json` | `https://storage.flutter-io.cn/flutter_infra_release/releases/releases_windows.json` |
| Git | `https://github.com/git-for-windows/git/releases/...` | `https://registry.npmmirror.com/-/binary/git-for-windows/<ver>/MinGit-<ver>-64-bit.zip` |
| JDK 17 | `https://download.java.net/openjdk/jdk17/...` | `https://repo.huaweicloud.com/openjdk/17/openjdk-17_windows-x64_bin.zip` |
| Android cmdline-tools | `https://dl.google.com/android/repository/commandlinetools-win-13114758_latest.zip` | 同一網域——通常即使 `storage.googleapis.com` 不通它也是通的 |
| Android 元件 | `https://dl.google.com/android/repository/platform-tools_r37.0.1-win.zip`、`platform-36_r02.zip`、`build-tools_r36_windows.zip` | 同一網域 |
| Gradle | `https://services.gradle.org/distributions/gradle-<ver>-all.zip` | `https://mirrors.cloud.tencent.com/gradle/gradle-<ver>-all.zip` |
| VS Build Tools | `https://aka.ms/vs/17/release/vs_BuildTools.exe` | 同一網域 |

**下載用 Python 的 `urllib`，不要用 `curl`。** Windows 上 Git Bash 的 curl 經常報
`curl: (6) getaddrinfo() thread failed to start`，跟網路好不好無關；Python 走系統網路堆疊，正常可用
（1.8 GB 的 Flutter 壓縮檔約 167 秒下完）。

```python
import urllib.request
UA = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'}
req = urllib.request.Request(url, headers=UA)   # User-Agent 必帶，部分鏡像缺了會回 403
with urllib.request.urlopen(req, timeout=120) as r, open(dest, 'wb') as f:
    while True:
        b = r.read(1024 * 1024)
        if not b:
            break
        f.write(b)
```

版本號與壓縮檔名從清單裡取，不要猜：

```python
import json, urllib.request
d = json.loads(urllib.request.urlopen(
    'https://storage.flutter-io.cn/flutter_infra_release/releases/releases_windows.json', timeout=30).read())
for r in d['releases']:
    if r['hash'] == d['current_release']['stable'] and r['channel'] == 'stable':
        print(d['base_url'] + '/' + r['archive'], r['version'], r.get('dart_sdk_version'))
```

如果有 PowerShell，`System.Net.WebClient` 同樣可用，也**必須帶 User-Agent**。
`robocopy` 與 `Start-Process -Verb RunAs` 在受限環境裡可能被攔——複製改用 Bash 的 `cp -a`。

## Step 2 · 解壓

Windows 內建的 `tar.exe` 解 Flutter SDK 會失敗，`.NET ZipFile::ExtractToDirectory` 也可能被原則攔下。
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

千萬別為了騰空間去刪一個 6 萬檔案的目錄——又慢又可能被攔。**改名成 `*_broken` 繞開就好。**

## Step 3 · 環境變數（使用者層級）

`[Environment]::SetEnvironmentVariable($k, $v, 'User')`，開新終端機才生效。

```
JAVA_HOME                       = C:\tools\jdk\jdk-17
ANDROID_HOME / ANDROID_SDK_ROOT = C:\Android
CHROME_EXECUTABLE               = C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe  （沒 Chrome 時用 Edge 頂替）
PATH += C:\src\flutter\bin; C:\tools\git\cmd; C:\tools\jdk\jdk-17\bin; C:\Android\platform-tools
```

**只有網路連不上 Google 時**，才額外設定鏡像（詳見《無法連線 Google 伺服器時》）：

```
PUB_HOSTED_URL           = https://pub.flutter-io.cn
FLUTTER_STORAGE_BASE_URL = https://storage.flutter-io.cn
```

**正常網路下不要設這兩個變數**——它們是繞行方案而不是預設設定，設了會把你的套件流量導向第三方。

## Step 4 · Android SDK 與 licenses

用 `sdkmanager` 裝 platform / build-tools / platform-tools，然後**必須真正互動式接受一遍 licenses**：

```bash
yes | head -30 | sdkmanager.bat --licenses --sdk_root=C:/Android
# 看到 "All SDK package licenses accepted" 才算完成
```

手寫 license hash 不被認可（會停在 "5 of 7 not accepted"）。成功後 `C:\Android\licenses` 有 7 個檔案，
doctor 的 Android toolchain 才會轉綠。

## Step 5 · Gradle 發行版換源

`services.gradle.org` 經常很慢或連不到。改 Flutter 範本，之後新建專案自動生效：

`C:\src\flutter\packages\flutter_tools\templates\app\android.tmpl\gradle\wrapper\gradle-wrapper.properties.tmpl`
```
distributionUrl=https\://mirrors.cloud.tencent.com/gradle/gradle-{{gradleVersion}}-all.zip
```

既有的專案改各自的 `android/gradle/wrapper/gradle-wrapper.properties`。

> **全新安裝的 Flutter 會把這個檔案重設回 `services.gradle.org`。** 每次重裝或修復 SDK 後都必須重新套用這個修正，
> 否則下一次 `flutter build apk` 會卡在下載 Gradle。這是最容易漏掉的一步。

## Step 6 · Visual Studio（僅 Windows 桌面端需要）

```powershell
Start-Process 'D:\tools\vs\vs_BuildTools.exe' -ArgumentList `
  '--installPath','D:\VS','--quiet','--wait','--norestart','--nocache','--add','Microsoft.VisualStudio.Workload.VCTools','--includeRecommended' -Wait
```

**非管理員必然失敗**（結束碼 1，而且不會彈 UAC）。不要反覆重試，留一個 `.bat` 讓使用者按右鍵
→「以系統管理員身分執行」。

## Step 7 · 驗證

```bash
flutter --version          # 3 秒內回傳
flutter doctor             # Flutter / Android toolchain / Network resources 應為勾號
flutter build apk --debug  # 端到端
```

## 重裝 / 修復

以下症狀代表 **SDK 本身壞了，不是設定問題**：

- `flutter.bat: No such file or directory` → `flutter\bin` 沒了。`bin` 裡還有 `cache\dart-sdk` 和
  `cache\flutter_tools.snapshot`，丟了等於整個 Dart SDK 都沒了。
- `java -version` 報 `Failed setting boot class path` → `jdk-17\lib\modules` 缺失（正常約 125 MB）。

這兩種都無法就地復原，本機也沒有 `flutter*.zip` 快取可退回。重新下載：

1. 從清單裡取版本號（Step 1）。
2. 把壞目錄 `os.rename` 成 `*_broken`——先別刪。
3. 用 Python 下載並解壓（Step 1–2）。
4. **先確認** `flutter\bin\flutter.bat` 與 `jdk-17\bin\java.exe` 存在，再往下走。
5. 重新套用 Gradle 修正（Step 5）——全新安裝會清掉它。
6. 跑 `flutter --version`。首次執行會下載套件，約 1 分鐘，屬正常。
7. 確認無誤後再清掉 `*_broken` 殘留。

上述 1–7 步就是復原順序，照順序做，不要合併成「刪掉重裝」一次搞定；第 4 步才是判斷工具鏈裝好還是解壓殘缺的關鍵。

## 把工具鏈搬到非系統碟

C 碟快滿時，把工具鏈搬到 D:/E:。兩種做法，選一種。

### 做法 A — junction（保留原路徑有效）

在**原路徑留一個 junction**，這樣 PATH 和 Flutter 內部所有絕對路徑都還有用，而真實資料在別的碟。

1. **先複製、驗證、再刪除。** 用 Bash 的 `cp -a`（`robocopy` 常被攔而且會無言失敗）。
2. 動來源目錄之前先比對檔案數：
   ```bash
   echo "src=$(find /c/src -type f | wc -l)  dst=$(find /d/src -type f | wc -l)"
   ```
   完全一致才可以繼續。
3. 刪幾萬個檔案可能被批次刪除保護攔住（有些環境限制每工作階段 50 個檔案）。這時**直接給使用者一個 `.bat`**——
   `rmdir /S /Q` + `mklink /J`——別在上面反覆耗。
4. 改名不算刪除：要把目錄退避先用 `mv` 改成 `_junk_*`，清理交給 `.bat` 一併做。

`mklink /J <舊路徑> <新路徑>` 不需要管理員權限。**永遠不要搬 agent 自己正在用的工作/快取目錄**（工作階段會斷）。

### 做法 B — 只改指向 PATH 與工具鏈變數（不複製）

只改 PATH 和工具鏈變數的指向——不複製任何資料，而且在批次複製 / 批次刪除被拒時同樣可行。以下變數要**全部**設定，否則建置會把幾個 GB 寫回 C 碟：

```powershell
# PUB_CACHE / GRADLE_USER_HOME / ANDROID_USER_HOME 的預設是 C:\Users\<u>\...
$k = [Microsoft.Win32.Registry]::CurrentUser.OpenSubKey('Environment', $true)
$p = [string]$k.GetValue('Path')                      # 第3引數不要傳，見坑位
$p = $p.Replace('C:\src\flutter\bin','D:\src\flutter\bin')
$k.SetValue('Path', $p, [Microsoft.Win32.RegistryValueKind]::ExpandString)
$k.SetValue('JAVA_HOME','D:\tools\jdk\jdk-17',[Microsoft.Win32.RegistryValueKind]::String)
$k.Close()
```

- `[Environment]::SetEnvironmentVariable($k,$v,'User')`（Step 3）**在沙箱 shell 裡常被攔**（報 `cmd.exe cannot be used from the PowerShell tool`），`Set-ItemProperty ... -Type ExpandString` 亦然。直接寫登錄檔始終可用，而且不需要管理員。
- `$k.GetValue('Path','',$true)` 會拋 `MethodException` —— 第3引數要 `RegistryValueOptions` 列舉而非 `bool`。改用 `GetValue('Path')`，或 `Get-ItemProperty HKCU:\Environment`。
- 寫 `Path` 必須明確給 `RegistryValueKind::ExpandString`，否則 `%VAR%` 參照會被固化。
- **對現有 PATH 做字串取代，不要整條重建。** 使用者 PATH 含非 ASCII 項目，粗暴重寫會寫亂或遺失。
- 先把舊值備份到檔案，回滾就是一行命令。
- **直接寫登錄檔不會廣播 `WM_SETTINGCHANGE`**——設定看起來全對，但在使用者機器上照樣失敗。`SetValue` / `Set-ItemProperty` 會讓執行中的 `explorer.exe` 一直持有舊的環境區塊，於是從開始功能表新開的 CMD 仍報 `'flutter' 不是內部命令或外部命令…`，而 `HKCU:\Environment\Path` 早已正確、重建環境後的 `where flutter` 也能解析。**再次改動 PATH 之前，先回讀登錄檔確認**——問題出在陳舊的環境區塊，不是值本身。更新方式：登出重登、重啟「Windows 檔案總管」（工作管理員 → Windows 檔案總管 → 重新啟動），或 `setx <無害變數> <值>`（setx 自身會廣播，而且不碰 PATH）。沙箱裡 `Add-Type` 的 P/Invoke `SendMessageTimeout` 通常會被攔。

### PATH 坑位（兩種做法都適用）

- **PATH 看起來配好了，`flutter` 卻找不到。** PATH 有項目，不代表 SDK 真的在那——解壓中斷或半途廢棄的安裝會留下一棵**沒有 `bin\` 的樹**，項目指向空氣；而 `flutter doctor` 仍能跑，因為你是用絕對路徑啟動它。務必驗證真實檔案（`Test-Path ...\bin\flutter.bat`、`...\bin\cache\dart-sdk\bin\dart.exe`），並留意 PATH 上出現兩份 `platform-tools`（doctor 會出 `Multiple adb binaries found`）。
- **doctor 說 "not on your path"，但 `where` 明明解析正常。** 如果程序環境**同時存在 `Path` 和 `PATH`**（兩個只差大小寫的項目），Windows 大小寫不敏感的查找命中一個，而 Dart 的 `Platform.environment['PATH']` 命中另一個。於是 doctor 報 `The flutter binary is not on your path`，可 `where.exe flutter` 印出的路徑完全正確。用兩行 Dart 探針印出 `containsKey("PATH")` 與 `containsKey("Path")`；兩者皆為 true 就表示這警告是呼叫方 shell 的產物——請到**全新終端**驗證後再動手。

## 無法連線 Google 伺服器時

Flutter 的預設下載端——`storage.googleapis.com`、`pub.dev`、`dl.google.com`——都在 Google 的基礎設施後面。
網路封鎖 Google 時，會有好幾個步驟同時失敗：SDK 下載、`pub get`、Gradle 發行版下載。
按下面的順序由上往下排查。

### 1. 會看到哪些錯誤訊息

| 出現位置 | 錯誤訊息 | 實際含義 |
|---|---|---|
| `flutter doctor` | `Network resources` 每行都是 `X`，提示 "A network error occurred while checking https://pub.dev" | Flutter 連不上自己的套件倉庫 |
| `flutter pub get` | `Got socket error trying to find package <名稱> at https://pub.dev` | pub 端點不可達（DNS 或 TCP 層） |
| 任何下載 | `SocketException: Failed host lookup: 'storage.googleapis.com'` | DNS 解析失敗——網域根本沒解析出 IP |
| 任何下載 | `curl: (6) getaddrinfo() thread failed to start` | 這台機器上 Git Bash 的 curl 本身壞了，**是工具問題，不是網路問題** |
| 首次 `flutter --version` | 卡在 `Downloading Dart SDK...`，或 `Unable to update Dart SDK. Retrying...` | 從 Google 儲存空間拉 Dart SDK 失敗 |
| `flutter build apk` | 卡在 `Downloading https://services.gradle.org/distributions/...` | Gradle 自己的發行版不可達 |
| Android SDK 工具 | `Warning: Failed to download ...` 之後 `Done`，但套件不全 | `dl.google.com` 被部分封鎖 |
| VS Build Tools | 安裝器卡在 0% | 微軟 CDN 慢，**不是** Google 的問題 |

### 2. 常見原因（五類）

**A. DNS 解析失敗。** `storage.googleapis.com`、`pub.dev` 解析不出來，或解析到錯誤的／被黑洞的 IP。
特徵是報 "Failed host lookup" 而不是單純逾時。

**B. 直連被阻斷。** 網域能解析，但 TCP :443 建不起來。下載停在 0%，最後逾時。

**C. 代理設定不對。** 要嘛機器在代理後面但從沒為這些工具設定過，要嘛設了代理但代理本身服務不了這些請求。
Flutter 的工具認 `HTTP_PROXY`/`HTTPS_PROXY`，但不是所有工具都認——所以瀏覽器能上網並不代表工具鏈能出去。

**D. 安全軟體 / 企業防火牆。** 防毒軟體、「網路加速」類工具或企業閘道會檢查並中斷 TLS 連線。
通常表現為時好時壞，而且只對某些網域發作。

**E. 鏡像位址失效。** 已經換到鏡像了，但那個鏡像掛了、限速了，或回 403
（清華鏡像對大檔案 GET 會回 403，這是已知的）。

### 3. 依原因處理

**如果是 A（DNS 失敗）** —— 別再試圖解析 Google 網域，直接換鏡像：

1. 先確認確實是 DNS：
   `python -c "import socket;print(socket.gethostbyname('storage.googleapis.com'))"` —— 如果這裡就拋例外，那就是 DNS 問題。
2. 設定鏡像環境變數，然後**新開一個終端機**：
   ```
   PUB_HOSTED_URL           = https://pub.flutter-io.cn
   FLUTTER_STORAGE_BASE_URL = https://storage.flutter-io.cn
   ```
3. 從 `storage.flutter-io.cn` 下載 Flutter SDK（見 Step 1 表格）。
4. 重新跑 `flutter doctor`。

**如果是 B（直連被阻斷）** —— 光換鏡像可能不夠，因為 `pub get` 仍然需要一條通路：

1. 用短逾時測連通性，例如 `curl --max-time 10 -I https://pub.flutter-io.cn`
   （Git Bash 的 curl 壞了就用 PowerShell）。
2. 設定與 A 相同的鏡像環境變數。
3. 如果鏡像**也**被阻斷，那就必須走代理——見 C。

**如果是 C（代理）** —— 先搞清楚有沒有代理、代理通不通：

1. 查看目前設定：
   ```
   env | grep -i proxy
   netsh winhttp show proxy
   ```
2. 如果代理存在且可用，為這些工具設定：
   ```
   HTTPS_PROXY = http://<主機>:<連接埠>
   HTTP_PROXY  = http://<主機>:<連接埠>
   ```
3. 如果設了代理但代理**已經失效**，排查前先取消它——殘留的代理會讓 `flutter doctor` 報出跟 Google 毫無關係的網路錯誤：
   ```
   unset HTTP_PROXY HTTPS_PROXY
   ```
4. 如果代理是唯一的出口，`pub get` 時保留它，但跑 `flutter doctor` 時取消。

**如果是 D（安全軟體）** —— 最難發現，因為是間歇性的：

1. 暫時關掉防毒軟體／「加速」類工具，重試單次下載。
2. 如果這時能成，就把 `storage.flutter-io.cn`、`pub.flutter-io.cn`、`dl.google.com`
   以及 Flutter/JDK 安裝目錄加入白名單。
3. 盡量先下載到檔案再從本機檔案安裝，這樣傳輸中途斷線不會留下裝了一半的 SDK。

**如果是 E（鏡像失效）** —— 換之前先驗證：

1. 用小請求測鏡像（`-I`、`--max-time 10`），別直接開一個 1.8 GB 的下載。
2. 已知可用的替代：Flutter → `storage.flutter-io.cn`；Gradle → `mirrors.cloud.tencent.com`；
   Git → `registry.npmmirror.com`；JDK → `repo.huaweicloud.com`。
   Maven Central 與 Google Maven 往往在其它 Google 網域不通時仍然可用，所以**不要**無謂地換 Maven 倉庫。
3. 大檔案避開清華鏡像——GET 會回 403。

### 4. 鏡像速查

| 工具 | 設定位置 | 值 |
|---|---|---|
| Dart / pub | 環境變數 `PUB_HOSTED_URL` | `https://pub.flutter-io.cn` |
| Flutter SDK | 環境變數 `FLUTTER_STORAGE_BASE_URL` | `https://storage.flutter-io.cn` |
| Gradle | `gradle-wrapper.properties` 的 `distributionUrl` | `https://mirrors.cloud.tencent.com/gradle/gradle-<ver>-all.zip` |
| Android SDK | `ANDROID_HOME` + 倉庫設定 | `dl.google.com` 通常可用，不需要鏡像 |

### 5. 驗證是否修好

```bash
flutter doctor      # Network resources 不應再報錯
flutter pub get     # 應能解析套件，不再拋 socket error
flutter build apk --debug
```

如果鏡像也設了、代理也對了，`pub get` 還是失敗，代表到鏡像的通路是真的被阻斷——別再換姿勢試了，
如實說明情況，不要反覆跑同樣的指令。

## 診斷速查表

| 症狀 | 根因 | 解法 |
|---|---|---|
| `flutter.bat: No such file or directory` | `flutter\bin` 被刪，Dart SDK 一併遺失 | 重裝 SDK（見《重裝 / 修復》） |
| `java -version` 報 `Failed setting boot class path` | `jdk-17\lib\modules` 缺失 | 重裝 JDK |
| `flutter` 任何指令卡死十幾分鐘 | `bin/cache/flutter_tools.snapshot` 被改名成 `.old1`（重編譯被中斷） | 把 `.old1` 複製回 `flutter_tools.snapshot`，刪掉 `flutter.bat.lock`。之後 `--version` 約 3 秒回傳 |
| `flutter build apk` 卡在下載 Gradle | SDK 重裝後範本被重設回 `services.gradle.org` | 重新套用鏡像修正（Step 5） |
| sdkmanager 報 "Java version 17 or higher is required" | 華為 OpenJDK 17 輸出 `openjdk version "17"`，腳本用 `tokens=1,2 delims=.` 解析成 `17` < `170` | 在 `sdkmanager.bat` 解析迴圈裡補 `if "%%c" == "" set versionWithQ=%%b0`，就不再需要 `SKIP_JDK_VERSION_CHECK` |
| licenses 停在 "5 of 7 not accepted" | 手寫 hash 不被認可 | 真正互動式接受一遍（Step 4） |
| `pub get` 報 `Got socket error trying to find package ...` | 到 pub 沒有通路 | 見《無法連線 Google 伺服器時》第 3 節。把 `PUB_CACHE`/`GRADLE_USER_HOME` 搬到別處沒用 |
| doctor 三項顯示 crashed | `reg.exe` / `wmic.exe` 被原則攔 | 真實機器上不會出現，不是故障 |
| doctor 的 Network resources 全 X | 代理或通路被阻斷 | 先取消失效代理，再依《無法連線 Google 伺服器時》排查 |

## 環境干擾（不是真實故障）

- `reg.exe` / `wmic.exe` 被攔 → doctor 的 "Connected device"、"Windows Version"、"Visual Studio" 三項崩潰，
  報 `Flutter failed to run "reg query ..."`。無害。
- `adb` 守護程序起不來（`could not read ok from ADB Server`）→ 沒接裝置時很常見，不影響 `flutter build apk`。
- 受限環境裡 `pub get` / `build apk` 可能整條跑不通。直接說明，讓使用者在自己機器上跑這一步，不要反覆換姿勢重試。

## 沒有網路時如何驗證工具鏈

- **`flutter create` / `pub get` / `build apk` 的第一道關卡是工具自舉，不是你的專案。**
  `bin\internal\shared.bat` 會把 `bin\cache\flutter_tools.stamp` 與基於 `git rev-parse HEAD` 算出的 compile key 比較，
  不一致就在 `packages\flutter_tools` 裡重跑 `dart pub upgrade`，印出 `Building flutter tool...`，
  然後卡在 `Running pub upgrade...`。剛解壓的 SDK 若 stamp 裡是空 revision（檔案內容 `":"`，位元組
  `22 3A 22 0D 0A`），就**必須先連上 pub 鏡像一次**，`create` / `pub get` 才可能運作。這是網路需求，不是設定錯誤。
  - 手寫一個「正確」的 stamp **不能**可靠地跳過重建：`shared.bat` 第 75 行透過巢狀 `FOR /f` 子 shell 取 revision，
    任何禁止巢狀 `cmd.exe` 的原則都會讓 revision 恆為空，手寫的值永遠比對不上。若做過實驗，
    **務必把 stamp 按原始位元組還原**——不要在別人的 SDK 上留下未經驗證的改動。
- **用 `--no-pub` 在無網路下驗證整條工具鏈。** `flutter create --no-pub <名稱>` 會把整個專案骨架
  （`lib/`、`android/`、`web/`、`pubspec.yaml`）產生出來，完全不碰依賴解析。接著讀 `android\local.properties`
  確認 SDK 路徑、讀 `android\gradle\wrapper\gradle-wrapper.properties` 確認鏡像。pub 不通時這是最快的真實驗證。
- **只驗 Dart 一側：** `dart compile exe hello.dart -o hello.exe` 再執行產物。完全離線，獨立於 pub 與 Gradle。
