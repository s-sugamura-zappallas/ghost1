以下に、Amazon Bedrock の `Converse` API を用いて AI エージェントが推論を行えるようにするための手順・サンプルコードをまとめた **Markdown ファイル**を提示します。  
このドキュメントを参考にしていただければ、AI エージェントが Python (Boto3) を介して Amazon Bedrock で推論 API (`Converse`) を呼び出せるようになるはずです。  

---

# Bedrock Converse API ドキュメント

## 目次

1. [概要](#概要)  
2. [前提条件](#前提条件)  
3. [環境変数 (`.env`) で AWS 認証情報を管理する例](#環境変数-env-で-aws-認証情報を管理する例)  
4. [Boto3 インストールと環境セットアップ](#boto3-インストールと環境セットアップ)  
5. [Converse API の基本構造](#converse-api-の基本構造)  
   1. [modelId](#modelid)  
   2. [messages](#messages)  
   3. [system (必須推奨)](#system-必須推奨)  
   4. [inferenceConfig](#inferenceconfig)  
   5. [その他のパラメータ](#その他のパラメータ)  
6. [サンプルコード](#サンプルコード)  
   1. [シンプルなテキスト会話例](#シンプルなテキスト会話例)  
   2. [ツールコール (toolUse) の例 (オプション)](#ツールコール-tooluse-の例-オプション)  
   3. [system プロンプトを含む完全版サンプル (リトライ機能つき)](#system-プロンプトを含む完全版サンプル-リトライ機能つき)  
7. [主なエラーと対処](#主なエラーと対処)  
8. [参考リンク](#参考リンク)  

---

## 1. 概要

Amazon Bedrock では、高性能な言語モデル (例: Claude, Amazon Titan Text, AI21, etc.) を、統一された API 経由で呼び出すことができます。  
**`Converse` API** は、マルチターンのチャット体験を提供するためのエンドポイントです。テキストメッセージをやり取りし、モデルによる応答を受け取ることができます。

このドキュメントでは、Python (Boto3) から Amazon Bedrock の `Converse` API を呼び出すための手順とサンプルコードを紹介します。

---

## 2. 前提条件

- **AWS アカウントと IAM ポリシー設定**  
  - `bedrock:InvokeModel` または `bedrock:*` 権限を持つ IAM ユーザー / ロールが必要です。  
  - Amazon Bedrock へのアクセス権 (ベータまたはGA対応) が付与されている必要があります。  

- **Python 3.7+ と Boto3**  
  - Python 3.7 以上の環境を推奨  
  - `boto3` がインストール済み (`pip install boto3`)  

- **(任意) AWS CLI 設定**  
  - `aws configure` 等で認証情報を設定しておくとスムーズ  
  - ただし、`.env` + `python-dotenv` などの方法でも可

---

## 3. 環境変数 (`.env`) で AWS 認証情報を管理する例

本番環境では AWS Secrets Manager 等の活用が望ましいですが、ローカル開発で簡易的に認証情報を読み込む方法として `.env` を利用する例を示します。

`.env` ファイルを作成して以下のように記述します (Git 管理から除外してください)。  

```bash
AWS_ACCESS_KEY_ID=<YOUR_ACCESS_KEY_ID>
AWS_SECRET_ACCESS_KEY=<YOUR_SECRET_ACCESS_KEY>
AWS_REGION=us-east-1
```

Python スクリプト内で `python-dotenv` を使って読み込むことができます:

```python
from dotenv import load_dotenv
import os

load_dotenv()
print(os.getenv("AWS_ACCESS_KEY_ID"))
print(os.getenv("AWS_SECRET_ACCESS_KEY"))
print(os.getenv("AWS_REGION"))
```

---

## 4. Boto3 インストールと環境セットアップ

1. **Boto3 のインストール**  
   ```bash
   pip install boto3
   ```

2. **(任意) python-dotenv のインストール**  
   ```bash
   pip install python-dotenv
   ```

3. **AWS CLI 設定 (参考)**  
   ```bash
   aws configure
   # AWS Access Key ID, Secret Access Key, region などを入力
   ```

---

## 5. Converse API の基本構造

### 5.1. `modelId`
`modelId` は、使用するモデルや推論設定を一意に特定する ID です。  
- `anthropic.claude-3-5-sonnet-20241022-v2:0`を使用すること
- Provisioned Throughput やカスタムモデル等の ARN を指定するケースもあります

### 5.2. `messages`
```python
messages = [
    {
        'role': 'user' or 'assistant',
        'content': [
            { 'text': '...' },
            # 画像やドキュメントなど、モデルがサポートする形式
        ]
    }
]
```
会話の履歴をリスト形式で渡します。  
- `role` が `user` ならユーザの発話、`assistant` は AI アシスタントの出力  
- `content` には複数のブロックを格納可能 (`text`, `image`, `document`, `toolUse`, 等)

### 5.3. **system (必須推奨)**

```python
system=[
    {"text": "You are a helpful AI assistant that responds concisely."}
]
```

- 会話全体に適用される指示や前提を設定する**システムプロンプト**です。  
- 例えば「あなたは専門的なプログラマーです」と指定すると、会話の文脈が技術寄りになる等、LLM の応答が変化します。  
- **ベストプラクティスとして必ず入れる**ことが推奨されます。

### 5.4. `inferenceConfig`
```python
inferenceConfig={
    "maxTokens": 512,
    "temperature": 0.5,
    "topP": 0.9,
    "stopSequences": [...]
}
```
- モデル推論時に利用する各種パラメータを指定します。  
- 未サポートのパラメータを指定すると `ValidationException` が発生する場合もあります。  

### 5.5. その他のパラメータ
- `guardrailConfig` : ガードレール(コンテンツフィルター)設定  
- `toolConfig` : ツールコール(関数呼び出し)設定  
- `additionalModelRequestFields` : モデル固有パラメータ  
- `promptVariables` : プロンプト管理機能を使う際の変数展開  
- など、必要に応じて使い分けてください。

---

## 6. サンプルコード

### 6.1. シンプルなテキスト会話例

```python
import boto3
from botocore.exceptions import ClientError

def simple_converse_example():
    # Bedrock Runtime クライアントを作成
    # (環境変数から認証情報やリージョンを取得する想定)
    bedrock_runtime = boto3.client("bedrock-runtime")

    response = None
    try:
        response = bedrock_runtime.converse(
            modelId="amazon.titan-text-express-v1",
            system=[{"text": "You are a concise, helpful AI assistant."}],
            messages=[
                {
                    "role": "user",
                    "content": [
                        {"text": "What is the purpose of a 'hello world' program?"}
                    ]
                }
            ],
            inferenceConfig={
                "maxTokens": 256,
                "temperature": 0.5,
                "topP": 0.9
            }
        )
    except ClientError as e:
        print("AWS Bedrock Converse Error:", e)
        return

    if response:
        print("Stop reason:", response.get("stopReason", "N/A"))
        model_message = response["output"]["message"]
        if "content" in model_message:
            # 仮に複数ブロックがある場合は先頭を使用
            text_block = model_message["content"][0]
            print("Model response text:", text_block["text"])

if __name__ == "__main__":
    simple_converse_example()
```

### 6.2. ツールコール (`toolUse`) の例 (オプション)

一部モデル (Anthropic Claude 3 など) は、関数呼び出し (tool use) をサポートします。  
**サンプル**: モデルが計算機 (calculator) を呼び出したいケースを想定。

```python
conversation = [
    {
        "role": "assistant",
        "content": [
            {
                "toolUse": {
                    "toolUseId": "calc123",
                    "name": "calculator",
                    "input": {"json": {"expression": "2 + 2"}}
                }
            }
        ]
    },
    {
        "role": "assistant",
        "content": [
            {
                "toolResult": {
                    "toolUseId": "calc123",
                    "content": [{"json": 4}],
                    "status": "success"
                }
            }
        ]
    }
]
```

- 実際にどうツールを呼び出すか、どのように結果をモデルに返すかはユースケースに依存します。  
- `toolConfig` でツール利用を有効化する必要がある場合もあります。

### 6.3. system プロンプトを含む完全版サンプル (リトライ機能つき)

```python
import time
import logging
import boto3
from botocore.exceptions import ClientError

logger = logging.getLogger(__name__)

def convert_chat_messages_to_conversation(messages):
    """
    ユーザー/アシスタントの発話リストを
    Bedrock Converse API用の形式に変換するサンプル関数
    """
    conversation = []
    for msg in messages:
        conversation.append({
            "role": msg["role"],  # "user" or "assistant"
            "content": [
                {"text": msg["text"]}
            ]
        })
    return conversation

def converse_with_retries(system_prompt, messages, model_id, region="us-east-1", max_retries=6):
    """
    system_prompt と user/assistant メッセージを用いて
    Converse API を呼び出し、必要に応じて指数関数的リトライを行うサンプル。
    """
    runtime = boto3.client("bedrock-runtime", region_name=region)
    conversation = convert_chat_messages_to_conversation(messages)

    backoff = 0.5
    for attempt in range(1, max_retries + 1):
        try:
            response = runtime.converse(
                modelId=model_id,
                system=[{"text": system_prompt}],
                messages=conversation,
                inferenceConfig={
                    "maxTokens": 512,
                    "temperature": 0.0,
                    "topP": 0.9
                },
            )
            logger.info(f"Converse call succeeded. Usage: {response['usage']}")
            return response
        except ClientError as e:
            logger.error(f"Attempt {attempt}, ClientError: {e}")
            if attempt == max_retries:
                raise
        except Exception as e:
            logger.error(f"Attempt {attempt}, Unexpected Error: {e}")
            if attempt == max_retries:
                raise

        # リトライ待機
        time.sleep(backoff)
        backoff *= 2.0

    raise RuntimeError("Failed to call Converse API after retries.")

# 実行例
if __name__ == "__main__":
    messages_sample = [
        {"role": "user", "text": "Explain the concept of 'Hello World' program in simple terms."}
    ]
    system_text = "You are a helpful AI assistant who provides clear, concise explanations."
    try:
        resp = converse_with_retries(
            system_prompt=system_text,
            messages=messages_sample,
            model_id="anthropic.claude-v2",  # Claudeモデル例
            region="us-east-1"
        )
        # 応答を取り出す
        ai_message = resp["output"]["message"]
        if "content" in ai_message:
            print("[AI Reply]:", ai_message["content"][0]["text"])
        else:
            print("No content found in AI message.")
        print("Usage info:", resp.get("usage", {}))
    except Exception as e:
        print("Error:", e)
```

---

## 7. 主なエラーと対処

1. **`AccessDeniedException`**  
   - IAM ポリシーが不足しているか、モデルへのアクセス権限が未付与。  
   - `bedrock:InvokeModel` or `bedrock:*` を確認。

2. **`ResourceNotFoundException`**  
   - `modelId` (もしくは ARN) が誤っている可能性。  
   - または、該当モデルが現在のリージョンに存在しない/サポートしていない場合も。

3. **`ValidationException`**  
   - パラメータに不備。モデルがサポートしていないパラメータを `inferenceConfig` 等に指定している場合や JSON 構造が不正な場合。

4. **`ThrottlingException` / `ServiceUnavailableException`**  
   - 同時アクセス数や API レート制限を超過している可能性。  
   - リトライ実装やレート制御を検討する。

5. **`ModelTimeoutException` / `ModelNotReadyException`**  
   - 大量の入力やコンテンツサイズ、モデル側の負荷増大によるタイムアウト。  
   - 入力サイズの削減やリトライ間隔の調整を検討。

---

## 8. 参考リンク

- [Amazon Bedrock ユーザーガイド - Converse API](https://docs.aws.amazon.com/bedrock/latest/userguide/conversation.html)  
- [AWS API Reference - Converse](https://docs.aws.amazon.com/bedrock/latest/APIReference/API_Converse.html)  
- [Use a guardrail with the Converse API](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails.html)  
- [Boto3 ドキュメント](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)  

---

以上の手順・コードを参考に、**system プロンプト**を必ず設定し、`messages` 配列とともに `Converse` API に送信することで、AI エージェントが Amazon Bedrock 上の言語モデルを呼び出せるようになります。  

本ドキュメントをもとに、各プロジェクトの要件に合わせてパラメータや構成を調整してご利用ください。