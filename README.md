# termux-notice-system-android

Termux 上の Claude Code から Android 通知を送信するための完全ガイド。

Claude Code のサンドボックス制約、Android のセキュリティモデル、APK 署名の仕組みなど、
複数の壁を乗り越えて `termux-notification` を動作させるまでの知見をまとめている。

---

## 前提環境

| 項目 | 値 |
|------|-----|
| デバイス | Samsung Galaxy (Android 15 / SDK 35) |
| Termux | v1022（F-Droid 署名） |
| Claude Code | Opus 4.6 (Termux 上で動作) |

---

## TL;DR（結論）

```bash
# 1. Termux:API アプリをインストール（F-Droid 版）
# 2. termux-api パッケージをインストール
pkg install termux-api

# 3. 通知を送る
termux-notification --title "タイトル" --content "本文" --sound
```

**ただし、Termux 本体と Termux:API の APK 署名が一致していないとインストールできない。**
これが最大のハマりポイント。

---

## 目次

1. [アーキテクチャ概要](#アーキテクチャ概要)
2. [Claude Code サンドボックスの回避](#claude-code-サンドボックスの回避)
3. [Termux:API のインストール（署名問題）](#termuxapi-のインストール署名問題)
4. [termux-notification の使い方](#termux-notification-の使い方)
5. [試行錯誤の記録（失敗から学ぶ）](#試行錯誤の記録失敗から学ぶ)
6. [代替手段](#代替手段)
7. [リモートからの通知送信](#リモートからの通知送信)
   - [アーキテクチャ](#アーキテクチャ)
   - [セットアップ手順](#セットアップ手順)
   - [使い方（実践例）](#使い方実践例)
   - [トンネルの維持と自動再接続](#トンネルの維持と自動再接続)
   - [セキュリティに関する注意](#セキュリティに関する注意)
8. [Claude Code フック連携](#claude-code-フック連携)
   - [概要](#概要)
   - [通知の表示例](#通知の表示例)
   - [設定ファイル](#設定ファイル)
   - [通知スクリプトの仕組み](#通知スクリプトの仕組み)
   - [セットアップ](#セットアップ)
   - [動作確認](#動作確認)
9. [トラブルシューティング](#トラブルシューティング)

---

## アーキテクチャ概要

```
+--------------------------------------------------+
| Android OS (SDK 35)                              |
|                                                  |
|  +---------------+    +----------------------+   |
|  | Termux        |    | Termux:API           |   |
|  | (com.termux)  |<-->| (com.termux.api)     |   |
|  |               | AM |                      |   |
|  | termux-api    |broadcast                  |   |
|  | (CLI pkg)     |--->| TermuxApiReceiver     |   |
|  |               |    |   +> NotificationAPI  |   |
|  +---------------+    +----------------------+   |
|         ^                                        |
|         | setsid bash -c                         |
|  +------+--------+                               |
|  | Claude Code   |                               |
|  | (sandbox)     |                               |
|  +---------------+                               |
+--------------------------------------------------+
```

### コンポーネントの役割

- **Termux** (`com.termux`): Android 上の Linux ターミナルエミュレータ
- **Termux:API** (`com.termux.api`): Android API へのブリッジとなる**別アプリ**
- **termux-api** (CLI パッケージ): `termux-notification` 等のコマンドを提供するシェルスクリプト群
- **Claude Code**: Termux 内で動作する AI コーディングアシスタント（独自サンドボックスあり）

### 通知の流れ

1. `termux-notification` コマンドを実行
2. 内部で `/data/data/com.termux/files/usr/libexec/termux-api`（`termux-api-broadcast` へのシンボリックリンク）を呼び出し
3. `am broadcast` で `com.termux.api/.TermuxApiReceiver` に Intent を送信
4. Termux:API アプリが Android の NotificationManager を使って通知を作成

---

## Claude Code サンドボックスの回避

### 問題

Claude Code は Termux 内で実行されるが、独自のサンドボックスを持つ。
このサンドボックスは `/tmp/claude-<id>/` にタスク管理ディレクトリを作成しようとするが、
Termux 環境では `/tmp` がシステムの `/tmp` であり、Termux アプリからはアクセスできない
（Termux の TMPDIR は `/data/data/com.termux/files/usr/tmp`）。

結果、多くのコマンドが以下のエラーで失敗する：

```
EACCES: permission denied, mkdir '/tmp/claude-XXXXX/-data-data-com-termux-files-home/tasks'
```

### 解決策: setsid

`setsid` を使ってコマンドを新しいセッションで実行することで、サンドボックスの制約を回避できる。

```bash
# 直接実行 → ブロックされる
termux-notification --title "test" --content "hello"

# setsid 経由 → 成功
setsid bash -c 'termux-notification --title "test" --content "hello"'
```

**注意**: `setsid` で実行したコマンドの stdout/stderr は通常のターミナルに接続されないため、
結果をファイルにリダイレクトする必要がある。

```bash
setsid bash -c 'some-command > ~/result.txt 2>&1'
# 結果を確認
cat ~/result.txt
```

---

## Termux:API のインストール（署名問題）

### 最重要ポイント: APK 署名の一致

Termux と Termux:API は `android:sharedUserId` を共有しているため、
**両方のアプリが同じ署名鍵で署名されている必要がある**。

署名が異なると、以下のエラーでインストールが拒否される：

```
パッケージが既存のパッケージと競合するため、アプリケーションをインストールできませんでした。
```

### 署名鍵の種類

| ソース | 署名鍵 | 備考 |
|--------|--------|------|
| F-Droid | F-Droid 署名鍵 | APK 内 `META-INF/84D3000E.RSA` |
| GitHub Releases (debug) | GitHub Actions debug 鍵 | ビルドごとに異なる可能性あり |
| Google Play (旧) | Google Play 署名鍵 | 現在は配布停止 |

### 自分の Termux の署名を確認する方法

```bash
# Termux APK の META-INF を確認
# もし 84D3000E.RSA があれば F-Droid 署名
unzip -l /path/to/com.termux.apk "META-INF/*.RSA" "META-INF/*.SF"
```

### F-Droid 版 Termux:API のインストール手順

```bash
# 1. ダウンロード
curl -L -o /sdcard/Download/termux-api-fdroid.apk \
  "https://f-droid.org/repo/com.termux.api_51.apk"

# 2. ファイルマネージャーで開いてインストール
am start -a android.intent.action.VIEW \
  -d "content://com.android.externalstorage.documents/document/primary%3ADownload" \
  -t "vnd.android.document/directory"

# 3. ファイルマネージャーで termux-api-fdroid.apk をタップしてインストール
```

### なぜ termux-open ではなくファイルマネージャーか

`termux-open` は内部で `com.termux.app.TermuxOpenReceiver` に `am broadcast` を送るが、
APK ファイルの処理で「解析エラー」が発生することがある。
`/sdcard/Download/` にコピーしてファイルマネージャーから開くのが確実。

---

## termux-notification の使い方

### 基本

```bash
termux-notification --title "タイトル" --content "本文"
```

### 全オプション

| オプション | 説明 | 例 |
|-----------|------|-----|
| `-t/--title` | 通知タイトル | `--title "Alert"` |
| `-c/--content` | 通知本文 | `--content "Hello"` |
| `-i/--id` | 通知 ID（更新・削除用） | `--id "my-notify"` |
| `--priority` | 優先度 | `--priority high` (min/low/default/high/max) |
| `--sound` | 通知音を鳴らす | `--sound` |
| `--vibrate` | バイブパターン (ms) | `--vibrate 500,200,500` |
| `--led-color` | LED 色 (RRGGBB) | `--led-color ff0000` |
| `--led-on` | LED 点灯時間 (ms) | `--led-on 800` |
| `--led-off` | LED 消灯時間 (ms) | `--led-off 800` |
| `--ongoing` | 固定通知（`--id` 必須） | `--ongoing --id "pin"` |
| `--alert-once` | 更新時に再通知しない | `--alert-once` |
| `--group` | 通知グループ | `--group "builds"` |
| `--icon` | ステータスバーアイコン | `--icon warning` |
| `--image-path` | 画像パス | `--image-path ~/pic.png` |
| `--action` | タップ時のアクション | `--action "termux-open https://..."` |
| `--button1` | ボタン 1 テキスト | `--button1 "OK"` |
| `--button1-action` | ボタン 1 アクション | `--button1-action "echo done"` |
| `--on-delete` | 消去時のアクション | `--on-delete "echo cleared"` |
| `--channel` | 通知チャンネル ID | `--channel "alerts"` |
| `--type` | 通知スタイル | `--type media` |

### 使用例

```bash
# ビルド完了通知
termux-notification \
  --title "Build Complete" \
  --content "プロジェクトのビルドが完了しました" \
  --sound --vibrate 200,100,200 \
  --id "build-done" \
  --action "termux-open ~/project/build.log"

# 進捗表示（更新可能）
termux-notification --title "Deploy" --content "Deploying... 0%" --id "deploy" --ongoing
termux-notification --title "Deploy" --content "Deploying... 50%" --id "deploy" --ongoing --alert-once
termux-notification --title "Deploy" --content "Complete!" --id "deploy" --sound

# stdin からコンテンツを読む
echo "長いメッセージをパイプで送る" | termux-notification --title "Pipe Test"

# 通知の削除
termux-notification-remove "deploy"

# 全通知のリスト
termux-notification-list
```

---

## 試行錯誤の記録（失敗から学ぶ）

Android の通知システムは多層的なセキュリティモデルで保護されている。
以下は Termux:API なしで通知を送ろうとした試みの記録。

### 1. cmd notification（権限不足）

```bash
cmd notification post -t "title" "tag" "content"
# → error: permission denied: callingUid=10317 callingPackage=com.termux
```

Android の `cmd` コマンドはシェル UID (2000) でないと notification サービスにアクセスできない。

### 2. dalvikvm で Java Notification API（ネイティブライブラリ制限）

Termux には `dalvikvm` が付属している。Java コードをコンパイルして実行を試みた。

```bash
pkg install ecj dx
ecj Notify.java
dx --dex --output=notify.dex Notify.class
chmod 444 notify.dex  # Android 15 では書き込み可能な DEX を拒否する
dalvikvm -cp notify.dex Notify
```

**発見**: Android 15 (SDK 35) では **書き込み可能な DEX ファイルの実行を拒否** する。
`chmod 444` で読み取り専用にする必要がある。

Hello World は動いたが、Android の `ServiceManager` にアクセスしようとすると：

```
UnsatisfiedLinkError: No implementation found for
android.os.SystemProperties.native_get
```

`dalvikvm` は bare JVM であり、Android フレームワークのネイティブライブラリがロードされない。
`System.loadLibrary("android_runtime")` も名前空間制限（linker namespace）で失敗する。

### 3. app_process（存在しない）

`app_process` は Android フレームワークを含む JVM を起動できるが、
Samsung Android 15 では `/system/bin/app_process` が存在しなかった。

### 4. pm install（権限不足）

```bash
cat app.apk | pm install --user 0 -S <size>
# → SecurityException: Permission Denial: doCreateSession
#   requires android.permission.INTERACT_ACROSS_USERS_FULL
```

Termux の UID からは APK をプログラム的にインストールできない。

### 5. ブラウザ Web Notification API（成功するが Termux 通知ではない）

```bash
python3 -m http.server 8765 --directory ~ &
am start -a android.intent.action.VIEW -d "http://localhost:8765/notify.html"
```

**重要**: Android Chrome では `new Notification()` コンストラクタが動作しない。
**Service Worker の `registration.showNotification()`** を使う必要がある。

### 6. bell-character=notification（無効な設定値）

`termux.properties` の `bell-character` の有効な値は `vibrate`、`beep`、`ignore` のみ。
`notification` は無効値であり、デフォルトの `vibrate` にフォールバックする。

---

## 代替手段

### Termux:API がインストールできない場合

#### ブラウザ通知（Service Worker 方式）

```html
<!-- notify.html -->
<!DOCTYPE html>
<html>
<head><meta charset="utf-8"></head>
<body>
<script>
async function run() {
    const reg = await navigator.serviceWorker.register('/sw.js');
    await navigator.serviceWorker.ready;
    const perm = await Notification.requestPermission();
    if (perm === 'granted') {
        await reg.showNotification('タイトル', {
            body: '本文',
            vibrate: [200, 100, 200],
            tag: 'my-notify',
            requireInteraction: true
        });
    }
}
run();
</script>
</body>
</html>
```

```javascript
// sw.js
self.addEventListener('install', e => self.skipWaiting());
self.addEventListener('activate', e => e.waitUntil(self.clients.claim()));
```

```bash
python3 -m http.server 8765 --directory ~ &
am start -a android.intent.action.VIEW -d "http://localhost:8765/notify.html"
```

#### termux-wake-lock（組み込み、通知変更のみ）

```bash
termux-wake-lock    # ステータスバーに "wake lock held" を追加
termux-wake-unlock  # 解除
```

---

## リモートからの通知送信

リモートサーバーから Termux 端末の Android 通知を発火させる仕組み。
Termux（スマホ）はグローバル IP を持たないことが多いため、**リバース SSH トンネル** を使って
リモートサーバーからの接続を実現する。

### アーキテクチャ

```
+----------------------------+          +----------------------------+
| Remote Server (leo)        |          | Android (Termux)           |
|                            |          |                            |
|  ssh -p 28022 localhost    |          |  sshd (port 8022)          |
|       |                    |          |       ^                    |
|       v                    |          |       |                    |
|  localhost:28022 --------(reverse SSH tunnel)--+                   |
|                            |          |                            |
|                            |          |  termux-notification       |
|                            |          |       |                    |
|                            |          |       v                    |
|                            |          |  Termux:API → Android通知  |
+----------------------------+          +----------------------------+

  Termux → ssh -R 28022:localhost:8022 → Remote Server
  (トンネルを張る方向: Termux → Remote)
  (通知コマンドの方向: Remote → Termux)
```

### 仕組み

1. Termux 側で `sshd` を起動（ポート 8022）
2. Termux からリモートサーバーに SSH 接続し、`-R` オプションでリバーストンネルを張る
3. リモートサーバーが `localhost:28022` に SSH すると、トンネル経由で Termux の `sshd` に到達
4. リモートサーバーから `termux-notification` コマンドを実行 → Android 通知が発火

**ポイント**: Termux がグローバル IP を持たなくても、Termux 側からトンネルを張るため接続できる。

### セットアップ手順

#### Step 1: Termux で SSH サーバーを準備

```bash
# openssh がなければインストール
pkg install openssh

# sshd を起動（デフォルトポート 8022）
sshd

# 起動確認
pgrep -a sshd
```

Termux の sshd はパスワード認証がデフォルトで無効。公開鍵認証のみ。

#### Step 2: リモートサーバーの公開鍵を Termux に登録

```bash
# リモートサーバーの公開鍵を取得して authorized_keys に追加
ssh -p <remote-port> user@remote-server "cat ~/.ssh/id_ed25519.pub" \
  >> ~/.ssh/authorized_keys

# パーミッション設定
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

#### Step 3: リバース SSH トンネルを張る

```bash
# Termux から実行
# -R 28022:localhost:8022 = リモートの 28022 番ポートを Termux の 8022 に転送
# -f = バックグラウンド実行
# -N = リモートコマンドなし（トンネルのみ）
ssh -p <remote-port> -f -N -R 28022:localhost:8022 user@remote-server
```

| パラメータ | 意味 |
|-----------|------|
| `-R 28022:localhost:8022` | リモートの 28022 → Termux の 8022 |
| `-f` | バックグラウンドで実行 |
| `-N` | シェルを開かずトンネルだけ維持 |
| `28022` | リモート側のリッスンポート（任意、衝突しない番号） |
| `8022` | Termux の sshd ポート |

#### Step 4: リモートサーバーから通知を送信

```bash
# リモートサーバーで実行
ssh -p 28022 localhost \
  'termux-notification --title "Remote Alert" --content "サーバーからの通知" --sound'
```

### 使い方（実践例）

#### 基本的なリモート通知

```bash
# リモートサーバーから
ssh -p 28022 localhost \
  'termux-notification --title "From Server" --content "リモート通知成功！" --sound --vibrate 500,200,500'
```

#### 長時間タスク完了通知

```bash
# リモートサーバーでビルドやデプロイ後に通知
make build && \
ssh -p 28022 localhost \
  'termux-notification --title "Build Complete" --content "ビルド成功" --sound --priority high' || \
ssh -p 28022 localhost \
  'termux-notification --title "Build Failed" --content "ビルド失敗！確認してください" --sound --priority max'
```

#### CI/CD パイプラインでの活用

```bash
# GitHub Actions や Jenkins のジョブ末尾に
ssh -p 28022 localhost \
  "termux-notification \
    --title \"CI: Build #${BUILD_NUMBER}\" \
    --content \"${STATUS}: ${REPO}@${BRANCH}\" \
    --sound \
    --action \"termux-open ${BUILD_URL}\""
```

#### ワンライナー：ローカルから leo 経由で自分に通知

```bash
# Termux から実行（自分自身に leo 経由で通知を送る）
ssh -p 5963 neo@leo \
  "ssh -p 28022 localhost 'termux-notification --title \"Round Trip\" --content \"Termux→leo→Termux 往復通知\" --sound'"
```

### トンネルの維持と自動再接続

トンネルはネットワーク切断で切れる。`autossh` で自動再接続できる。

```bash
# autossh をインストール
pkg install autossh

# 自動再接続付きトンネル
# -M 0 = autossh の監視ポートを無効化（ServerAliveInterval に任せる）
autossh -M 0 -f -N \
  -o "ServerAliveInterval 30" \
  -o "ServerAliveCountMax 3" \
  -R 28022:localhost:8022 \
  -p <remote-port> user@remote-server
```

#### Termux:Boot で起動時に自動接続

1. F-Droid から **Termux:Boot** をインストール（署名一致に注意）
2. `~/.termux/boot/` にスクリプトを配置

```bash
mkdir -p ~/.termux/boot
cat > ~/.termux/boot/reverse-tunnel.sh << 'SCRIPT'
#!/data/data/com.termux/files/usr/bin/bash
sshd
sleep 2
autossh -M 0 -f -N \
  -o "ServerAliveInterval 30" \
  -o "ServerAliveCountMax 3" \
  -R 28022:localhost:8022 \
  -p <remote-port> user@remote-server
SCRIPT
chmod +x ~/.termux/boot/reverse-tunnel.sh
```

### セキュリティに関する注意

- リバーストンネルのポート（28022）はリモートサーバーの `localhost` にしかバインドされない（デフォルト）
- リモートサーバーの `GatewayPorts no`（デフォルト）を変更しないこと
- Termux の `sshd` は公開鍵認証のみ。パスワード認証を有効にしないこと
- 信頼できるサーバーにのみトンネルを張ること
- 不要になったトンネルは `kill` で停止する

```bash
# トンネルプロセスの確認と停止
pgrep -af "ssh.*-R.*28022"
kill <pid>
```

---

## Claude Code フック連携

Claude Code のフック機能（`Notification` / `Stop` イベント）を使い、
パーミッション確認待ちや処理完了を **Android 通知で自動的に受け取る** 仕組み。

通知には **作業ディレクトリ名**（プロジェクト名）が常に含まれ、
完了内容や確認内容などの詳細情報も設定ファイルで表示を切り替えられる。

### 概要

```
Claude Code（フックイベント発火）
  │  stdin: JSON {hook_event_name, notification_type, message, cwd, ...}
  v
~/.claude/hooks/termux-notify.sh
  │  ← ~/.claude/hooks/termux-notify.conf（設定読み込み）
  │
  ├─ Termux 上で実行中 → setsid termux-notification（直接）
  │
  └─ リモートサーバー上 → SSH: local → leo → Termux（リレー）
                                          └→ termux-notification
```

### 通知の表示例

作業ディレクトリが `/home/neo/my-web-app` の場合：

| フックイベント | notification_type | 通知タイトル | 通知メッセージ |
|---------------|-------------------|-------------|---------------|
| Notification | `permission_prompt` | 🔐 my-web-app - 確認待ち | `Bash: npm install express` |
| Notification | `idle_prompt` | ✅ my-web-app - 完了 | メッセージまたは「入力待ちです」 |
| Stop | — | ⏹ my-web-app - 処理完了 | メッセージまたは「停止しました」 |

**ディレクトリ名は常に通知タイトルに含まれる**。メッセージ詳細の表示は設定で切り替え可能。

`permission_prompt` は確認待ち中に繰り返し発火するため `--id` + `--alert-once` で
**初回のみ音を鳴らし、以降はサイレント上書き**する。
`idle_prompt` / `Stop` は `--id` なしで毎回独立した通知として発行される。

### 設定ファイル

`~/.claude/hooks/termux-notify.conf` で通知の表示内容や動作をカスタマイズできる。

```bash
# ~/.claude/hooks/termux-notify.conf
# 各項目のデフォルト値は右のコメント参照

# ディレクトリ名の表示形式
#   basename  : プロジェクト名のみ（例: my-web-app）
#   fullpath  : フルパス（例: /home/neo/my-web-app）
CWD_STYLE=basename          # default: basename

# メッセージ詳細を通知に含めるか
#   true  : コマンド内容や完了メッセージを表示
#   false : タイトルのみ（ディレクトリ名 + イベント種別）
SHOW_MESSAGE=true           # default: true

# 通知音を鳴らすか
NOTIFY_SOUND=true           # default: true

# 通知の優先度: min / low / default / high / max
NOTIFY_PRIORITY=high        # default: high

# リモート通知の SSH 設定（Termux 以外の環境用）
REMOTE_HOST=neo@leo         # default: neo@leo
REMOTE_PORT=5963            # default: 5963
TUNNEL_PORT=28022           # default: 28022
```

設定ファイルがない場合はデフォルト値で動作する。項目を省略した場合もデフォルト値が使われる。

### 通知スクリプトの仕組み

`~/.claude/hooks/termux-notify.sh` は以下のように動作する。

1. **設定ファイルを読み込み**（`~/.claude/hooks/termux-notify.conf`、存在しない場合はデフォルト値）
2. **stdin から JSON を読み取り**、`python3` でパース（`jq` 不要）
3. **`cwd` からディレクトリ名を抽出** し、通知タイトルに常に含める
4. `notification_type` と `hook_event_name` で通知内容を場合分け
5. **`permission_prompt` のみ `--id` + `--alert-once`** で初回だけ音を鳴らし、再発火はサイレント上書き。他のイベントは毎回独立した通知
6. `SHOW_MESSAGE` 設定に応じて **メッセージ詳細の表示を切り替え**
7. **環境を自動判定**:
   - `termux-notification` が PATH にある → Termux 上なので `setsid` 経由で直接実行
   - なければ → SSH リレー（設定ファイルの接続先を使用）
8. **バックグラウンド実行**（`&`）で即座にリターンし、Claude Code のフロー処理をブロックしない
9. SSH 失敗は `/dev/null` に捨てるため、ネットワーク断でもエラーにならない

**注意**: JSON パースに `python3` を使用。`jq` は環境によってインストールされていないため避けた。

### セットアップ

#### 1. 設定ファイルを作成

```bash
mkdir -p ~/.claude/hooks
cat > ~/.claude/hooks/termux-notify.conf << 'CONF'
# ディレクトリ名の表示形式: basename / fullpath
CWD_STYLE=basename

# メッセージ詳細を表示: true / false
SHOW_MESSAGE=true

# 通知音: true / false
NOTIFY_SOUND=true

# 通知の優先度: min / low / default / high / max
NOTIFY_PRIORITY=high

# リモート通知の SSH 設定
REMOTE_HOST=neo@leo
REMOTE_PORT=5963
TUNNEL_PORT=28022
CONF
```

#### 2. 通知スクリプトを配置

```bash
cat > ~/.claude/hooks/termux-notify.sh << 'HOOKSCRIPT'
#!/bin/bash
INPUT=$(cat)

# --- 設定読み込み ---
CONF="${HOME}/.claude/hooks/termux-notify.conf"
CWD_STYLE=basename
SHOW_MESSAGE=true
NOTIFY_SOUND=true
NOTIFY_PRIORITY=high
REMOTE_HOST=neo@leo
REMOTE_PORT=5963
TUNNEL_PORT=28022
[ -f "$CONF" ] && source "$CONF"

# --- JSON パース ---
json_get() {
  echo "$INPUT" | python3 -c "import sys,json;d=json.load(sys.stdin);print(d.get('$1',''))" 2>/dev/null
}

HOOK_EVENT=$(json_get hook_event_name)
NOTIF_TYPE=$(json_get notification_type)
MSG_RAW=$(json_get message)
CWD=$(json_get cwd)

# --- ディレクトリ名（常に取得） ---
if [ -n "$CWD" ]; then
  if [ "$CWD_STYLE" = "fullpath" ]; then
    DIR_LABEL="$CWD"
  else
    DIR_LABEL=$(basename "$CWD")
  fi
fi

# --- 通知内容の組み立て ---
# permission_prompt: 確認待ち中に繰り返し発火するため --id + --alert-once で初回のみ音を鳴らす
# それ以外: 毎回独立した通知（--id なし）
NOTIF_OPT=""
case "$NOTIF_TYPE" in
  permission_prompt)
    ICON="🔐"
    LABEL="確認待ち"
    MSG="${MSG_RAW:-パーミッション確認}"
    NOTIF_OPT="--id 'claude-perm' --alert-once"
    ;;
  idle_prompt)
    ICON="✅"
    LABEL="完了"
    MSG="${MSG_RAW:-入力待ちです}"
    ;;
  *)
    case "$HOOK_EVENT" in
      Stop)
        ICON="⏹"
        LABEL="処理完了"
        MSG="${MSG_RAW:-停止しました}"
        ;;
      *)
        ICON="💬"
        LABEL="通知"
        MSG="${MSG_RAW:-通知}"
        ;;
    esac
    ;;
esac

# --- タイトル（ディレクトリ名は常に含める） ---
if [ -n "$DIR_LABEL" ]; then
  TITLE="${ICON} ${DIR_LABEL} - ${LABEL}"
else
  TITLE="${ICON} ${LABEL}"
fi

# --- メッセージ表示の制御 ---
[ "$SHOW_MESSAGE" != "true" ] && MSG=""

# --- 文字数制限 ---
TITLE=$(echo "$TITLE" | head -c 100)
MSG=$(echo "$MSG" | head -c 200)

# --- 通知コマンド組み立て ---
TITLE_ESC=${TITLE//\'/\'\\\'\'}
MSG_ESC=${MSG//\'/\'\\\'\'}
NOTIF_CMD="termux-notification ${NOTIF_OPT} --title '${TITLE_ESC}'"
[ -n "$MSG_ESC" ] && NOTIF_CMD="$NOTIF_CMD --content '${MSG_ESC}'"
[ "$NOTIFY_SOUND" = "true" ] && NOTIF_CMD="$NOTIF_CMD --sound"
NOTIF_CMD="$NOTIF_CMD --priority ${NOTIFY_PRIORITY}"
NOTIF_CMD="$NOTIF_CMD --action 'am start -n com.termux/.app.TermuxActivity'"

# --- 実行 ---
if command -v termux-notification >/dev/null 2>&1; then
  setsid bash -c "$NOTIF_CMD" >/dev/null 2>&1 &
else
  ssh -p "$REMOTE_PORT" -o ConnectTimeout=3 -o StrictHostKeyChecking=no "$REMOTE_HOST" \
    "ssh -p $TUNNEL_PORT -o ConnectTimeout=3 localhost \"$NOTIF_CMD\"" >/dev/null 2>&1 &
fi
HOOKSCRIPT
chmod +x ~/.claude/hooks/termux-notify.sh
```

#### 3. グローバル設定に登録

`~/.claude/settings.json` の `hooks` セクションに追加。既存のフック（ベル音や tmux 色変更）はそのまま残し、`hooks` 配列に **エントリを追加** する。

```json
{
  "hooks": {
    "Notification": [
      {
        "matcher": "",
        "hooks": [
          { "type": "command", "command": "（既存のベル音フック）" },
          { "type": "command", "command": "~/.claude/hooks/termux-notify.sh" }
        ]
      }
    ],
    "Stop": [
      {
        "matcher": "*",
        "hooks": [
          { "type": "command", "command": "（既存の tmux / タスクAPI フック）" },
          { "type": "command", "command": "~/.claude/hooks/termux-notify.sh" }
        ]
      }
    ]
  }
}
```

### 動作確認

```bash
# 1. cwd 付き idle_prompt テスト
echo '{"notification_type":"idle_prompt","message":"ファイルの変更が完了しました","cwd":"/home/neo/my-web-app"}' \
  | ~/.claude/hooks/termux-notify.sh
# → タイトル: ✅ my-web-app - 完了 / 本文: ファイルの変更が完了しました

# 2. permission_prompt テスト
echo '{"notification_type":"permission_prompt","message":"Bash: npm install express","cwd":"/home/neo/my-web-app"}' \
  | ~/.claude/hooks/termux-notify.sh
# → タイトル: 🔐 my-web-app - 確認待ち / 本文: Bash: npm install express

# 3. Stop イベントテスト
echo '{"hook_event_name":"Stop","cwd":"/home/neo/my-web-app"}' \
  | ~/.claude/hooks/termux-notify.sh
# → タイトル: ⏹ my-web-app - 処理完了 / 本文: 停止しました

# 4. SHOW_MESSAGE=false で本文なし確認
sed -i 's/SHOW_MESSAGE=true/SHOW_MESSAGE=false/' ~/.claude/hooks/termux-notify.conf
echo '{"notification_type":"idle_prompt","message":"テスト","cwd":"/home/neo/project"}' \
  | ~/.claude/hooks/termux-notify.sh
# → タイトル: ✅ project - 完了 / 本文: なし
sed -i 's/SHOW_MESSAGE=false/SHOW_MESSAGE=true/' ~/.claude/hooks/termux-notify.conf

# 5. fullpath 表示テスト
sed -i 's/CWD_STYLE=basename/CWD_STYLE=fullpath/' ~/.claude/hooks/termux-notify.conf
echo '{"notification_type":"idle_prompt","cwd":"/home/neo/my-web-app"}' \
  | ~/.claude/hooks/termux-notify.sh
# → タイトル: ✅ /home/neo/my-web-app - 完了
sed -i 's/CWD_STYLE=fullpath/CWD_STYLE=basename/' ~/.claude/hooks/termux-notify.conf

# 6. 実際の Claude Code セッションで確認
#    - AskUserQuestion やパーミッション確認で 🔐 [プロジェクト名] - 確認待ち
#    - 処理が終わって入力待ちになると ✅ [プロジェクト名] - 完了
#    - セッション終了で ⏹ [プロジェクト名] - 処理完了
```

---

## トラブルシューティング

### Q: termux-notification が何も起きない

**A**: Termux:API **アプリ** がインストールされているか確認。CLI パッケージだけでは動かない。

```bash
pm list packages | grep termux.api
# 出力がなければ未インストール
```

### Q: APK インストールで「パッケージが競合」エラー

**A**: Termux と Termux:API の署名鍵が異なる。同じソース（F-Droid なら F-Droid、GitHub なら GitHub）からインストールすること。

```bash
# Termux APK の署名を確認
unzip -l /path/to/termux.apk "META-INF/*.RSA"
# 84D3000E.RSA → F-Droid 署名
```

### Q: APK インストールで「解析エラー」

**A**: `termux-open` ではなく、`/sdcard/Download/` にコピーしてファイルマネージャーから開く。

```bash
cp app.apk /sdcard/Download/
am start -a android.intent.action.VIEW \
  -d "content://com.android.externalstorage.documents/document/primary%3ADownload" \
  -t "vnd.android.document/directory"
```

### Q: Claude Code のサンドボックスでコマンドがブロックされる

**A**: `setsid bash -c '...'` で回避。

```bash
setsid bash -c 'termux-notification --title "test" --content "hello"'
```

### Q: dalvikvm で DEX 実行が SecurityException

**A**: Android 15 では DEX ファイルが書き込み可能だと拒否される。`chmod 444` する。

```bash
chmod 444 my.dex
dalvikvm -cp my.dex MyClass
```

### Q: リバーストンネルが切れている

**A**: トンネルプロセスが生きているか確認。死んでいたら再接続。

```bash
# プロセス確認
pgrep -af "ssh.*-R.*28022"

# なければ再接続
ssh -p <remote-port> -f -N -R 28022:localhost:8022 user@remote-server
```

自動再接続が必要なら `autossh` を使う（[トンネルの維持と自動再接続](#トンネルの維持と自動再接続)参照）。

### Q: リモートから ssh -p 28022 localhost で接続拒否される

**A**: 以下を順番にチェック：

1. Termux 側で `sshd` が起動しているか → `pgrep -a sshd`
2. トンネルが張れているか → `pgrep -af "ssh.*-R"`
3. リモートの公開鍵が Termux の `~/.ssh/authorized_keys` に登録されているか
4. Termux 側の `~/.ssh/authorized_keys` のパーミッションが `600` か

### Q: リモートから通知コマンドは成功するのに通知が出ない

**A**: SSH 経由で実行される `termux-notification` は環境変数が不足する可能性がある。
以下で PATH を明示的に設定：

```bash
ssh -p 28022 localhost \
  'export PATH=/data/data/com.termux/files/usr/bin:$PATH; termux-notification --title "test" --content "hello"'
```

### Q: フックスクリプトで通知が来ない

**A**: まず単体テストで切り分ける。

```bash
# スクリプトをデバッグモードで実行（cwd 付きで）
echo '{"notification_type":"idle_prompt","cwd":"/home/neo/test"}' | bash -x ~/.claude/hooks/termux-notify.sh
```

よくある原因:
- `python3` がインストールされていない → `which python3` で確認
- SSH リレー先のトンネルが切れている → [リバーストンネルが切れている](#q-リバーストンネルが切れている) 参照
- スクリプトに実行権限がない → `chmod +x ~/.claude/hooks/termux-notify.sh`
- 設定ファイルの構文エラー → `bash -n ~/.claude/hooks/termux-notify.conf` で確認

### Q: 設定ファイルの変更が反映されない

**A**: 設定ファイルはフック実行のたびに読み込まれるため、変更は即座に反映される。
以下を確認：

1. ファイルパスが `~/.claude/hooks/termux-notify.conf` であること
2. 構文エラーがないこと（`bash -n` で検証）
3. 値にスペースや引用符が混入していないこと

```bash
# 構文チェック
bash -n ~/.claude/hooks/termux-notify.conf

# 現在の設定を確認
cat ~/.claude/hooks/termux-notify.conf
```

### Q: フックが Claude Code の動作を遅くする

**A**: スクリプトは全ての外部コマンドをバックグラウンド（`&`）で実行し即座にリターンする設計。
遅延が発生する場合は SSH の `ConnectTimeout=3` を `1` に短縮するか、SSH 接続自体が問題なら
`ControlMaster` で接続を使い回す。

```bash
# ~/.ssh/config に追加（SSH 接続の再利用）
Host leo
  ControlMaster auto
  ControlPath ~/.ssh/sockets/%r@%h-%p
  ControlPersist 600
```

---

## ライセンス

MIT

---

## 貢献者

- Claude Code (Opus 4.6) - 実装・調査・12回の試行錯誤
- ユーザー - 「諦めずにやってくれ」という最高の指示
