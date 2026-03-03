---
title: "MinecraftサーバーををIaC化してみた【Part1】- 環境構築編"
emoji: "🔥"
type: "tech"
topics:
  - "aws"
  - "docker"
  - "minecraft"
  - "localstack"
  - "opentofu"
published: true
published_at: "2026-02-21 18:24"
---

## この記事で書くこと
MinecraftサーバーをIaCで作るプロジェクトのPhase1として、ローカル開発環境を作りました。

- LocalStack（ローカルAWS）をDockerで起動
- MinecraftサーバーをDockerで起動
- RCONでサーバー操作できることを確認
- volumesでワールドが消えない（永続化）ことを確認

自分はDocker Composeを書いたのが初めてで、途中で
「portsって何？」「環境変数ってただの変数？」「マウントって何？」みたいに詰まったので、学んだことをまとめます。

初学者の備忘録なので間違いとか頭悪そうでも許してください、、

---

## Phase1で作ったもの
### 1) LocalStack + Minecraft の docker-compose
LocalStackと、Minecraftサーバー（itzg/minecraft-server）をcomposeで起動できるようにしました。

構成はこんな感じ：

- LocalStack： http://localhost:4566 でAWS APIっぽいものが動く
- Minecraft： localhost:25565 でゲーム接続
- RCON： localhost:25575 で管理コマンド（ローカル検証用）

---

## Docker Composeの読み方
Composeは「アプリを動かす箱（コンテナ）」を定義するファイルです。

よく出てくる要素と役割はこれ：

- image：コンテナの元になる“材料”
- ports：ホスト(mac) ↔ コンテナ を繋ぐ窓口
- environment：コンテナ内アプリの設定値（image側が解釈する）
- volumes：データをコンテナの外に逃がす（永続化）
- networks：コンテナ同士のLAN
- healthcheck：中身のアプリが生きてるかチェック

---

## ports（ポートマッピング）は何をしてる？
例えばこれ：

    ports:
      - "25565:25565"

意味はこうです。

- Mac（ホスト側）の 25565 番にアクセスしたら
- コンテナの 25565 番へ転送してね

なのでMinecraftクライアントは `localhost:25565` で入れます。

同様にRCONもこうなります：

    ports:
      - "25575:25575"

Macの25575 → コンテナの25575（RCON受付）になります。

---

## environment（環境変数）は「ただの変数」？
結論：YAML的にはただの文字列ですが、image側が“設定項目”として解釈するので意味があります。

例えば itzg/minecraft-server は、環境変数を見て server.properties を自動生成したり変更したりしてくれます。

例：

- EULA=TRUE → EULA同意済みとして起動
- VERSION=1.21.4 → そのバージョンのserver.jarを落として起動
- ENABLE_RCON=true → RCONを有効化
- RCON_PASSWORD=... → パスワード設定

つまり「環境変数 → 設定ファイルに反映」がimage内部で起きています。

---

## volumes / マウント / 永続化って何？
### コンテナの落とし穴
コンテナは気軽に作り直せるのがメリットですが、裏を返すとこうなります。

- コンテナ内に保存したデータは、コンテナを消すと一緒に消える

Minecraftサーバーだとワールドが消えるのは致命的です。

だから volumes を使って、ワールドデータを「コンテナの外」に保存します。

---

## 2種類のvolume
### ① Bind mount（ホストのフォルダ直結）
LocalStackで使ったのがこれ：

    volumes:
      - "./localstack-data:/var/lib/localstack"

意味：

- 自分のPCの `localstack-data/` に
- LocalStackの状態を保存する

メリット：

- PC側から見える（Finderでも確認できる）
- どこにデータがあるか分かりやすい

---

### ② Named Volume（Dockerが管理）
Minecraftで使ったのがこれ：

    volumes:
      - minecraft-data:/data

意味：

- minecraft-data というDocker管理の保存領域を作って
- コンテナ内の /data にくっつける（マウント）

/data にはワールドや設定が入るので、これで down→up してもワールドが残ります（永続化）。

確認コマンド：

    docker volume ls
    docker volume inspect minecraft-data

---

## 「マウント」って結局どういうこと？
自分の理解だと、マウントは「別の場所にある保存領域を、あるパスにくっつけて同じフォルダみたいに見せる」って感じです。

- Named Volume（Dockerが持ってる保存領域）を
- コンテナ内の /data に“接続”して
- コンテナからは /data を普通のフォルダとして扱える

だからMinecraftサーバーは /data/world に普通に保存してるだけなのに、実体はコンテナの外（volume）に残る、という挙動になります。

---

## LocalStackの「Dockerソケット共有」って何？
LocalStack側でこういう設定が出てきます。

    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"

これは「ホストのDockerを、コンテナ内から操作できるようにする」設定です。

LocalStackはLambdaの実行などで内部的にDockerを使うことがあり、そのために必要になる場合があります。

要するに：

- LocalStackコンテナの中から
- ホストのDockerエンジンを呼び出せるようにしている

※便利ですが強い権限になるので、本番では扱いに注意です。

---

## healthcheckって何？
healthcheck は「コンテナの中身のアプリが正常か」を定期的に確認する仕組みです。

例えばLocalStackなら health endpoint を叩いてOKかを見ます。

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4566/_localstack/health"]
      interval: 10s
      timeout: 5s
      retries: 3

Minecraftなら itzg/minecraft-server に `mc-health` が用意されていて、起動完了してるかをチェックできます。

---

## networkって何？
Composeで networks を作ると、同じネットワークにいるコンテナ同士が「LAN内のPC」みたいに通信できます。

メリット：

- IPじゃなくコンテナ名で通信できる
- 後々、監視用コンテナやBot用コンテナを足すときに便利

---

## RCONって結局なに？
RCONはDockerの機能ではなく、Minecraftサーバーの機能（Remote Console）です。

公式サーバー起動時に出る「ログが流れてコマンド打てるコンソール」がありますが、
RCONはその“コマンド操作部分”をネットワーク越しにできる仕組みです。

できること：

- list（接続中プレイヤー一覧）
- save-all（保存）
- say ...（全体チャット）
- stop（停止）

Phase1では、Discord Botで /server stop をやるために「RCONで save-all → stop が打てる」ことを確認しました。

実行例：

    docker exec -i minecraft-dev rcon-cli --password test
    # list
    # save-all
    # say Hello
    # exit

---

## AWS CLI設定ファイルが無いと言われた件（自分が詰まった）
AWS CLIの設定ファイルは最初から無いことがあります。なので「無いなら自分で作る」が正解でした。以下のコマンド作成しました。

    aws configure --profile localstack 

---

## Phase1のコマンド
LocalStack起動：

    docker-compose up -d --build 

LocalStackにAWS CLIで接続（例：S3）：

    aws --endpoint-url=http://localhost:4566 --profile localstack s3 ls
    aws --endpoint-url=http://localhost:4566 --profile localstack s3 mb s3://test-bucket

Minecraft起動：

    docker-compose --profile minecraft up -d --build

RCON起動：

    docker exec -i minecraft-local rcon-cli --password minecraft-dev

---

## Phase1で学んだこと（自分の理解）
- portsは「ホスト→コンテナへの転送」
- environmentは「ただの変数」ではなく、imageが設定として解釈する値
- volumesは「コンテナを消してもデータが残る」ための仕組み
- bind mount と named volume は別物（前者はPCのフォルダ、後者はDocker管理）
- healthcheckで「動いてる」だけじゃなく「正常か」が分かる
- RCONはMinecraftサーバーの遠隔操作口で、Discord運用に必須

---

## 次にやること（Phase2以降）
Phase1でローカル検証の土台ができたので、次はOpenTofuで VPC/EC2/S3/EIP を作って、LocalStackで一回回してから本番AWSに持っていく予定です。

また進んだら記事に残します！
