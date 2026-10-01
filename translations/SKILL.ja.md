---
name: flutter-windows-setup-ja
description: Windows で Flutter 開発環境を構築・修復する（管理者権限不要）。Flutter SDK / Dart / JDK / Android SDK / Gradle / Visual Studio のインストール・再インストール、flutter doctor のエラー、flutter コマンドのフリーズ、flutter や pub get が進まない、Android licenses が未承諾、sdkmanager が Java を見つけられない、Google サーバーに到達できない場合のネットワーク障害の切り分けに使用する。
agent_created: true
---

# Windows での Flutter 環境構築

目的：まっさらな Windows マシンから `flutter build apk` が通る状態にする。管理者権限は不要で、すべてポータブル zip で導入する。
パスはすべて英数字：`C:\src`（Flutter）、`C:\tools`（Git/JDK）、`C:\Android`（SDK）。

> Google に到達できないネットワークでは、以下の複数手順が同時に失敗する（SDK ダウンロード、`pub get`、Gradle）。
> 末尾の「Google サーバーに接続できないとき」に、エラーメッセージ・原因・手順別の対処をまとめてある。

## 言語バージョン

このファイルは日本語版である。他の言語は `translations/` にあり、本ファイルと一対一で対応する
（章立て・コマンド・パスは完全に同一で、説明文のみ翻訳されている）：

| 言語 | ファイル |
|---|---|
| English（既定） | `SKILL.md` |
| 简体中文 | `translations/SKILL.zh-Hans.md` |
| 繁體中文 | `translations/SKILL.zh-Hant.md` |
| 日本語 | `translations/SKILL.ja.md` |

ユーザーが書いている言語に合う版を使う。4 版は同期を保つこと——いずれかを編集したら、
残り 3 つにも同じ変更を適用する。

## まず判断：Visual Studio は本当に必要か

VS が影響するのは **Windows デスクトップ**向け開発だけである。Android / iOS / Web しか対象にしないなら丸ごとスキップしてよい。
`flutter build apk` も `flutter build web` も MSVC を使わない。doctor の `[X]` を一つ消すためだけに 10 GB を入れてはならない。

## Step 0 · まず調査する（急いで入れない）

ログはファイルに書き出してから読み返す——制限環境ではコンソール出力が失われやすい。

確認項目：既存の flutter/dart/git/java、`winget --version`、空き容量（VS は約 10 GB）、管理者かどうか、
そして **`flutter\bin` が実在するか**（ディレクトリはあっても `bin` が消えていることがある）。

**管理者権限がない場合、winget は UAC で十数分フリーズする** → パッケージマネージャは諦め、ポータブル zip で通す。

## Step 1 · ダウンロード元

公式 URL を優先する。自分のネットワークから到達できない場合は「ミラー」列が使える——判断方法は
「Google サーバーに接続できないとき」を参照。

| コンポーネント | 公式 | ミラー（Google に到達できない場合） |
|---|---|---|
| Flutter SDK | `https://storage.googleapis.com/flutter_infra_release/releases/stable/windows/flutter_windows_<ver>-stable.zip` | `https://storage.flutter-io.cn/flutter_infra_release/releases/stable/windows/flutter_windows_<ver>-stable.zip` |
| バージョン一覧 | `https://storage.googleapis.com/flutter_infra_release/releases/releases_windows.json` | `https://storage.flutter-io.cn/flutter_infra_release/releases/releases_windows.json` |
| Git | `https://github.com/git-for-windows/git/releases/...` | `https://registry.npmmirror.com/-/binary/git-for-windows/<ver>/MinGit-<ver>-64-bit.zip` |
| JDK 17 | `https://download.java.net/openjdk/jdk17/...` | `https://repo.huaweicloud.com/openjdk/17/openjdk-17_windows-x64_bin.zip` |
| Android cmdline-tools | `https://dl.google.com/android/repository/commandlinetools-win-13114758_latest.zip` | 同一ホスト——`storage.googleapis.com` が不通でも通常は到達できる |
| Android コンポーネント | `https://dl.google.com/android/repository/platform-tools_r37.0.1-win.zip`、`platform-36_r02.zip`、`build-tools_r36_windows.zip` | 同一ホスト |
| Gradle | `https://services.gradle.org/distributions/gradle-<ver>-all.zip` | `https://mirrors.cloud.tencent.com/gradle/gradle-<ver>-all.zip` |
| VS Build Tools | `https://aka.ms/vs/17/release/vs_BuildTools.exe` | 同一ホスト |

**ダウンロードは `curl` ではなく Python の `urllib` を使う。** Windows の Git Bash の curl は
`curl: (6) getaddrinfo() thread failed to start` で頻繁に失敗し、接続性とは無関係である。Python は OS の
ネットワークスタックを通るため確実で、1.8 GB の Flutter を約 167 秒で取得できた。

```python
import urllib.request
UA = {'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64)'}
req = urllib.request.Request(url, headers=UA)   # User-Agent は必須。無いと 403 を返すミラーがある
with urllib.request.urlopen(req, timeout=120) as r, open(dest, 'wb') as f:
    while True:
        b = r.read(1024 * 1024)
        if not b:
            break
        f.write(b)
```

バージョンとアーカイブ名は推測せず一覧から取得する：

```python
import json, urllib.request
d = json.loads(urllib.request.urlopen(
    'https://storage.flutter-io.cn/flutter_infra_release/releases/releases_windows.json', timeout=30).read())
for r in d['releases']:
    if r['hash'] == d['current_release']['stable'] and r['channel'] == 'stable':
        print(d['base_url'] + '/' + r['archive'], r['version'], r.get('dart_sdk_version'))
```

PowerShell がある場合 `System.Net.WebClient` も使えるが、同じく **User-Agent が必須**。
`robocopy` と `Start-Process -Verb RunAs` は制限環境でブロックされることがある——コピーは Bash の `cp -a` に切り替える。

## Step 2 · 展開

Windows 標準の `tar.exe` は Flutter SDK の展開に失敗し、`.NET ZipFile::ExtractToDirectory` もポリシーで弾かれることがある。
Python の `zipfile` が確実：

```python
import zipfile, os, sys
z = zipfile.ZipFile(sys.argv[1])
for i in z.infolist():
    if not i.filename.endswith('/'):
        p = os.path.join(sys.argv[2], i.filename)
        os.makedirs(os.path.dirname(p), exist_ok=True)
        open(p, 'wb').write(z.read(i))
```

容量を空けるために 6 万ファイルのディレクトリを削除してはならない——遅いうえにブロックされ得る。
**`*_broken` にリネームして回避する。**

## Step 3 · 環境変数（ユーザーレベル）

`[Environment]::SetEnvironmentVariable($k, $v, 'User')` — 新しいターミナルで有効になる。

```
JAVA_HOME                       = C:\tools\jdk\jdk-17
ANDROID_HOME / ANDROID_SDK_ROOT = C:\Android
CHROME_EXECUTABLE               = C:\Program Files (x86)\Microsoft\Edge\Application\msedge.exe  (Chrome が無ければ Edge で代用)
PATH += C:\src\flutter\bin; C:\tools\git\cmd; C:\tools\jdk\jdk-17\bin; C:\Android\platform-tools
```

**Google に到達できないネットワークの場合のみ**、ミラーも設定する（詳細は「Google サーバーに接続できないとき」）：

```
PUB_HOSTED_URL           = https://pub.flutter-io.cn
FLUTTER_STORAGE_BASE_URL = https://storage.flutter-io.cn
```

**通常のネットワークではこの 2 つを設定しない**——回避策であって既定値ではなく、設定するとパッケージの
通信が第三者を経由することになる。

## Step 4 · Android SDK とライセンス

`sdkmanager` で platform / build-tools / platform-tools を入れ、**ライセンスは必ず実際に対話形式で承諾する**：

```bash
yes | head -30 | sdkmanager.bat --licenses --sdk_root=C:/Android
# "All SDK package licenses accepted" が出て初めて完了
```

ライセンスのハッシュを手書きしても認識されない（"5 of 7 not accepted" で止まる）。成功すると `C:\Android\licenses` に
7 ファイルが入り、doctor の Android toolchain が緑になる。

## Step 5 · Gradle 配布元の置換

`services.gradle.org` は遅いか到達不能なことが多い。Flutter のテンプレートを書き換えれば以降の新規プロジェクトに自動適用される：

`C:\src\flutter\packages\flutter_tools\templates\app\android.tmpl\gradle\wrapper\gradle-wrapper.properties.tmpl`
```
distributionUrl=https\://mirrors.cloud.tencent.com/gradle/gradle-{{gradleVersion}}-all.zip
```

既存プロジェクトはその `android/gradle/wrapper/gradle-wrapper.properties` を修正する。

> **Flutter を新規インストールすると、このファイルは `services.gradle.org` に戻る。** 再インストールや SDK 修復のたびに
> この修正を再適用すること。忘れると次回の `flutter build apk` が Gradle のダウンロードで止まる。最も忘れやすい手順。

## Step 6 · Visual Studio（Windows デスクトップ向けのみ）

```powershell
Start-Process 'D:\tools\vs\vs_BuildTools.exe' -ArgumentList `
  '--installPath','D:\VS','--quiet','--wait','--norestart','--nocache','--add','Microsoft.VisualStudio.Workload.VCTools','--includeRecommended' -Wait
```

**管理者権限がない場合は必ず失敗する**（終了コード 1、UAC も出ない）。何度も再試行せず、`.bat` を置いてユーザーに
右クリック →「管理者として実行」してもらう。

## Step 7 · 検証

```bash
flutter --version          # 3 秒以内に返る
flutter doctor             # Flutter / Android toolchain / Network resources がチェックマーク
flutter build apk --debug  # end to end
```

## 再インストール / 修復

以下の症状は **SDK 自体の破損**であり、設定ミスではない：

- `flutter.bat: No such file or directory` → `flutter\bin` が消えている。`bin` には `cache\dart-sdk` と
  `cache\flutter_tools.snapshot` も入っており、失うと Dart SDK ごと失う。
- `java -version` が `Failed setting boot class path` → `jdk-17\lib\modules` が欠落（通常約 125 MB）。

どちらもその場では復旧できず、`flutter*.zip` のキャッシュも無い。再ダウンロードする：

1. 一覧からバージョンを取得（Step 1）。
2. 壊れたディレクトリを `os.rename` で `*_broken` にする——まだ削除しない。
3. Python でダウンロードして展開（Step 1–2）。
4. `flutter\bin\flutter.bat` と `jdk-17\bin\java.exe` の存在を**先に確認**してから次へ進む。
5. Gradle の修正を再適用（Step 5）——新規インストールで消えるため。
6. `flutter --version` を実行。初回は依存のダウンロードで約 1 分かかるが正常。
7. 問題ないと確認できてから `*_broken` の残骸を削除する。

上の 1〜7 が復旧の手順順です。削除と再インストールを一度に済ませる形にまとめず、手順どおりに進めてください。4 番が「入った」か「解凍途中で壊れている」かを分ける確認点です。

## ツールチェーンをシステムドライブから移す

C: が逼迫したら、D:/E: に移す。方法は二つ。どちらかを選ぶ。

### 方法 A — junction（元のパスを有効に保つ）

元のパスに **junction** を残す。これで PATH と Flutter 内部の絶対パスはすべて有効なまま、実体は別ドライブに置ける。

1. **先にコピー、検証、その後に削除。** Bash の `cp -a` を使う（`robocopy` はブロックされやすく、しかも無言で失敗する）。
2. 元を触る前にファイル数を突き合わせる：
   ```bash
   echo "src=$(find /c/src -type f | wc -l)  dst=$(find /d/src -type f | wc -l)"
   ```
   完全一致してから次へ進む。
3. 数万ファイルの削除は一括削除ガードでブロックされることがある（セッションあたり 50 ファイル制限の環境もある）。その場合は**ユーザーに `.bat` を渡す**——`rmdir /S /Q` + `mklink /J`——ここで往復して時間を浪費しない。
4. リネームは削除ではない：ディレクトリを退避させるには `mv` で `_junk_*` にし、掃除は `.bat` にまとめる。

`mklink /J <旧パス> <新パス>` に管理者権限は不要。**エージェント自身が使用中の作業/キャッシュディレクトリは絶対に移さない**（セッションが切れる）。

### 方法 B — PATH とツールチェーン変数だけを付け替える（コピーなし）

PATH とツールチェーン変数の向き先だけを変える——何もコピーせず、一括コピー / 一括削除が拒否される状況でも使える。以下を**すべて**設定しないと、ビルドが数 GB を C: に書き戻す：

```powershell
# PUB_CACHE / GRADLE_USER_HOME / ANDROID_USER_HOME の既定は C:\Users\<u>\...
$k = [Microsoft.Win32.Registry]::CurrentUser.OpenSubKey('Environment', $true)
$p = [string]$k.GetValue('Path')                      # 第3引数は渡さない（落とし穴を参照）
$p = $p.Replace('C:\src\flutter\bin','D:\src\flutter\bin')
$k.SetValue('Path', $p, [Microsoft.Win32.RegistryValueKind]::ExpandString)
$k.SetValue('JAVA_HOME','D:\tools\jdk\jdk-17',[Microsoft.Win32.RegistryValueKind]::String)
$k.Close()
```

- `[Environment]::SetEnvironmentVariable($k,$v,'User')`（Step 3）は**サンドボックス化されたシェルではしばしばブロックされる**（`cmd.exe cannot be used from the PowerShell tool`）。`Set-ItemProperty ... -Type ExpandString` も同様。レジストリを直接書けば常に動作し、管理者権限も不要。
- `$k.GetValue('Path','',$true)` は `MethodException` を投げる——第3引数は `bool` ではなく `RegistryValueOptions` を要求する。`GetValue('Path')` か `Get-ItemProperty HKCU:\Environment` を使う。
- `Path` には必ず `RegistryValueKind::ExpandString` を指定する。さもないと `%VAR%` 参照が固定化される。
- **既存 PATH は文字列置換で更新し、丸ごと作り直さない。** ユーザー PATH には非 ASCII の項目があり、雑な書き換えは文字化けや欠落を招く。
- 先に旧値をファイルへバックアップする。ロールバックはコマンド一行で済む。
- **レジストリを直接書いても `WM_SETTINGCHANGE` はブロードキャストされない**——設定は完璧に見えるのに、ユーザーのマシンでは失敗し続ける。`SetValue` / `Set-ItemProperty` では稼働中の `explorer.exe` が古い環境ブロックを 持ち続けるため、スタートメニューから開いた CMD は `'flutter' は内部コマンドまたは外部コマンド…` を報告し続ける。一方 `HKCU:\Environment\Path` は既に正しく、環境を再構築したシェルでは `where flutter` が解決できる。**PATH に再び手を入れる前に、レジストリを読み戻して確認すること**——原因は値ではなく陳腐化した環境ブロックにある。更新方法は、ログオフ / ログオン、Windows エクスプローラーの再起動（タスクマネージャー → Windows エクスプローラー → 再起動）、または `setx <無害な変数> <値>`（setx 自身がブロードキャストし、PATH には触れない）。サンドボックスでは `Add-Type` による P/Invoke `SendMessageTimeout` は通常ブロックされる。

### PATH の落とし穴（両方の方法に共通）

- **PATH は正しく見えるのに `flutter` が見つからない。** PATH に項目があることは、SDK がそこにある証明ではない——解凍が中断された放置インストールは** `bin\` を持たない木**を残し、項目は空を指す。それでも `flutter doctor` は動く。絶対パスで起動しているからだ。実ファイルを必ず検証すること（`Test-Path ...\bin\flutter.bat`、`...\bin\cache\dart-sdk\bin\dart.exe`）。また PATH 上に `platform-tools` が二重に現れないか注意する（doctor は `Multiple adb binaries found` を出す）。
- **doctor が "not on your path" と言うが `where` は正しく解決する。** プロセス環境に **`Path` と `PATH` が同時に存在する**（大文字小文字だけが異なる2つの項目）場合、Windows の大文字小文字を区別しない検索は一方に当たり、Dart の `Platform.environment['PATH']` はもう一方に当たる。その結果 doctor は `The flutter binary is not on your path` と警告するが、`where.exe flutter` は正しいパスを表示する。2行の Dart プローブで `containsKey("PATH")` と `containsKey("Path")` を出力して確認する。両方 true なら その警告は呼び出し側シェルの産物である——**真新しいターミナル**で検証してから手を動かすこと。
## Google サーバーに接続できないとき

Flutter の既定のダウンロード先——`storage.googleapis.com`、`pub.dev`、`dl.google.com`——はいずれも Google の
インフラ上にあり、Google を遮断するネットワークでは複数の手順が同時に壊れる：SDK のダウンロード、`pub get`、
Gradle 配布物の取得。以下を上から順に確認する。

### 1. 表示されるエラー

| 場所 | メッセージ | 実際の意味 |
|---|---|---|
| `flutter doctor` | `Network resources` が全行 `X`、"A network error occurred while checking https://pub.dev" | Flutter が自身のパッケージレジストリに到達できない |
| `flutter pub get` | `Got socket error trying to find package <名前> at https://pub.dev` | pub エンドポイントに到達不能（DNS または TCP 層） |
| 任意のダウンロード | `SocketException: Failed host lookup: 'storage.googleapis.com'` | DNS 解決の失敗——名前が IP に解決されていない |
| 任意のダウンロード | `curl: (6) getaddrinfo() thread failed to start` | このマシンの Git Bash の curl 自体が壊れている。**ネットワークではなくツールの問題** |
| 初回 `flutter --version` | `Downloading Dart SDK...` で停止、または `Unable to update Dart SDK. Retrying...` | Google ストレージからの Dart SDK 取得に失敗 |
| `flutter build apk` | `Downloading https://services.gradle.org/distributions/...` で停止 | Gradle 自身の配布物に到達できない |
| Android SDK ツール | `Warning: Failed to download ...` の後 `Done` だがパッケージ欠落 | `dl.google.com` が部分的に遮断 |
| VS Build Tools | インストーラが 0% で停止 | Microsoft の CDN が遅いだけで、**Google の問題ではない** |

### 2. 原因（よくある 5 パターン）

**A. DNS 解決の失敗。** `storage.googleapis.com` や `pub.dev` が解決できない、または誤った／ブラックホール化された
IP に解決される。単なるタイムアウトではなく "Failed host lookup" と出るのが特徴。

**B. 直接接続の遮断。** 名前は解決するが TCP :443 が確立しない。ダウンロードは 0% で止まり、やがてタイムアウトする。

**C. プロキシ設定の誤り。** プロキシ配下なのにこれらのツールに設定していない、あるいは設定したプロキシ自体が
これらのリクエストを処理できない。Flutter のツールは `HTTP_PROXY`/`HTTPS_PROXY` を尊重するが、すべてのツールが
そうではない——ブラウザで見られることは、ツールチェーンが出られる証明にはならない。

**D. セキュリティソフト / 企業ファイアウォール。** アンチウイルス、「ネットワーク高速化」系ツール、企業ゲートウェイが
TLS 接続を検査して切断する。断続的で、特定のホストにだけ起きることが多い。

**E. ミラーが無効。** すでにミラーへ切り替えたが、そのミラーが落ちている、帯域制限、または 403 を返す
（清華ミラーは大容量 GET で 403 を返すことで知られる）。

### 3. 原因別の対処

**A（DNS 失敗）の場合** —— Google の名前を解決しようとするのをやめ、ミラーへ切り替える：

1. まず DNS であることを確定する：
   `python -c "import socket;print(socket.gethostbyname('storage.googleapis.com'))"` —— ここで例外が出れば DNS が原因。
2. ミラーの環境変数を設定し、**新しい**ターミナルを開く：
   ```
   PUB_HOSTED_URL           = https://pub.flutter-io.cn
   FLUTTER_STORAGE_BASE_URL = https://storage.flutter-io.cn
   ```
3. `storage.flutter-io.cn` から Flutter SDK をダウンロードする（Step 1 の表）。
4. `flutter doctor` を再実行する。

**B（直接接続の遮断）の場合** —— `pub get` には依然として経路が必要なので、ミラーだけでは足りないことがある：

1. 短いタイムアウトで到達性を確認する。例：`curl --max-time 10 -I https://pub.flutter-io.cn`
   （Git Bash の curl が壊れているなら PowerShell を使う）。
2. A と同じミラー環境変数を設定する。
3. ミラー**も**遮断されている場合はプロキシが必要——C を参照。

**C（プロキシ）の場合** —— プロキシの有無と動作を確認する：

1. 現在の設定を調べる：
   ```
   env | grep -i proxy
   netsh winhttp show proxy
   ```
2. プロキシがあり動作するなら、ツール用に設定する：
   ```
   HTTPS_PROXY = http://<ホスト>:<ポート>
   HTTP_PROXY  = http://<ホスト>:<ポート>
   ```
3. プロキシが設定されているが**死んでいる**場合は、切り分けの前に外す——残存プロキシは Google とは無関係な
   ネットワークエラーを `flutter doctor` に報告させる：
   ```
   unset HTTP_PROXY HTTPS_PROXY
   ```
4. プロキシが唯一の出口なら `pub get` では残し、`flutter doctor` の実行時には外す。

**D（セキュリティソフト）の場合** —— 断続的なため最も見つけにくい：

1. アンチウイルス／「高速化」系ツールを一時的に無効化し、単発のダウンロードを再試行する。
2. それで成功するなら、`storage.flutter-io.cn`、`pub.flutter-io.cn`、`dl.google.com` と Flutter/JDK の
   インストール先を許可リストに追加する。
3. 途中で切れても中途半端な SDK が残らないよう、まずファイルへダウンロードし、ローカルファイルからインストールする。

**E（ミラーが無効）の場合** —— 切り替える前に検証する：

1. 1.8 GB のダウンロードを始めるのではなく、小さなリクエストでミラーを試す（`-I`、`--max-time 10`）。
2. 実績のある代替：Flutter → `storage.flutter-io.cn`；Gradle → `mirrors.cloud.tencent.com`；
   Git → `registry.npmmirror.com`；JDK → `repo.huaweicloud.com`。Maven Central と Google Maven は他の Google ホストが
   不通でも通ることが多いので、**Maven リポジトリを不要に差し替えない**。
3. 大容量ファイルで清華ミラーを避ける——GET で 403 を返す。

### 4. ミラー早見表

| ツール | 設定場所 | 値 |
|---|---|---|
| Dart / pub | 環境変数 `PUB_HOSTED_URL` | `https://pub.flutter-io.cn` |
| Flutter SDK | 環境変数 `FLUTTER_STORAGE_BASE_URL` | `https://storage.flutter-io.cn` |
| Gradle | `gradle-wrapper.properties` の `distributionUrl` | `https://mirrors.cloud.tencent.com/gradle/gradle-<ver>-all.zip` |
| Android SDK | `ANDROID_HOME` + リポジトリ設定 | `dl.google.com` は通常通る。ミラー不要 |

### 5. 修正の確認

```bash
flutter doctor      # Network resources がエラーを出さなくなる
flutter pub get     # パッケージを解決でき、socket error を出さない
flutter build apk --debug
```

ミラーを設定しプロキシも正しいのに `pub get` が失敗するなら、ミラーへの経路が本当に遮断されている。
手を変え品を変え試すのはやめ、そのまま伝える。同じコマンドを繰り返さない。

## 切り分け早見表

| 症状 | 原因 | 対処 |
|---|---|---|
| `flutter.bat: No such file or directory` | `flutter\bin` が削除され Dart SDK ごと消失 | SDK を再インストール（「再インストール / 修復」参照） |
| `java -version` が `Failed setting boot class path` | `jdk-17\lib\modules` の欠落 | JDK を再インストール |
| `flutter` のコマンドが十数分フリーズ | `bin/cache/flutter_tools.snapshot` が `.old1` にリネーム（再コンパイル中断） | `.old1` を `flutter_tools.snapshot` に戻し、`flutter.bat.lock` を削除。以後 `--version` は約 3 秒で返る |
| `flutter build apk` が Gradle のダウンロードで止まる | SDK 再インストールでテンプレートが `services.gradle.org` に戻った | ミラー修正を再適用（Step 5） |
| sdkmanager が "Java version 17 or higher is required" | ファーウェイ OpenJDK 17 は `openjdk version "17"` を出力し、`tokens=1,2 delims=.` の解釈で `17` < `170` になる | `sdkmanager.bat` の解析ループに `if "%%c" == "" set versionWithQ=%%b0` を追加。`SKIP_JDK_VERSION_CHECK` は不要になる |
| ライセンスが "5 of 7 not accepted" で止まる | 手書きハッシュは認識されない | 実際に対話形式で承諾する（Step 4） |
| `pub get` が `Got socket error trying to find package ...` | pub への経路がない | 「Google サーバーに接続できないとき」の第 3 節を参照。`PUB_CACHE`/`GRADLE_USER_HOME` を移しても**効果はない** |
| doctor の 3 項目が crashed 表示 | `reg.exe` / `wmic.exe` がポリシーでブロック | 通常のマシンでは起きない。実障害ではない |
| doctor の Network resources が全滅 | プロキシまたは経路の遮断 | まず死んだプロキシを外し、次に「Google サーバーに接続できないとき」を参照 |

## 環境による干渉（実際の障害ではない）

- `reg.exe` / `wmic.exe` がブロック → doctor の "Connected device"、"Windows Version"、"Visual Studio" が
  `Flutter failed to run "reg query ..."` でクラッシュする。無害。
- `adb` デーモンが起動しない（`could not read ok from ADB Server`）→ デバイス未接続時によくある。`flutter build apk` には影響しない。
- 制限環境では `pub get` / `build apk` が最後まで通らないことがある。はっきり伝え、ユーザー自身のマシンで実行してもらう。
  手を変え品を変え再試行しない。

## ネットワークなしでツールチェーンを検証する

- **`flutter create` / `pub get` / `build apk` の最初の関門はツールの自己ブートストラップであり、あなたのプロジェクトではない。**
  `bin\internal\shared.bat` は `bin\cache\flutter_tools.stamp` を `git rev-parse HEAD` から作った compile key と比較し、
  一致しないと `packages\flutter_tools` で `dart pub upgrade` を再実行し、`Building flutter tool...` を表示した後
  `Running pub upgrade...` で止まる。解凍直後の SDK で stamp が空リビジョン（ファイル内容 `":"`、バイト列
  `22 3A 22 0D 0A`）の場合、`create` / `pub get` が動くには**一度 pub ミラーに到達する必要がある**。
  これはネットワーク要件であり、設定ミスではない。
  - 「正しい」stamp を手書きしても再ビルドは**確実には**回避できない：`shared.bat` の 75 行目はネストした
    `FOR /f` サブシェルでリビジョンを取得するため、ネストした `cmd.exe` を禁じるポリシー下ではリビジョンが常に空になり、
    手書きの値は決して一致しない。実験した場合は、**stamp を元のバイト列どおりに復元**すること——
    他人の SDK に未検証の変更を残してはならない。
- **`--no-pub` でネットワークなしにツールチェーン全体を検証する。** `flutter create --no-pub <名前>` は
  プロジェクトの骨組み（`lib/`、`android/`、`web/`、`pubspec.yaml`）を丸ごと生成し、依存解決には一切触れない。
  その後 `android\local.properties` で SDK パスを、`android\gradle\wrapper\gradle-wrapper.properties` で
  ミラーを確認する。pub に到達できないときの最速の実検証。
- **Dart 単体を検証する場合：** `dart compile exe hello.dart -o hello.exe` を実行し、生成物を動かす。完全オフラインで、
  pub と Gradle から独立している。
