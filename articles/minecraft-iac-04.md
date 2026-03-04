---
title: "MinecraftサーバーをIaC化してみた【Part4】- Discord Bot実装編"
emoji: "🤖"
type: "tech"
topics:
  - "aws"
  - "lambda"
  - "discord"
  - "python"
  - "opentofu"
published: true
---

## この記事で書くこと

MinecraftサーバーをIaCで管理するプロジェクトのPart4です。

今回はDiscord Botを実装して、Discordのスラッシュコマンドでサーバーを操作できるようにしました。

- Discord Interactions APIの仕組みを理解する
- Lambda + Lambda Function URLでサーバーレスBotを構築する
- `/server start`, `stop`, `status`, `logs`, `backup` を実装する
- OpenTofuでLambdaをデプロイする
- **Deferred Responseパターン**でタイムアウト問題を解決する

実装中にかなりハマったので、詰まったポイントも含めて残します。

---

## 作ったものの全体像

```
Discord: /server status
    │
    ▼
Lambda（Receiver）
    ├── 署名検証（本物のDiscordか？）
    ├── 「考え中...」を即返す（3秒以内）
    └── Worker Lambdaを非同期起動

Lambda（Worker）
    ├── EC2の状態を確認
    ├── SSMでMinecraftコンテナに問い合わせ
    └── Discordに結果をPATCH送信
```

DiscordのスラッシュコマンドはHTTPで届くので、APIサーバーとして**Lambda + Function URL**を使います。Gateway APIのようにWebSocket常時接続は不要で、使った分だけ課金のサーバーレス構成にできます。

---

## ファイル構成

```
discord-bot/src/
    ├── handler.py         # Receiver Lambda（入口）
    ├── worker.py          # Worker Lambda（実際の処理）
    ├── commands/
    │   └── server.py      # /server コマンドの処理
    └── utils/
        ├── discord_utils.py  # Discord APIヘルパー
        ├── ec2.py            # EC2操作
        └── ssm.py            # SSMコマンド実行
```

---

## discord_utils.py - レスポンスを組み立てる関数たち

### IntEnumで数値に名前をつける

DiscordとのやりとりはすべてJSONです。リクエストには `"type": 2`（スラッシュコマンド）、レスポンスには `"type": 4`（メッセージを送る）のように数値が使われます。

この数値をそのままコードに書くと意味不明になるので、`IntEnum` で管理します。

```python
from enum import IntEnum

class InteractionType(IntEnum):
    PING = 1               # Discordからの疎通確認
    APPLICATION_COMMAND = 2 # スラッシュコマンド

class InteractionResponseType(IntEnum):
    PONG = 1                                    # PINGへの返事
    CHANNEL_MESSAGE_WITH_SOURCE = 4             # 通常のメッセージ
    DEFERRED_CHANNEL_MESSAGE_WITH_SOURCE = 5    # 「考え中...」
```

`IntEnum` を継承することで、名前で管理しながら数値としても使えます。

```python
# 意図が不明
if interaction_type == 2:

# 意図が明確
if interaction_type == InteractionType.APPLICATION_COMMAND:
```

これは「マジックナンバーを避ける」という設計原則の実践です。

### 2種類のレスポンス

Discordには「通常メッセージ」と「Embed（カード形式）」があります。

```python
def create_response(content: str, ephemeral: bool = False) -> dict:
    """ただのテキストメッセージ"""
    response = {
        "type": InteractionResponseType.CHANNEL_MESSAGE_WITH_SOURCE,
        "data": {"content": content}
    }
    if ephemeral:
        response["data"]["flags"] = 64  # 送信者にだけ見える
    return response

def create_embed_response(title, description, color=0x00ff00, fields=None, ...) -> dict:
    """タイトル・色・フィールドを持つカード形式"""
    embed = {"title": title, "description": description, "color": color}
    if fields:
        embed["fields"] = fields
    ...
```

使い分けはこうです。

- `create_embed_response()` → あらゆる結果表示に使う（情報量に関わらずEmbedで統一）
- `create_error_response()` → 赤色Embedのエラー専用ラッパー
- `create_deferred_response()` → 「考え中...」を返すDeferredパターン専用
- `create_response()` → 定義はしたが今回は未使用（将来的にシンプルな通知で使う想定）

---

## ec2_utils.py - EC2を操作する

`boto3.client("ec2")` でEC2を操作します。状態を取得・起動・停止するのに加えて、**Waiter**という仕組みを使います。

```python
def wait_for_instance_running(timeout: int = 300) -> bool:
    ec2 = get_ec2_client()
    waiter = ec2.get_waiter("instance_running")
    try:
        waiter.wait(
            InstanceIds=[EC2_INSTANCE_ID],
            WaiterConfig={"Delay": 15, "MaxAttempts": timeout // 15}
        )
        return True
    except Exception:
        return False
```

Waiterは「ある状態になるまで定期的に確認し続ける」boto3の機能です。自分でポーリングループを書く必要がなくなります。

`Delay: 15` は15秒ごとに確認、`MaxAttempts: timeout // 15` は最大試行回数です。`300 // 15 = 20` なので最大5分待ちます。

---

## ssm_utils.py - EC2の中でコマンドを実行する

EC2の中でコマンドを叩くのに、SSHではなくSSMを使います。

**SSHを使わない理由：**
- 秘密鍵の管理が不要
- 22番ポートを開放しなくていい
- IAMで権限管理できる

```python
def run_command(commands: List[str], timeout_seconds: int = 60) -> Dict[str, Any]:
    ssm = get_ssm_client()
    response = ssm.send_command(
        InstanceIds=[EC2_INSTANCE_ID],
        DocumentName="AWS-RunShellScript",  # シェルコマンドを実行するテンプレート
        Parameters={"commands": commands},
        TimeoutSeconds=timeout_seconds
    )
    command_id = response["Command"]["CommandId"]

    # ポーリングで完了を待つ
    for _ in range(timeout_seconds // 2):
        time.sleep(2)
        try:
            result = ssm.get_command_invocation(
                CommandId=command_id, InstanceId=EC2_INSTANCE_ID
            )
            if result["Status"] in ["Success", "Failed", "TimedOut", "Cancelled"]:
                return {"success": result["Status"] == "Success", ...}
        except ssm.exceptions.InvocationDoesNotExist:
            continue  # まだ結果が準備できていない→次のループへ
```

`send_command()` は命令を送るだけで完了を待ちません。なので `get_command_invocation()` で結果をポーリングします。送信直後は結果が存在しないので `InvocationDoesNotExist` 例外を `continue` でスキップするのがポイントです。

### RCONパスワードの落とし穴

```python
# ❌ $RCON_PASSWORD はEC2の環境変数 → 空になる
'docker exec minecraft-server rcon-cli --password "$RCON_PASSWORD" list'

# ✅ Lambda側の環境変数を展開してから渡す
RCON_PASSWORD = os.environ.get("RCON_PASSWORD", "")
f'docker exec minecraft-server rcon-cli --password "{RCON_PASSWORD}" list'
```

このコマンドはLambdaからSSM経由でEC2上で実行されます。`$RCON_PASSWORD` はEC2の環境変数を参照しますが、EC2側には設定されていないので空文字になります。LambdaのPythonコードでf文字列として展開してから渡す必要がありました。

---

## 最初のつまずき：「アプリケーションが応答しませんでした」

実装してデプロイしたら最初にこのエラーに遭遇しました。

CloudWatchを見ると：

```
08:33:01.828Z  Handling command: server  ← ここまで正常
08:33:06.185Z  END                       ← 4.3秒後に終了
```

**Discordの仕様：3秒以内にレスポンスがなければ失敗と判断します。**

`/server status` は `get_instance_status()` に加えて `get_minecraft_players()` でSSMコマンドを実行しており、その中に `time.sleep(2)` があったため4秒超えてしまいました。

---

## Deferred Responseパターンで解決する

「考え中...」を先に返して、後から結果を送る仕組みです。

### なぜ1個のLambdaでは解決できないのか

```python
def lambda_handler(event, context):
    return create_deferred_response()  # ← ここで return = Lambda終了
    # この後にコードを書いても実行されない
```

`return` した瞬間にLambdaは終了します。「返してから続きをやる」は1個では物理的に不可能です。

### Lambdaを2個に分ける

```
Receiver Lambda（既存を改修）
    ├── 署名検証
    ├── type:5（「考え中...」）を即 return ← 1秒以内
    └── Worker Lambdaを非同期起動（待たない）

Worker Lambda（新規作成）
    ├── EC2確認・SSM実行（時間かかってOK）
    └── Discord Webhookに結果をPATCH
```

### Receiver側

```python
# handler.py
lambda_client = boto3.client("lambda")
lambda_client.invoke(
    FunctionName=WORKER_FUNCTION_NAME,
    InvocationType="Event",  # ← 非同期。"Event"=待たない
    Payload=json.dumps(body).encode()
)

return {
    "statusCode": 200,
    "body": json.dumps(create_deferred_response())  # type:5を即返す
}
```

`InvocationType="Event"` が非同期呼び出しのキーワードです。`"RequestResponse"`（デフォルト）だとWorkerの完了を待ってしまいます。

### Worker側

```python
# worker.py
def send_followup_message(token: str, content: dict) -> None:
    url = (
        f"https://discord.com/api/v10/webhooks/"
        f"{DISCORD_APPLICATION_ID}/{token}/messages/@original"
    )
    data = json.dumps(content).encode("utf-8")
    req = urllib.request.Request(
        url, data=data, method="PATCH",
        headers={
            "Content-Type": "application/json",
            "User-Agent": "DiscordBot (https://github.com/minecraft-bot, 1.0.0)"
        }
    )
    with urllib.request.urlopen(req) as res:
        print(f"Discord webhook response: {res.status}")
```

`PATCH /messages/@original` で「考え中...」を結果に上書きします。`User-Agent` ヘッダーが**必須**です。これがないとCloudflare（Discordの前段のWAF）に `403: error code 1010` で弾かれます。

### Discordの Interaction Token

Workerが後からメッセージを更新するには、Discordのリクエストに含まれる `token` が必要です。

```json
{
  "type": 2,
  "token": "aW50ZXJhY3...",
  "data": {"name": "server", ...}
}
```

ReceiverからWorkerに `body` をそのまま渡すことで、WorkerもこのTokenを使えます。

---

## OpenTofu: Worker Lambda を追加する

```hcl
# Worker Lambda用のIAM権限（ReceiverがWorkerを呼べるようにする）
resource "aws_iam_role_policy" "discord_bot_invoke_worker" {
  ...
  policy = jsonencode({
    Statement = [{
      Action   = ["lambda:InvokeFunction"]
      Resource = aws_lambda_function.discord_bot_worker.arn
    }]
  })
}

# Worker Lambda
resource "aws_lambda_function" "discord_bot_worker" {
  function_name = "${local.name_prefix}-discord-bot-worker"
  handler       = "worker.lambda_handler"  # worker.py のlambda_handler
  runtime       = "python3.11"
  timeout       = 300  # 5分。SSM待機があるので長めに
  ...
  environment {
    variables = {
      DISCORD_APPLICATION_ID = var.discord_application_id
      EC2_INSTANCE_ID        = aws_instance.minecraft.id
      RCON_PASSWORD          = var.rcon_password
      ...
    }
  }
}
```

ReceiverのLambdaにも `WORKER_FUNCTION_NAME` を環境変数として追加します。

```hcl
# Receiver側の環境変数に追加
WORKER_FUNCTION_NAME = aws_lambda_function.discord_bot_worker.function_name
```

---

## 起動・停止完了を通知する

Deferredパターンを導入したことで、WorkerのLambdaタイムアウトが最大5分になりました。これを活かして「完全に起動/停止するまで待ってから通知する」実装にしました。

```python
# server.py（handle_start）
result = start_instance()
if not result["success"]:
    return create_error_response(result["message"])

# EC2がrunningになるまで待つ（最大5分）
running = wait_for_instance_running(timeout=300)

if running:
    new_status = get_instance_status()
    return create_embed_response(
        title="🟢 サーバー起動完了",
        description="Minecraft サーバーに接続できます！",
        fields=[{"name": "接続先", "value": f"`{new_status['public_ip']}:25565`"}]
    )
else:
    return create_embed_response(title="⏰ 起動タイムアウト", ...)
```

`/server start` を打つと「考え中...」が表示され、2〜3分後に「🟢 起動完了」に更新されます。

---

## 詰まったところまとめ

| 症状 | 原因 | 解決 |
|------|------|------|
| 「アプリケーションが応答しませんでした」 | Discordの3秒制限を超えた | Deferredパターンに移行 |
| `403: error code 1010` | `User-Agent`ヘッダーがない | `DiscordBot (url, version)` 形式を追加 |
| プレイヤー情報が常に「起動中/準備中」 | `$RCON_PASSWORD`がEC2で空になる | Lambda側でf文字列に展開してから渡す |
| `NameError: name 'boto3' is not defined` | importを書き忘れた | `import boto3` を追加 |
| コードを直しても反映されない | `tofu apply`が差分なしとスキップ | `tofu apply -replace=aws_lambda_function.xxx` |

---

## デプロイフロー

```bash
# コードを修正したら
tofu apply

# archive_fileリソースがsource_dirの変更を検知して
# ZIPを作り直し → source_code_hashが変われば自動デプロイ
```

`tofu apply` だけで反映されます。手動でZIPを作り直す必要はありません。

---

## 動作確認

```
/server status  → EC2状態・IPアドレス・プレイヤー数が表示される
/server start   → 「考え中...」→ 2〜3分後に「🟢 起動完了 + IP」に更新
/server stop    → 「考え中...」→ 1〜2分後に「🛑 停止完了」に更新
/server logs    → 直近10行のサーバーログが表示される
/server backup  → バックアップが実行される
```

---

## Phase4で学んだこと

**Discord API**

- Interactions APIはHTTPのリクエスト/レスポンス方式なのでサーバーレスと相性がいい
- リクエストには必ずEd25519署名が付いており、公開鍵で検証する
- 3秒ルールと、それを回避するDeferredパターン

**設計パターン**

- IntEnumでマジックナンバーを排除する
- Deferredパターン：「即レスポンス + 非同期処理 + 後から更新」
- `InvocationType="Event"` による非同期Lambda呼び出し

**Python**

- `List[str]`, `Dict[str, Any]` などの型ヒント
- デフォルト引数に可変オブジェクトを渡してはいけない（`fields=None` パターン）
- f文字列でコマンドの変数を展開してからEC2に渡す

**AWS**

- Lambda Function URLはAPI Gatewayなしで使えるシンプルなHTTPSエンドポイント
- SSMはSSHより安全（鍵不要・ポート開放不要）
- Waiterでポーリングの実装を省略できる

---

## 次にやること（Phase5）

- CloudWatch監視・アラート
- CI/CDパイプライン（GitHub Actions）
- ポートフォリオとしての整備

また進んだら記事にします！