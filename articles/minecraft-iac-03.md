---
title: "MinecraftサーバーををIaC化してみた【Part3】 - Ansibleで構成管理を自動化した"
emoji: "🐕"
type: "tech"
topics: :
  - "ansible"
  - "docker"
  - "minecraft"
  - "localstack"
  - "opentofu"
published: true
---

# MinecraftサーバーをIaCで作る Part3 - Ansibleで構成管理を自動化した

## この記事で書くこと

MinecraftサーバーをIaCで作るプロジェクトのPart3として、**AnsibleでEC2の構成管理を自動化**しました。

Part1でローカル環境（LocalStack + Docker）を、Part2でOpenTofuを使ってAWSにVPC/EC2/S3などのインフラを構築しました。

Part3はその続きで、

- AnsibleでEC2にDockerとMinecraftサーバーをセットアップ
- バックアップの自動化（cron + S3）
- systemdでEC2再起動時にMinecraftが自動起動
- 途中でめちゃくちゃ詰まったので、原因と解決方法もまとめる

という内容です。

---

## Part3で作ったもの

### 目標
「`ansible-playbook`を1回叩くだけでMinecraftサーバーが動く状態」にする。

### 実際に作ったもの

- Ansibleのディレクトリ構造（Role, Playbook, Inventory）
- commonロール：タイムゾーン、パッケージインストール
- dockerロール：Docker + Docker Composeのインストール
- minecraftロール：docker-compose.ymlの配置、コンテナ起動、systemdサービス化
- バックアップスクリプト（S3へのアップロード + cron設定）
- 各種Playbook（setup / deploy / start / stop / backup / upgrade）

---

## Ansibleって何をするもの？

OpenTofuとAnsibleの役割分担がずっとぼんやりしていたので、ここで整理した。

```
OpenTofu → 「箱」を作る
             EC2、VPC、S3などのインフラを構築

Ansible  → 「箱の中身」を設定する
             DockerのインストールやMinecraftの設定など
```

例えるなら、OpenTofuがマンションを建てる工事で、Ansibleが引っ越し後の家具の配置や内装工事みたいなイメージ。

Ansibleの特徴として「**冪等性**（べきとうせい）」という概念がある。何度実行しても同じ結果になる、という性質のことで、これがシェルスクリプトとの大きな違いだった。

```bash
# シェルスクリプトの場合：実行するたびに同じ行が追加されていく
echo "export PATH=$PATH:/opt/bin" >> ~/.bashrc
# 2回目: 重複する、3回目: さらに重複する...

# Ansibleの場合：既に同じ内容があればスキップされる
- name: Add PATH to bashrc
  ansible.builtin.lineinfile:
    path: ~/.bashrc
    line: 'export PATH=$PATH:/opt/bin'
# 何回実行しても1行だけ
```

---

## ディレクトリ構造

```
ansible/
├── ansible.cfg                          # Ansible全体の設定（接続方式・インベントリパス等）
├── inventory/
│   ├── group_vars/
│   │   └── all/                         # 全ホスト共通の変数（ディレクトリ形式で分割管理）
│   │       ├── main.yml                 # 通常変数（ポート番号・パス・設定値等）
│   │       └── vault.yml                # 暗号化変数（パスワード・APIキー等）
│   ├── host_vars/
│   │   ├── minecraft-server.yml         # minecraft-server固有の変数（実際の値）
│   │   └── minecraft-server.yml.sample  # host_varsのサンプル（git管理用テンプレート）
│   ├── hosts.yml                        # 接続先サーバーの定義（実際の値）
│   └── hosts.yml.example                # hostsのサンプル（git管理用テンプレート）
├── playbooks/
│   ├── setup.yml                        # 初期セットアップ（Docker等の環境構築）
│   ├── deploy.yml                       # Minecraftデプロイ（コンテナ起動含む）
│   ├── start.yml                        # サーバー起動
│   ├── stop.yml                         # サーバー停止
│   ├── backup.yml                       # ワールドデータのバックアップ
│   ├── restore.yml                      # バックアップからのリストア
│   └── upgrade.yml                      # Minecraftバージョンアップ
├── requirements.yml                     # 外部ロール・コレクションの依存定義（ansible-galaxy用）
└── roles/
    ├── common/                          # 全サーバー共通の設定
    │   └── tasks/
    │       └── main.yml                 # タスクのエントリポイント
    ├── docker/                          # Dockerのセットアップ
    │   ├── defaults/
    │   │   └── main.yml                 # Dockerのデフォルト変数（バージョン等）
    │   ├── handlers/
    │   │   └── main.yml                 # Dockerデーモンの再起動等
    │   └── tasks/
    │       └── main.yml                 # Dockerインストール・設定タスク
    └── minecraft/                       # Minecraft固有の設定
        ├── defaults/
        │   └── main.yml                 # Minecraftのデフォルト変数（バージョン・メモリ等）
        ├── handlers/
        │   └── main.yml                 # コンテナ再起動等
        ├── tasks/
        │   ├── main.yml                 # タスクのエントリポイント（各タスクをinclude）
        │   ├── setup.yml                # 初回セットアップ処理
        │   └── backup.yml               # バックアップ処理
        └── templates/
            ├── docker-compose.yml.j2    # docker-compose定義のテンプレート
            ├── minecraft.service.j2     # systemdサービス定義のテンプレート
            └── backup.sh.j2             # バックアップスクリプトのテンプレート
```

**Roleとは？** タスクをまとめた再利用可能なモジュールのこと。「Docker」「Minecraft」のように機能単位で分割することで、管理しやすくなる。

---

## 詰まったところ集

ここからが本題。Part3は「なぜ動かないの？」が続く修羅場だった。

---

### 1. Roleが見つからない（roles_pathの問題）

Playbookを実行したら最初からこれ。

```
the role 'common' was not found in 
  .../ansible/playbooks/roles
  .../ansible/roles  ← ここが検索対象に入っていない
```

`ansible.cfg`に`roles_path = roles`と書いていたのに、どこを探しているかを見ると`ansible/playbooks/roles`になっていた。

**原因:** `roles_path`の相対パスが実行ディレクトリ基準で解決される。`ansible/`から実行していれば`ansible/roles`になるはずが、うまく効いていなかった。

**解決方法:** `ansible/`ディレクトリの中から実行するのを徹底した。

```bash
cd ansible
ansible-playbook playbooks/setup.yml
```

---

### 2. Workerが死んでいる（macOSのfork問題）

```
A worker was found in a dead state
```

接続自体は通っているのになぜ？と思ったら、これはmacOS固有の既知バグだった。

**原因:** macOSのObjective-CランタイムがAnsibleの`fork()`を安全でないと判断してプロセスを殺す。

**解決方法:** `.env`ファイルに環境変数を追加して回避した。

```bash
export OBJC_DISABLE_INITIALIZE_FORK_SAFETY=YES
```

---

### 3. Docker Composeのcompose pluginがない

```
Docker CLI /usr/bin/docker does not have the compose plugin installed
```

Amazon Linux 2023のdnfリポジトリには`docker-compose-plugin`パッケージが存在しないため、dnfでのインストールが失敗していた。

さらにAnisbleがcompose pluginを要求するモジュール（`community.docker.docker_compose_v2`）を使っていたため、standalone版の`docker-compose`コマンドがあるだけでは動かなかった。

**解決方法:** standalone版のバイナリをDockerのpluginとして認識させるシンボリックリンクを作成した。

```yaml
- name: Create Docker CLI plugins directory
  ansible.builtin.file:
    path: /usr/lib/docker/cli-plugins
    state: directory
    mode: '0755'

- name: Symlink docker-compose as Docker CLI plugin
  ansible.builtin.file:
    src: /usr/local/bin/docker-compose
    dest: /usr/lib/docker/cli-plugins/docker-compose
    state: link
```

Docker CLIは`/usr/lib/docker/cli-plugins/`にバイナリがあればpluginとして認識する仕様になっている。これを知るまでかなり時間がかかった。

```
docker compose version を実行
    ↓
Docker CLIが /usr/lib/docker/cli-plugins/ を探す
    ↓
シンボリックリンクが見つかる
    ↓
plugin として認識・実行される
```

---

### 4. YAMLのクォートの閉じ忘れ

```
yaml: while parsing a block collection at line 6, column 7:
did not find expected '-' indicator
```

docker-compose.yml.j2テンプレートの中に、クォートを閉じ忘れていた箇所があった。

```yaml
# 壊れてる
- "{{ rcon_port }}:25575   ← " がない

# 正しい
- "{{ rcon_port }}:25575"
```

地味だけど見落としやすかった。エラーメッセージから該当箇所を特定するのに少し時間がかかった。

---

### 5. イメージタグの二重指定

```
unable to get image 'itzg/minecraft-server:latest:latest'
```

`group_vars/all.yml`に`minecraft_image: "itzg/minecraft-server:latest"`と書いていて、テンプレートでさらに`:{{ minecraft_image_tag }}`を付けていたため、`:latest:latest`になっていた。

**解決方法:** `group_vars/all.yml`の変数からタグを外した。

```yaml
# 修正前
minecraft_image: "itzg/minecraft-server:latest"

# 修正後
minecraft_image: "itzg/minecraft-server"
```

変数の定義が`group_vars/all.yml`と`roles/defaults/main.yml`に分散していて、片方が上書きされていることに気づかなかったのが原因。

---

### 6. crontabコマンドが見つからない

```
Failed to find required executable "crontab" in paths
```

Amazon Linux 2023のminimal AMIには`crontab`コマンドが入っていない。`cronie`パッケージをインストールする必要があった。

```yaml
- name: Install required packages
  ansible.builtin.dnf:
    name:
      - wget
      - tar
      - git
      - cronie  # ← これを追加
    state: present
```

---

### 7. `restarted`パラメータが存在しない

```
Unsupported parameters for (community.docker.docker_compose_v2) module: restarted
```

古いバージョンのdocker_composeモジュールのドキュメントを参考にしていたため、存在しないパラメータを使っていた。

**解決方法:** `restarted`の代わりに`recreate: always`を使う。

```yaml
- name: Restart Minecraft if config changed
  community.docker.docker_compose_v2:
    project_src: "{{ minecraft_dir }}"
    state: present
    recreate: always  # restarted ではなくこちら
  when: compose_file.changed or env_file.changed
```

---

## Jinja2テンプレートで詰まったこと

AnsibleのテンプレートはJinja2という形式を使う。`{{ 変数名 }}`で変数を埋め込む仕組みだが、Dockerのformat文字列と衝突する場面があった。

```jinja2
# docker psのformat文字列をそのまま書くと...
docker ps --format "{{.Names}}"
# → Ansibleがこれを変数として解釈しようとしてエラーになる

# エスケープが必要
docker ps --format '{{"{{"}}.Names{{"}}"}}'
# → Ansibleが処理した後は {{.Names}} になる
```

最初見た時は「なにこの記述」となったが、意味を理解したら納得できた。

---

## Inventoryの変数管理をきれいにした

途中で「変数がいろんなところに散らばっていて管理しにくい」という問題に気づいた。

整理前は：

- `hosts.yml`の`vars:`にMinecraftのバージョンやポート番号
- `group_vars/all.yml`にイメージ名や難易度
- `roles/defaults/main.yml`に重複して定義

どこが正として使われているか分かりにくかったので、役割を明確に分けた。

```
host_vars/minecraft-server.yml  → 環境依存・秘密情報
  - ec2_instance_id             （デプロイのたびに変わる）
  - ssm_bucket_name
  - backup_s3_bucket
  - rcon_password

group_vars/all.yml              → アプリの設定値
  - minecraft_version
  - minecraft_memory
  - minecraft_port_java
  - その他ゲーム設定

hosts.yml                       → 接続情報のみ
  - ansible_host
  - ansible_connection
  - SSM接続設定
```

`host_vars/`と`group_vars/`はAnsibleの予約ディレクトリなので、ここに置いたファイルは自動で読み込まれる。

また、`host_vars/minecraft-server.yml`はEC2のIPやパスワードを含むので`.gitignore`に追加して、`host_vars/minecraft-server.yml.example`というダミー値のサンプルファイルだけGit管理した。

---

## systemdへの登録

EC2を再起動したときにMinecraftサーバーが自動的に起動するようにsystemdに登録した。

```ini
[Unit]
Description=Minecraft Server (Docker Compose)
Requires=docker.service
After=docker.service

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/minecraft
ExecStart=/usr/local/bin/docker-compose up -d
ExecStop=/usr/local/bin/docker-compose down

[Install]
WantedBy=multi-user.target
```

ここで学んだポイントが2つ。

**`Requires`と`After`は別物：**
- `Requires=docker.service` → Dockerが死んだらMinecraftも止まる（依存関係）
- `After=docker.service` → Dockerが起動した後に起動する（順序の保証）
- 両方書かないと「依存はあるが同時に起動しようとする」という事故が起きる

**`Type=oneshot`と`RemainAfterExit=yes`の組み合わせ：**
- `docker-compose up -d`はバックグラウンド起動なのでコマンド自体はすぐ終わる
- systemdはプロセスが終わったら「サービスが死んだ」と思う
- `Type=oneshot`で「1回実行して終わるタイプ」と宣言
- `RemainAfterExit=yes`で「終わっても起動中として扱う」
- これがないと`systemctl status minecraft`がinactiveになってしまう

---

## バックアップの仕組み

バックアップスクリプトの処理フロー：

```
1. コンテナが起動中か確認
   ├── 起動中 → プレイヤーに通知 → save-off → save-all → save-on
   └── 停止中 → スキップ

2. tar.gz でデータをアーカイブ

3. aws s3 cp でS3にアップロード

4. 7日以上前のローカルバックアップを削除
```

Ansibleのcronモジュールで毎日4:00(JST)に自動実行するよう設定した。

```yaml
- name: Setup backup cron job
  ansible.builtin.cron:
    name: "Minecraft daily backup"
    minute: "0"
    hour: "19"  # UTC 19:00 = JST 4:00
    job: "{{ minecraft_dir }}/backup.sh >> /var/log/minecraft-backup.log 2>&1"
    user: root
```

---

## Part3で学んだこと

- AnsibleのRole構造（tasks / handlers / templates / defaults）
- Jinja2テンプレートの書き方と変数のスコープ
- 冪等性を意識したタスク設計
- systemdの`Requires`と`After`の使い分け
- `Type=oneshot`の意味と使いどころ
- Ansibleの変数の優先順位（`group_vars` > `defaults`など）
- macOS固有の`fork()`問題
- Docker pluginとstandalone版の違い

詰まった箇所が多かった分、それぞれの原因を自分で追いかけたので理解度は高いと感じている。

---

## 次にやること（Part4以降）

Part3でEC2上でMinecraftが自動起動する状態になったので、次は

- Discord BotをLambdaで実装
- スラッシュコマンドでEC2の起動/停止/状態確認
- API Gatewayとの連携

を進める予定。

また記事にまとめます。

---

## おまけ：Part3でよく使ったコマンド

```bash
# 接続テスト
ansible minecraft -m ping

# セットアップ実行
ansible-playbook playbooks/setup.yml

# Minecraftデプロイ
ansible-playbook playbooks/deploy.yml

# 特定のタグだけ実行
ansible-playbook playbooks/setup.yml --tags docker

# 詳細ログで実行（デバッグ時）
ansible-playbook playbooks/deploy.yml -v

# ドライラン（実際には変更しない）
ansible-playbook playbooks/deploy.yml --check

# 変数を上書きして実行
ansible-playbook playbooks/upgrade.yml -e "minecraft_version=1.21.5"
```