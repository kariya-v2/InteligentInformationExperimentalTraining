# 目次
1. [目的と完成条件](#1-目的と完成条件)
2. [全体構成図とポート設計](#2-全体構成図とポート設計)
3. [前提条件と事前準備](#3-前提条件と事前準備)
4. [構築手順（Docker Compose設定）](#4-構築手順docker-compose設定)
5. [動作確認と検証](#5-動作確認と検証)
6. [トラブルシューティング](#6-トラブルシューティング)
7. [セキュリティ配慮と環境の初期化](#7-セキュリティ配慮と環境の初期化)
8. [参考資料](8-参考資料)

# 1. 目的と完成条件
## 1.1 目的

  WSL2 上に Docker を用いて Pi-hole サーバーを構築し、LAN 内（またはローカルホスト）のブラウザ広告をネットワークレベルで遮断する。

## 1.2 完成条件

- 管理画面へのアクセス: ブラウザで http://localhost:8080/admin にアクセスし、Pi-holeのダッシュボードが表示されること。

- DNS名前解決の成功: ターミナルで nslookup コマンドを実行し、Pi-hole経由で正しいIPアドレスが返ってくること。

- 広告ドメインの拒否: 既知の広告ドメインを問い合わせた際、IPアドレスが 0.0.0.0（またはブロック用IP）として返されること。


# 2. 全体構成図とポート設計

## 2.1 全体構成図
本システムは、Windows 11上の WSL2 (Ubuntu 22.04 LTS) 内で動作する Dockerコンテナ として構築します。クライアント（ブラウザやアプリ）からのDNSリクエストをPi-holeコンテナが受け取り、広告ドメインであればブロック、正規のドメインであれば上位のDNSサーバー（Google DNSなど）へ転送する仕組みです。Host: Windows 11 / WSL2 (Ubuntu 22.04)Container: Pi-hole (Official Docker Image)Upstream DNS: Google Public DNS ($8.8.8.8$) または Cloudflare ($1.1.1.1$)

## 2.2 ポート設計
Pi-holeを正常に動作させるため、ホスト（WSL2）とコンテナ間で以下のポートをマッピングします。

| ホスト側ポート                          | コンテナ側ポート                                         | プロトコル          |用途        |
| :---------------------------- | :------------------------------------------- | :------------------- |:--------------------|
| 53                    | 53                                 | UDP/TCP`             |DNSサービス: 名前解決のリクエストを受け付ける最重要ポート。|
| 8080       | 80           | TCP   |管理画面 (Web UI): ブラウザから設定・統計を確認するためのポート。|
| 443 (任意)                    |443             | TCP |HTTPS: Webインターフェースの暗号化用（本構成では基本HTTPを使用）。
| 

> [!IMPORTANT]
ポート53の競合について
多くのLinuxディストリビューション（Ubuntu等）では、デフォルトで systemd-resolved というサービスがポート53を使用しています。このままではPi-holeを起動できないため、次章以降の「事前準備」でこの競合を解消する設定を行います。

## 2.3 データの永続化
コンテナを削除しても設定やログが消えないよう、ホスト上のディレクトリをコンテナ内にマウントします。

- `./etc-pihole/` → `/etc/pihole/`: 広告リストや基本設定を保存

- `./etc-dnsmasq.d/` → `/etc/dnsmasq.d/`: DNSサーバー（dnsmasq）の詳細設定を保存

# 3. 前提条件と事前準備
## 3.1 実行環境（前提条件）
本手順書は、以下の環境で動作を確認しています。これらを用意してください。

- OS: Windows 11 Pro / Home (22H2以上推奨)

- プラットフォーム: WSL2 (Windows Subsystem for Linux 2)

- ディストリビューション: Ubuntu 22.04 LTS

- ネットワーク: インターネット接続環境（固定IPは不要だが、WSL内のIP確認が必要）

## 3.2 手順①：WSL2とUbuntuのセットアップ
まだWSL2を導入していない場合は、Windowsターミナル（PowerShell）を管理者権限で開き、以下のコマンドを実行します。

```
# WSL2とUbuntuのインストール
wsl --install -d Ubuntu-22.04

# インストール後、PCを再起動し、ユーザー名・パスワードを設定してください。
```
## 3.3 手順②：パッケージの更新
Ubuntuにログインし、システムを最新の状態にします。
```
# パッケージリストの更新とアップグレード
sudo apt update && sudo apt upgrade -y
```
## 3.4 手順③：ポート53の競合回避（重要）
Ubuntu 22.04では、systemd-resolved が DNS 用のポート 53 を既に使用しています。これを停止させないと Pi-hole が起動できません。
1.サービスの停止と無効化:
```
sudo systemctl stop systemd-resolved
sudo systemctl disable systemd-resolved
```
2.DNS解決設定の変更:
`/etc/resolv.conf` が `systemd-resolved` へのシンボリックリンクになっているため、削除して新しく作成します。
```
sudo rm /etc/resolv.conf
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```
**注釈**: これにより、WSL自体が一時的にGoogle DNS ($8.8.8.8$) を使うようになり、Dockerイメージのダウンロードが可能になります。

## 3.5 手順④：Docker & Docker Compose のインストール
コンテナ環境を構築するためのツールをインストールします。
```
# Dockerのインストール
sudo apt install -y docker.io docker-compose

# 現在のユーザーをdockerグループに追加（sudoなしで実行可能にするため）
sudo usermod -aG docker $USER

# 反映のために一度ログアウトして再ログイン、または以下のコマンドを実行
newgrp docker
```

# 4. 構築手順（Docker Compose設定）
## 4.1 プロジェクトディレクトリの作成
設定ファイルを整理するため、専用の作業ディレクトリを作成して移動します。

```
# プロジェクト用ディレクトリの作成
mkdir ~/pihole-project && cd ~/pihole-project

# データ永続化用のディレクトリを作成
mkdir -p ./etc-pihole ./etc-dnsmasq.d
```
## 4.2 環境変数ファイル（.env）の作成
管理画面のパスワードやタイムゾーンなどの機密・環境依存情報を分離します。
以下の内容で .env という名前のファイルを新規作成してください。

- 保存パス: ~/pihole-project/.env

- 権限: 600 (所有者のみ読み書き可能)

```
# .env ファイルの作成
cat <<EOF > .env
TZ=Asia/Tokyo
WEBPASSWORD=AdminPassword123!
PIHOLE_DNS=8.8.8.8;1.1.1.1
EOF

# 権限変更（セキュリティ配慮）
chmod 600 .env
```
**注釈**: WEBPASSWORD は管理画面のログインに使用します。各自で強力なパスワードに変更してください。

## 4.3 Docker Composeファイルの作成（完全版）
Pi-holeコンテナの構成を定義する docker-compose.yml を作成します。授業で学んだコンテナの知識を活かし、ポートマッピングとボリュームマウントを正確に記述します。

- 保存パス: ~/pihole-project/docker-compose.yml

```
version: "3"

services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    # ホストとコンテナのポートを紐付け
    ports:
      - "53:53/tcp"
      - "53:53/udp"
      - "8080:80/tcp" # 管理画面をホストの8080で公開
    environment:
      TZ: ${TZ}
      WEBPASSWORD: ${WEBPASSWORD}
      PIHOLE_DNS_: ${PIHOLE_DNS}
    # データの永続化設定
    volumes:
      - './etc-pihole:/etc/pihole'
      - './etc-dnsmasq.d:/etc/dnsmasq.d'
    # ネットワーク設定
    # WSL2環境での名前解決トラブルを防ぐためホストモードは使わずブリッジを使用
    restart: unless-stopped
```
## 4.4 コンテナの起動
作成した設定ファイルをもとに、バックグラウンドでサービスを開始します。

```
# コンテナのビルドと起動
docker-compose up -d

# 起動状態の確認（StatusがUpになっているか）
docker ps
```

# 5. 動作確認と検証
構築した Pi-hole が正常に動作しているか、以下の 3 つのステップで検証を行います。

## 5.1 管理画面（Web UI）へのアクセス確認
ブラウザを起動し、ホスト（Windows）側からコンテナ内の管理画面にアクセスできるか確認します。

ブラウザで http://localhost:8080/admin を開きます。

ログイン画面が表示されたら、.env で設定した WEBPASSWORD を入力してログインします。

ダッシュボードの表示: 下記のような統計画面（Queries Blocked 等）が表示されれば、Web サーバーとしての機能は正常です。

## 5.2 DNS 名前解決のテスト（疎通確認）
ターミナル（PowerShell または WSL2）から、Pi-hole が DNS サーバーとして機能しているか nslookup コマンドで検証します。

```
# ローカルホスト（Pi-hole）を指定して google.com を名前解決
nslookup google.com 127.0.0.1
```
- 期待される結果: Google の正しい IP アドレスが返ってくること。

- 確認ポイント: Server: 127.0.0.1 からの回答であることを確認します。

## 5.3 広告ブロック機能の検証（判定条件）
次に、既知の広告ドメインを問い合わせ、Pi-hole が正しく「ブロック」するかを確認します。

```
# 代表的な広告配信ドメイン（doubleclick.net）を問い合わせ
nslookup doubleclick.net 127.0.0.1
```
期待される結果: IP アドレスが 0.0.0.0 として返ってくること。

ログ確認: Pi-hole 管理画面の「Query Log」を開き、doubleclick.net が OK (blocked) と赤く表示されていることを確認してください。

## 5.4 サービスの永続化確認
WSL2 または Docker を再起動しても、サービスが自動で立ち上がるか確認します。

```
# Dockerコンテナの再起動ポリシー確認
docker inspect pihole --format='{{.HostConfig.RestartPolicy.Name}}'
```
unless-stopped と表示されれば、手動で止めない限り OS 起動時に自動実行されます。

# 6. トラブルシューティング
構築過程で発生しやすい代表的なエラーと、その具体的な対処法を記述します。

## 6.1 ポート53が既に使用されている（Bind for 0.0.0.0:53 failed）
docker-compose up 時に、ポート53の競合エラーが発生した場合。

- 原因: 第3章の systemd-resolved の停止が不完全であるか、他のDNSサービス（以前のインストール物など）が動いている。

- 対処法: 以下のコマンドで、現在ポート53を占有しているプロセスを確認して停止します。

```
# ポート53を使用しているプロセスID(PID)を確認
sudo lsof -i :53
# 該当するプロセスを停止（例：PID 1234の場合）
sudo kill -9 1234
```
## 6.2 WSL2再起動後にインターネットに繋がらない
WSL2を再起動した後、apt update 等が失敗する場合。

- 原因: /etc/resolv.conf を手動で書き換えたため、WSL2が本来参照すべきホストOSのDNS設定を見失っている。

- 対処法: 本構成ではPi-hole自身がDNSになるため、コンテナが動いていれば解消されます。コンテナが停止している場合は、一時的に手動で書き換えます。

```
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
```
## 6.3 管理画面にアクセスできない（Connection Refused）
ブラウザで http://localhost:8080/admin が開けない場合。

- 原因: コンテナの起動失敗、またはWindows側のファイアウォールによる遮断。

- 対処法:

  - コンテナの状態を確認: docker ps で pihole が Up か確認。

  - ログを確認: docker logs pihole でエラーメッセージ（PHPやLighttpdの起動失敗など）が出ていないか確認。

  - ポートの確認: docker-compose.yml で 8080:80 と正しく設定されているか再確認。

## 6.4 DNS設定が反映されない（広告が消えない）
コンテナは動いているが、実際にブラウザの広告が消えない場合。

- 原因: Windows（ホストOS）のネットワーク設定で、DNSサーバーとしてWSL2のIPを指定していない。

- 対処法:

  - WSL2のIPを確認: ip addr show eth0

  - Windowsの「ネットワーク設定」→「アダプターのオプション」から、DNSサーバーのアドレスに上記IPを優先設定する。

  - ブラウザのキャッシュをクリアする。

最終章となる第7章では、実運用を想定したセキュリティ面での工夫と、評価基準にある「更新方針」、そして不要になった際にシステムを完全にクリーンアップする手順を記述します。

# 7. セキュリティ配慮と環境の初期化
## 7.1 セキュリティ配慮
本サーバーを安全に運用するために、以下の対策を講じています。

- 秘密情報の分離 (.env の利用):
管理画面のパスワードを docker-compose.yml に直接記述せず、外部ファイル .env で管理しています。これにより、GitHub 等にコードを公開する際、誤ってパスワードを流出させるリスクを低減しています（.gitignore への追加を推奨）。

- 最小権限の原則:
管理画面用ポートを、標準の 80 番ではなく 8080 番に変更して公開することで、ホスト OS 側の Web サービスとの衝突を防ぎ、意図しないアクセスを抑制しています。

- 不要なポートの閉鎖:
DNS サービスに必要なポート（53）と管理用（8080）以外は、コンテナ外に公開していません。

## 7.2 運用と更新方針
コンテナや広告リストを最新の状態に保つため、定期的に以下のコマンドを実行することを推奨します。

```
# 1. ディレクトリへ移動
cd ~/pihole-project

# 2. 最新のイメージを取得
docker-compose pull

# 3. コンテナの再作成（最新イメージの適用）
docker-compose up -d

# 4. 未使用の古いイメージを削除
docker image prune -f
```
## 7.3 環境の初期化（元の状態に戻す方法）
本課題で作成した環境を削除し、OS の設定を元の状態に戻す手順です。

① コンテナとデータの削除
```
cd ~/pihole-project
# コンテナとネットワークの削除
docker-compose down
# 永続化データの削除（必要に応じて）
cd ..
rm -rf ~/pihole-project
```
② DNS 設定（systemd-resolved）の復元
WSL2（Ubuntu）の DNS 設定を標準状態に戻します。

```
# 1. シンボリックリンクの再作成（OS標準の状態へ）
sudo rm /etc/resolv.conf
sudo ln -s /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf

# 2. サービスの再開と有効化
sudo systemctl start systemd-resolved
sudo systemctl enable systemd-resolved
```
# 第8章：参考資料
- Pi-hole Official Documentation

- Pi-hole Docker Hub (Official Image)

- WSL2 での Docker Compose インストールガイド（Ubuntu公式）
