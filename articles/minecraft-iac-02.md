---
title: "MinecraftサーバーををIaC化してみた【Part2】- OpenTofuでAWSをデプロイ"
emoji: "🏝️"
type: "tech"
topics:
  - "aws"
  - "terraform"
  - "minecraft"
  - "localstack"
  - "opentofu"
published: true
published_at: "2026-03-03 16:26"
---

## この記事で書くこと
MinecraftサーバーをIaCで作るプロジェクトのPart2として、**OpenTofuでAWSにインフラをデプロイ**しました。

Part1ではローカルで

- LocalStack（ローカルAWS）をDockerで起動
- MinecraftサーバーをDockerで起動
- RCONで操作できることを確認
- volumesでワールドが消えない（永続化）ことを確認

までやりました（Part1の記事は別に書いてます）。

Part2はその続きで、

- OpenTofuでVPC/EC2/S3/IAMをコード化
- plan → apply の流れを回す
- **SSMでSSHなし接続**を目指す
- 途中で詰まったので、学びと「何が原因だったか」をまとめる

って感じです。

初学者の備忘録なので、頭悪そうでも許してください、、

---

## Part2で作ったもの（目標と成果）
### 目標
「AWS上にMinecraftサーバーを立てる土台（インフラ）をコードで再現できる状態」にする。

### 実際に作ったもの
- VPC（自分専用ネットワーク）
- Public Subnet（EC2を置く場所）
- Internet Gateway + Route Table（インターネットに出られるように）
- Security Group（25565だけ開ける、RCONは絞る）
- EC2（Amazon Linux 2023）+ EBS（ストレージ）
- Elastic IP（固定IP）
- S3（バックアップ置き場）+ ライフサイクル（自動削除）
- IAM Role / Instance Profile（EC2にS3アクセス権限 + SSM権限）
- そして **SSM Session Managerで接続**

---

## 全体像（ざっくりアーキテクチャ）
「Part2時点」なので、Discord BotとかAnsibleはまだ前提にしません（それはPart3/4）。

```text
Internet
  |
  v
[IGW] ----> (RouteTable: 0.0.0.0/0 -> IGW)
  |
  v
VPC (10.0.0.0/16)
  |
  v
Public Subnet (10.0.1.0/24)
  |
  v
EC2 (Amazon Linux 2023)  --- EBS(gp3)
  |
  +--- Security Group: 25565/tcp open to 0.0.0.0/0
  |
  +--- (SSM Agent) -> AWS SSM endpoints (443)
  |
  +--- backup -> S3 bucket
```

---

## OpenTofuで「AWSをコードにする」ってどういうこと？
正直、Partまで「AWSをコンソールで触る」側のイメージだったので、OpenTofuの考え方が一番大きい学びでした。

- `resource` = 作るもの
- `data` = 参照するもの（AMI検索とか）
- `plan` = 何が変わるかのプレビュー（事故防止）
- `apply` = 実行
- `state` = “現状はこう”を覚えてるファイル（これが壊れると地獄）

---

## 勉強になったポイント（コードの要点）
### 1) AMIを固定しない（でも勝手に作り直さない）
Amazon Linux 2023のAMIを `data` で「最新検索」してました。

```hcl
data "aws_ami" "amazon_linux_2023" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

でもこれだけだと「AMIが新しくなった」＝「差分」になって、意図せずEC2が作り直されることがあるので、

```hcl
lifecycle {
  ignore_changes = [ami]
}
```

を入れて **勝手に再作成されない**ようにしました。

---

### 2) user_dataは「初回起動時の自動セットアップ」
EC2起動直後に実行するスクリプト。Docker入れたり、ディレクトリ作ったり。

```hcl
user_data = base64encode(<<-EOF
  #!/bin/bash
  exec > >(tee /var/log/user-data.log) 2>&1

  dnf update -y
  dnf install -y docker
  systemctl enable docker
  systemctl start docker

  mkdir -p /opt/minecraft
  chown ec2-user:ec2-user /opt/minecraft
EOF
)
```

**学び：** user_dataは「再起動では走らない」。基本「初回だけ」なので、修正したらリソース作り直し（or cloud-initや別手段）が必要。

---

### 3) Security Groupの感覚（CIDR / tcp / Ingress/Egress）
Part2で一番「インフラっぽい知識が増えた」と思ったのはここ。

- `cidr_ipv4 = "0.0.0.0/0"` = 全世界に公開
- `ip_protocol = "tcp"` = TCPだけ許可
- `egress` 全許可は “よくある”

Minecraftは25565/TCPを開ける。

```hcl
resource "aws_vpc_security_group_ingress_rule" "minecraft_game" {
  security_group_id = aws_security_group.minecraft.id
  from_port   = 25565
  to_port     = 25565
  ip_protocol = "tcp"
  cidr_ipv4   = "0.0.0.0/0"
}
```

RCONは危険なので、VPC内に絞る（後でSSM経由で叩く想定）

```hcl
resource "aws_vpc_security_group_ingress_rule" "minecraft_rcon" {
  security_group_id = aws_security_group.minecraft.id
  from_port   = 25575
  to_port     = 25575
  ip_protocol = "tcp"
  cidr_ipv4   = var.vpc_cidr
}
```

---

### 4) metadata_options と monitoring は「セキュリティと監視の話」
- `monitoring = false`：詳細監視（1分粒度）をオフ（お金も絡む）
- `metadata_options`：IMDSv2必須にしてセキュリティ強める

```hcl
metadata_options {
  http_endpoint               = "enabled"
  http_tokens                 = "required"
  http_put_response_hop_limit = 1
}
```

ここは「雰囲気でコピペ」になりがちだけど、意味を理解すると一気に “インフラやってる感” が出る。

---

## Part2で一番詰まったところ
ここからが本題。Part2は「インフラは作れたけど、運用と接続で詰む」フェーズでした。

### 0) LocalStackに繋がらない（`localhost:4566 connection refused`）
最初の頃、`tofu plan` が

- `http://localhost:4566/` に繋げず落ちる

っていうエラーが出て、何だこれ…ってなりました。

原因は単純で
- LocalStackが起動してない
- そもそもポート公開してない
- コンテナ内でtofuを実行してて `localhost` が別物

みたいなやつ。

**学び：** 「ローカルAWS」って言っても、裏ではDockerのネットワークがあるので `localhost` の意味が変わる。

---

### 1) AWS認証で沼（`InvalidClientTokenId`）
`aws sts get-caller-identity` が失敗して、何回も詰みました。

特にやらかしたのが

- `AWS_ACCESS_KEY_ID=test` が環境変数に残ってた
- `AWS_PROFILE=minecraft-prod` を指定してるつもりでも、環境変数が優先されてた

結果、profileのキーを正しく入れても `test/test` を掴んで落ちました。

**学び：** AWS CLI / SDKは **環境変数が最優先**。  
困ったらまずこれを見る。

```bash
env | grep '^AWS_'
aws configure list --profile minecraft-prod
aws sts get-caller-identity --profile minecraft-prod
```

---

### 2) LocalStackのstateが本番AWSに混ざって壊れた（SG ID malformed）
これが個人的に一番びっくりした。

`InvalidGroupId.Malformed: Invalid id: "sg-..."` って出て、
「え、AWSが返したIDをAWSが無効って言うの？」って混乱。

原因はだいたい

- LocalStackで作ったIDがstateに残ってる
- そのstateのまま本番AWSへ向けて `refresh` しようとして壊れる

**学び：** `state` は“世界線”。LocalStackとAWSを混ぜると事故る。

対処は「stateから外して作り直し」みたいなことをやりました。

```bash
tofu state show aws_security_group.minecraft
tofu state rm aws_security_group.minecraft
tofu apply
```

---

### 3) S3 lifecycleのWarning（filter/prefix必須）
S3 lifecycle設定で

> `rule[0].filter` or `rule[0].prefix` をどっちか1つ入れろ

というWarningが出ました（今は警告、将来エラー）。

「全オブジェクト対象」でも、明示しないと怒られる。

```hcl
rule {
  id     = "expire-old-backups"
  status = "Enabled"

  filter {
    prefix = ""
  }

  expiration {
    days = var.backup_retention_days
  }
}
```

**学び：** Providerの仕様が変わるので、Warningは早めに潰す。

---

### 4) SSMが繋がらない（TargetNotConnected）
Part2の最終目標の一つが「SSHを開けずにSSMで入る」でした。

状態はこう：

- EC2はrunning
- Public IPもある
- ルートテーブルもIGW向いてる
- Security Groupのegressも開いてる
- IAM roleにも `AmazonSSMManagedInstanceCore` 付いてる

なのに

```bash
aws ssm describe-instance-information \
  --filters Key=InstanceIds,Values=i-xxxx \
  --region ap-northeast-1 \
  --profile minecraft-prod
```

が `[]`（空）で、`start-session` は `TargetNotConnected`。

ここで学んだのは「SSMは“サーバー側もクライアント側も”準備がいる」ということ。

#### (A) EC2側：SSM Agentが必要
「Amazon Linux 2023なら入ってるはず」って思ってたけど、AMIや構成によって入ってない（or serviceが無い）ケースがありました。

最終的に user_data で明示的に入れるのが一番確実でした。

```bash
dnf install -y amazon-ssm-agent
systemctl enable --now amazon-ssm-agent
```

#### (B) ローカル側：Session Manager Plugin が必要
SSMで入ろうとしたら次はこれ。

> `SessionManagerPlugin is not found`

つまり、**自分のMacにプラグインが入ってない**。

```bash
brew install session-manager-plugin
session-manager-plugin --version
```

#### (C) 何が正しいか確認する手順（ここだけ覚えた）
SSMで詰んだら、とりあえずこの順で潰すのが良かった。

1. EC2が存在するか（本物のInstanceIdか）
2. IAM roleが付いてるか
3. そのroleに `AmazonSSMManagedInstanceCore` が付いてるか
4. サブネットのRouteが `0.0.0.0/0 -> igw` か
5. SG egress が開いてるか
6. `describe-instance-information` に載るか
7. ローカルに plugin があるか

---

## Part2で学んだこと（まとめ）
箇条書きだけど、自分の中で大きかったのはこのへん。

- OpenTofuの `plan → apply` は想像以上に事故防止になる
- `state` はめちゃ大事。LocalStackとAWSを混ぜると壊れる（世界線がズレる）
- AWS認証は「profile設定したのにダメ」より「正しいprofileを使えてるかを意識」
- Security Groupは “ポートを開ける” じゃなくて「必要な通信だけ通す」考え方
- SSMは「IAMとネットワークが正しくても、Agentが無ければ繋がらない」
- 逆に、Agentが動いてても「ローカルにpluginが無い」と繋がらない

---

## 次にやること（Part3以降）
Part2で「AWS上に箱を用意」できたので、次は

- AnsibleでDocker/Docker Composeをちゃんと整える
- MinecraftコンテナをEC2で動かす（compose配置）
- バックアップ（S3）とバージョンアップ（安全手順）を自動化

へ進む予定です。

また進んだら記事に残します！

---

## おまけ：Part2で使いがちだったコマンド集
（自分用メモ）

```bash
# AWS認証が生きてるか
aws sts get-caller-identity --profile minecraft-prod

# EC2が生きてるか
aws ec2 describe-instances --instance-ids i-xxxx --profile minecraft-prod --region ap-northeast-1

# SSM管理対象に載ってるか
aws ssm describe-instance-information \
  --filters Key=InstanceIds,Values=i-xxxx \
  --region ap-northeast-1 \
  --profile minecraft-prod

# SSMで入る（プラグイン必要）
aws ssm start-session --target i-xxxx --profile minecraft-prod --region ap-northeast-1

# ルートテーブル確認（Public subnetのIGWルート）
aws ec2 describe-route-tables \
  --filters "Name=association.subnet-id,Values=subnet-xxxx" \
  --region ap-northeast-1 \
  --profile minecraft-prod \
  --query "RouteTables[].Routes[]"
```