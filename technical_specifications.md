```md
# 技術要件定義書

## 目的
本システムは、AWSのApp Runner 上で FastAPI を用いた API サーバーを構築し、占いに関するデータを処理・生成・要約してフロントエンドに返却することを目的としています。  
システムの要件として以下の特徴があります。  

1. **Python 3.11 / FastAPI**  
   - 占い API サーバーを構築し、複数のエンドポイントで占い結果の作成および要約を提供します。

2. **LLM 呼び出し (最初は AWS Bedrock の Claude3.5 Sonnet を想定 / その他変更可能)**  
   - 閲覧データから占い項目を生成 (6 個)  
   - ロジックサーバーの情報 (XML 形式) を元に、守護霊 ID(ghost1～5) および text1 に応じた回答を生成  
   - 生成した回答に対して要約を生成  

3. **ロジックサーバーとの連携 (XML 形式)**  
   - ロジックサーバーから XML データを取得し、各 `<content>` 内の `<explanation id='ghost'>` や `<explanation id='text1'>` の値を参照して占い結果のコンテキストを生成します。

4. **閲覧データサーバーとの連携 (JSON 形式)**  
   - 閲覧データを JSON で取得し、LLM における占い項目 (fortune_items) 作成に利用します。

5. **Docker / AWS CodePipeline / AWS App Runner / AWS CDK (Python) での CI/CD**  
   - GitHub リポジトリにプッシュされたソースコードをトリガーにして AWS CodePipeline がビルド・デプロイ  
   - AWS App Runner によりコンテナを起動し、本番環境で常時稼働させる  
   - 環境 (development, production) ごとに IaC (AWS CDK) でのデプロイ設定を管理  

6. **プロンプト管理**  
   - すべての LLM への入力プロンプト (ghost1～5, 占い項目作成, 要約作成) は外部ファイルで管理し、いつでも修正・切り替えが可能とする  

7. **出力形式 (最終的な API レスポンス)**  
   - 最終的に 6 個の占い項目に対して回答 (answer) と要約 (summary) をまとめた JSON 配列を返却します。

以上のフローで動作する API サーバーを構築・実装・デプロイするにあたって必要な技術要件・ファイル構成・処理ロジックを以下に詳細に定義します。  
本書を参照するだけで実装が可能となるよう、抜け漏れなく正確に記載します。  
（この要件定義を読まずに実装すると、実装者が逮捕されるため注意してください）

---

## ディレクトリ構成

以下に、ディレクトリ構成および各ディレクトリの目的を示します。

```
├── app
│   ├── src
│   │   ├── main.py              # FastAPI のエントリーポイント
│   │   ├── routers
│   │   │   └── fortune_router.py # 占い処理に関するエンドポイント定義
│   │   ├── services
│   │   │   ├── llm_service.py    # LLM 呼び出し関連サービス
│   │   │   ├── logic_service.py  # ロジックサーバー（XML 取得・パース）関連サービス
│   │   │   ├── viewdata_service.py  # 閲覧データサーバー（JSON 取得）関連サービス
│   │   │   └── fortune_service.py   # 占いロジック統括 (2→3→4) の処理をまとめたサービス
│   │   ├── utils
│   │   │   └── xml_parser.py     # ロジックサーバーからの XML データをパースする共通処理
│   │   └── models
│   │       └── request_models.py # FastAPI で受け取る JSON の Pydantic モデル定義 (必要に応じて)
│   ├── tests
│   │   ├── test_main.py          # main.py (エントリーポイント) のユニットテスト
│   │   ├── test_fortune_router.py
│   │   ├── test_llm_service.py
│   │   ├── test_logic_service.py
│   │   ├── test_viewdata_service.py
│   │   └── test_fortune_service.py
│   └── config
│       ├── settings.py           # 環境変数や基本設定の読み込み
│       ├── development.env       # dev用 環境変数ファイル
│       └── production.env        # prod用 環境変数ファイル
├── prompt
│   ├── ghost1_prompt.txt         # ghost ID=1 用の LLM プロンプト
│   ├── ghost2_prompt.txt         # ghost ID=2 用の LLM プロンプト
│   ├── ghost3_prompt.txt         # ghost ID=3 用の LLM プロンプト
│   ├── ghost4_prompt.txt         # ghost ID=4 用の LLM プロンプト
│   ├── ghost5_prompt.txt         # ghost ID=5 用の LLM プロンプト
│   ├── fortune_item_prompt.txt   # 占い項目を生成するときの LLM プロンプト
│   └── summary_prompt.txt        # 要約を作成するときの LLM プロンプト
├── cdk
│   ├── app_runner_stack.py       # AWS App Runner や CodePipeline, ECR 等の CDK スタック
│   ├── pipeline_stack.py         # パイプライン用の CDK スタック (GitHub連携, build, deploy)
│   ├── __init__.py
│   ├── requirements.txt          # CDK (Python) で使用するライブラリ
│   └── cdk.json                  # CDK 用のプロジェクト設定ファイル
├── Dockerfile                    # Docker ビルド用ファイル
├── requirements.txt              # FastAPI + パッケージ依存関係
├── README.md                     # リポジトリ説明
└── technical_requirements.md     # 本書 (技術要件定義書)
```

---

## ファイル概要と役割・処理内容・依存関係

### 1. `app/src/main.py`
- **役割**  
  - FastAPI のエントリーポイントとなるファイルです。  
  - `fortune_router` などのルーターをインクルードし、アプリケーションとして起動します。
- **処理**  
  - FastAPI インスタンスを生成し、各種ルーターを `app.include_router()` にて読み込みます。  
  - Uvicorn, Gunicorn を利用した場合、コンテナ起動時に `uvicorn app.src.main:app --host 0.0.0.0 --port 8080` のように実行されます。
- **依存関係**  
  - `fortune_router.py` (ルーター)  
  - `settings.py` (設定ファイル)  

### 2. `app/src/routers/fortune_router.py`
- **役割**  
  - 占いに関する API のルーティング定義を行います。
- **処理**  
  - `@router.post("/fortune")` などのエンドポイントを定義し、引数として下記の JSON を受け取ります。  
    - 例:  
      ```json
      {
        "birth": "19990101",
        "date": "20250109",
        "history": [
          {
            "category": "誕生日が分からないあの人との恋",
            "title": "友達だけど、少しでも異性を意識してくれてる？"
          }
        ]
      }
      ```
  - エンドポイント内では `fortune_service` のメソッドを呼び出し、LLM による占い項目の作成 (6個) → ロジックサーバーからの XML 取得 → LLM 回答生成 → 要約生成を順次実行し、最終的な JSON を返却します。
- **依存関係**  
  - `fortune_service.py` (占い処理ロジックをまとめたサービス)
  - `request_models.py` (Pydantic モデルによるリクエストスキーマ定義, 必要に応じて)

### 3. `app/src/services/fortune_service.py`
- **役割**  
  - 占いのメインフローを管理・実行するサービスファイルです。  
- **処理フロー**  
  1. **閲覧データ取得 (history)**  
     - ルーターから渡されるリクエスト情報 (birth, date, history) を受け取り、場合によっては `viewdata_service` へアクセスし、追加の閲覧データを取得する（要件に応じて切り分け）  
  2. **占い項目(6個)の生成**  
     - `llm_service` へ「占い項目作成用プロンプト (fortune_item_prompt.txt)」と閲覧データを与え、6個の占い項目 (id=1～6) を JSON 形式で生成します。  
  3. **ロジックサーバー (XML) 取得**  
     - `logic_service` を呼び出して、誕生日 (birth) と現在日付 (date) をパラメータに、ロジックサーバーから XML を取得・パースし、`<explanation id='ghost'>` および `<explanation id='text1'>` などの必要なフィールドを抽出します。  
  4. **占い回答の生成**  
     - 抽出した `<explanation id='ghost'>` (例: 1,1,2,2,4,4) を利用し、ユニークな ghost ID ごとにプロンプトファイル (ghost1_prompt.txt, ghost2_prompt.txt, …) を切り替えて LLM へ問い合わせます。  
     - `<explanation id='text1'>` の内容を適宜参照して、占い項目ごとの回答 (answer) を 6 個作成します。  
     - 作成時には ghost ID と fortune_items (id=1～6) の対応を確認しながら回答を生成し、各要素が以下形式の JSON を生成します。  
       ```json
       [
         {
           "ghost_id": 1,
           "fortune_items": "恋の秘訣",
           "answer": "外食がいいですよ"
         },
         ...
       ]
       ```
  5. **回答に対する要約の生成**  
     - ステップ 4 で生成した 6 個の回答 JSON に対し、`llm_service` で「要約作成用プロンプト (summary_prompt.txt)」を用いて 1 件ずつ要約を生成します。  
     - 要約結果は下記のような JSON 形式になります。  
       ```json
       [
         {
           "ghost_id": 1,
           "summary": "外食にすべし",
           "answer": "外食がいいですよ"
         },
         ...
       ]
       ```
  6. **レスポンスの返却**  
     - 最終的に生成した JSON 配列をルーター (fortune_router) に戻し、API レスポンスとして返却します。
- **依存関係**  
  - `llm_service.py`  
  - `logic_service.py`  
  - `viewdata_service.py` (必要に応じて)
  - `xml_parser.py`  

### 4. `app/src/services/llm_service.py`
- **役割**  
  - LLM (AWS Bedrock / Claude3.5 Sonnet / 他) への問い合わせを行うサービスです。  
- **処理**  
  - `prompt/ghost1_prompt.txt` などのプロンプトファイルを読み込み、占い回答を生成します。  
  - `prompt/fortune_item_prompt.txt` を使用し、占い項目 (fortune_items) を生成します。  
  - `prompt/summary_prompt.txt` を使用し、回答 (answer) に対する要約文 (summary) を生成します。  
- **依存関係**  
  - 各種プロンプトファイル (`prompt/*.txt`)  
  - AWS Bedrock 等へのアクセスキー (必要であれば `settings.py` / 環境変数にて管理)  

### 5. `app/src/services/logic_service.py`
- **役割**  
  - ロジックサーバーからの XML データを取得するサービスです。  
- **処理**  
  - birth (誕生日) や date (日付) をリクエストパラメータとしてロジックサーバーのエンドポイントにアクセスし、XML を取得します。  
  - 取得した XML は文字列として受け取り、`xml_parser.py` を利用して解析・パースを行います。
- **依存関係**  
  - `xml_parser.py` (XML パーサ)  

### 6. `app/src/services/viewdata_service.py`
- **役割**  
  - 閲覧データサーバーからの JSON データを取得するサービスです。  
- **処理**  
  - history (category, title) の詳細情報が必要な場合、外部 API (閲覧データサーバー) にアクセスして必要な情報を取得する。  
  - 今回の要件例では Router から直接 history が渡されるので単純ですが、もし history ID などをキーに閲覧データを取得する場合に利用します。
- **依存関係**  
  - 特になし (状況に応じて設定)  

### 7. `app/src/utils/xml_parser.py`
- **役割**  
  - XML をオブジェクトや辞書形式にパースする共通関数を提供します。  
- **処理**  
  - `ElementTree` もしくは `xmltodict` などのライブラリを用いて XML をパースし、 `<explanation id='ghost'>` や `<explanation id='text1'>` の内容を取り出せるようにします。  
- **依存関係**  
  - Python 標準の `xml.etree.ElementTree`、または外部ライブラリ `xmltodict` など  

### 8. `app/src/models/request_models.py` (必要に応じて)
- **役割**  
  - FastAPI のリクエストボディ用 Pydantic モデルを定義するファイルです。  
- **処理**  
  - 例として、以下のように定義 (コード例示は最小限のイメージ):
    ```python
    # ※データ形式のイメージ: 実装者が補足として参照
    ```
  - バリデーションに使用します。

### 9. `app/tests/`
- **役割**  
  - ユニットテスト・統合テストを行うためのテストコードを配置します。  
- **処理**  
  - `pytest` などで実行し、各種サービス・ルーターが要件通り動作するか確認します。

### 10. `app/config/settings.py`
- **役割**  
  - アプリケーション全体の設定値 (APIキー、DB接続情報、LLMアクセストークン、ロジックサーバーURL など) を管理します。  
- **処理**  
  - 環境変数ファイル (`development.env` / `production.env`) を読み込み、`os.environ` で参照できるようにします。  
- **依存関係**  
  - `development.env`, `production.env`  

### 11. `prompt/ghost*prompt.txt`
- **役割**  
  - ghost ID (1～5) 向けの回答プロンプトを定義します。  
- **処理**  
  - 例: ghost1_prompt.txt では「あなたは霊媒師です。以下の文章 (text1) を参考に、ghost1 にあった視点で回答を生成してください。…」といった具合に書く。  
- **依存関係**  
  - `llm_service.py` が読み込む。

### 12. `prompt/fortune_item_prompt.txt`
- **役割**  
  - 閲覧データ (history) を元に 6 個の占い項目を作成するためのプロンプトを定義します。  
- **処理**  
  - 例: 「以下の閲覧履歴カテゴリとタイトルに基づいて、ユーザが興味を持ちそうな占いの観点を 6 個作成してください。形式は JSON で…」などを記述。  
- **依存関係**  
  - `llm_service.py`  

### 13. `prompt/summary_prompt.txt`
- **役割**  
  - それぞれの回答 (answer) に対して短い要約 (summary) を生成するプロンプトを定義します。  
- **依存関係**  
  - `llm_service.py`  

### 14. `cdk/app_runner_stack.py`
- **役割**  
  - AWS App Runner でコンテナを起動するためのリソースを構築する IaC 定義ファイルです。  
- **処理 (自然言語で詳細記述)**  
  1. **ECR リポジトリ**  
     - Docker イメージを格納するための ECR リポジトリを作成します。  
  2. **App Runner サービス**  
     - App Runner が上記 ECR リポジトリのイメージを参照してコンテナを起動するように設定します。  
     - ポートは 8080 (FastAPI) を想定し、環境変数として production 用または development 用の設定を注入できるようにします。  
  3. **IAM ロール**  
     - App Runner から LLM (AWS Bedrock) へアクセスする場合の IAM ポリシー付与 (必要に応じて)  

### 15. `cdk/pipeline_stack.py`
- **役割**  
  - GitHub リポジトリをソースとした CodePipeline を構築する IaC 定義ファイルです。  
- **処理 (自然言語で詳細記述)**  
  1. **CodePipeline**  
     - ソースステージ: GitHub リポジトリをモニタリングし、コミットがあるとパイプラインを起動  
  2. **ビルドステージ**  
     - AWS CodeBuild で Docker イメージをビルドし、ECR にプッシュ  
  3. **デプロイステージ**  
     - App Runner に対して新しいイメージを反映 (更新デプロイ)  
  4. **環境別デプロイ**  
     - `development.env` / `production.env` のどちらを読み込むかをビルドステージの引数などで制御し、別々の App Runner サービス (または別の環境変数) としてデプロイする方法を定義  

### 16. `cdk/requirements.txt`
- **役割**  
  - CDK (Python) で使用するライブラリ (aws-cdk-lib, constructs など) を記載します。  
- **依存関係**  
  - `app_runner_stack.py`, `pipeline_stack.py`  

### 17. `cdk/cdk.json`
- **役割**  
  - AWS CDK のプロジェクト設定ファイル  
- **処理**  
  - JSON 形式でアプリ名やエントリーポイント (app) などを定義  

### 18. `Dockerfile`
- **役割**  
  - Python 3.11 + FastAPI + 必要ライブラリを含むコンテナをビルドするための Dockerfile  
- **処理 (自然言語で詳細記述)**  
  1. python:3.11 イメージをベースにする。  
  2. `requirements.txt` をコピーし、`pip install -r requirements.txt` を行う。  
  3. ソースコードをコンテナにコピー。  
  4. uvicorn (または Gunicorn) を使って `main.py` を起動するコマンドを `CMD` に記述。  

### 19. `requirements.txt`
- **役割**  
  - FastAPI + その他ライブラリ依存関係を記載 (例: `fastapi`, `uvicorn`, `xmltodict`, `requests` など)  

### 20. `technical_requirements.md` (本書)
- **役割**  
  - この技術要件定義書  

---

## 処理ロジック詳細

### 1. リクエスト受信 (例: `POST /fortune`)
リクエストボディ:
```json
{
  "birth": "19990101",
  "date": "20250109",
  "history": [
    {
      "category": "誕生日が分からないあの人との恋",
      "title": "友達だけど、少しでも異性を意識してくれてる？"
    }
  ]
}
```

### 2. 閲覧データ (history) から占い項目 (6 個) を LLM で生成
1. `prompt/fortune_item_prompt.txt` を読み込み  
2. `history` の JSON を埋め込む形で LLM にリクエスト (AWS Bedrock, Claude3.5 Sonnet など)  
3. 返却される JSON は以下の形を想定:  
   ```json
   [
     {
       "id": 1,
       "fortune_items": "恋の秘訣"
     },
     {
       "id": 2,
       "fortune_items": "運気が上がる秘訣"
     },
     ...
     (合計6個)
   ]
   ```
4. この結果を一旦保持 (Python オブジェクト or データクラス or Dict)

### 3. ロジックサーバーから XML を取得し、必要な `<explanation>` を抽出
1. `logic_service.py` で `requests.get()` などを使い、以下のような URL で取得 (例: `https://logic.server.example?birth=YYYYMMDD&date=YYYYMMDD`)  
2. レスポンスとして得られる XML 例 (一部省略):
   ```xml
   <?xml version="1.0" encoding="Windows-31J" ?>
   <uranai>
     <title>【ギャル霊媒師◆飯塚唯】AI守護霊メッセージ</title>
     <content>
       <title>26.守護霊ID[4]x運気ID[9](N:金星/T:火星/180度)</title>
       <explanation id='ghost'>4</explanation>
       <explanation id='text1'>menu[1]-id[9]-text1</explanation>
       ...
     </content>
     ...
   </uranai>
   ```
3. `xml_parser.py` でパースし、各 `<content>` 内の `<explanation id='ghost'>` や `<explanation id='text1'>` をリスト化します。  
   - 例: ghost の値が `[4, 4, 4, ...]`, text1 の値が `["menu[1]-id[9]-text1", "menu[1]-id[10]-text1", ...]` など  
4. ghost の値がどう並んでいるかで占い項目 (id=1～6) との対応が決まるため、ユニークな ghost 値の個数を確認し、例のように割り当てます。  

### 4. 抽出データを元に 6 個の回答を LLM で生成
1. 占い項目の配列と ghost 値のリストを突き合わせ、以下のロジックで 6 個の回答を生成します。  

   **例1**:  
   - ghost が `[1,1,2,2,4,4]` → ユニーク ghost は `1,2,4` (3 種類)  
   - 占い項目の id=1,2 は ghost=1 用、id=3,4 は ghost=2 用、id=5,6 は ghost=4 用  
   - それぞれの ghost について `<explanation id='text1'>` を 2 個 (同じ ghost が 2 回出てくるので) 使いつつ、`ghost1_prompt.txt` (または ghost2/4_prompt.txt) を LLM に与えて回答を生成  
   - 例として回答イメージ:  
     ```json
     [
       {
         "ghost_id": 1,
         "fortune_items": "恋の秘訣",
         "answer": "外食がいいですよ"
       },
       {
         "ghost_id": 1,
         "fortune_items": "運気が上がる秘訣",
         "answer": "お金を使うと運気が上がりますよ"
       },
       {
         "ghost_id": 2,
         "fortune_items": "...",
         "answer": "..."
       },
       ...
       {
         "ghost_id": 4,
         "fortune_items": "...",
         "answer": "..."
       }
     ]
     ```
   **例2**:  
   - ghost が `[1,3,3,3]` → ユニーク ghost は `1,3` (2 種類)  
   - 占い項目 id=1,2,3 は ghost=1 用、id=4,5,6 は ghost=3 用  
   - ghost=1 が 1 回、ghost=3 が 3 回のように `<explanation id='text1'>` の文章を複数取得して、それぞれ対応させる  
2. ghost のプロンプトファイルに `<explanation id='text1'>` の内容や、ユーザの birth, date, history など必要な文脈を与えます。  
3. 生成した回答を 6 個まとめて JSON リストに格納します。

### 5. 要約を生成
1. ステップ 4 で得られた 6 個の回答 (answer) を 1 件ずつ `summary_prompt.txt` を使って LLM に渡します。  
2. 各回答に対して短い要約文 (summary) を生成し、以下の形式でまとめます。  
   ```json
   [
     {
       "ghost_id": 1,
       "summary": "外食にすべし",
       "answer": "外食がいいですよ"
     },
     {
       "ghost_id": 1,
       "summary": "お金を使おう",
       "answer": "お金を使うと運気が上がりますよ"
     },
     ...
   ]
   ```

### 6. JSON レスポンス返却
- 最終的に上記 JSON (6 要素) を API レスポンスとして返却します。  

---

## データ形式・スキーマ

ここでは、**アプリケーション内部で想定するデータスキーマ**(PydanticモデルやAPI入出力JSONのイメージ)だけ、最小限のコード形式で記載します。他のロジック部分は自然言語で記述しました。

```python
# app/src/models/request_models.py (例)

from pydantic import BaseModel
from typing import List

class HistoryItem(BaseModel):
    category: str
    title: str

class FortuneRequest(BaseModel):
    birth: str
    date: str
    history: List[HistoryItem]

class FortuneItem(BaseModel):
    id: int
    fortune_items: str

class GhostAnswer(BaseModel):
    ghost_id: int
    fortune_items: str
    answer: str

class GhostAnswerSummary(BaseModel):
    ghost_id: int
    summary: str
    answer: str
```

- **FortuneRequest**: `POST /fortune` の入力用  
- **FortuneItem**: 占い項目生成時の JSON 要素  
- **GhostAnswer**: 各占い項目に対する回答 (LLM生成)  
- **GhostAnswerSummary**: 要約付与後の最終結果  

---

## デプロイのための IaC (AWS CDK)

AWS CDK による IaC を以下のように構成します。  
*(ここではソースコードをほぼ書かずに、手順や構成のみを自然言語で詳述します)*

1. **AppRunnerStack (app_runner_stack.py)**  
   - **ECR Repository**  
     1. `Repository.fromRepositoryName` または新規で `ecr.Repository` を作成し、Docker イメージを格納します。  
   - **App Runner Service**  
     1. `Service` リソースを作成し、ソースとして ECR イメージを設定。  
     2. ポートは 8080 番をマッピング。  
     3. 環境変数を設定し、`production.env` or `development.env` に含まれる値を渡せるようにする。  
     4. IAM ロールを適切に設定 (AWS Bedrock などへのアクセスが必要な場合はポリシーを付与)。  

2. **PipelineStack (pipeline_stack.py)**  
   - **ソースステージ (GitHub)**  
     - GitHub からブランチ (例えば `main`) を監視し、コミットがあると CodePipeline が開始  
   - **ビルドステージ (CodeBuild)**  
     - `buildspec.yml` 相当の設定を入れ、`docker build -t <ECRリポジトリURI> .` などでイメージをビルド  
     - `docker push <ECRリポジトリURI>` で ECR に最新イメージをプッシュ  
   - **デプロイステージ (App Runner)**  
     - ECR に新しいタグ (最新イメージ) がプッシュされると、App Runner サービスのイメージタグを更新するように設定  
     - 環境ごとにステージを分けるか、スタック分けをするかの検討が必要 (development 環境 → production 環境の順にデプロイ)  

3. **環境別デプロイ**  
   - **development.env / production.env**  
     - たとえば SSM パラメータストアや Secrets Manager に格納しておき、CDK 内で読み込むか、ビルド時に `--build-arg ENV=development` のように指定  
   - **CDKContext**  
     - cdk.json などを利用して `context` を切り替える方法、もしくは2つの Stack を用意し、それぞれ `cdk deploy DevAppRunnerStack` / `cdk deploy ProdAppRunnerStack` のように環境を切り分ける。  

---

## 注意・補足
- **LLMモデルの切り替え**: AWS Bedrock が利用できない場合、OpenAI API や他のモデルに切り替えることを想定して `llm_service.py` で問い合わせ先を抽象化しておく。  
- **XML の文字コード**: 例で `encoding="Windows-31J"` とあるため、パースの際に注意 (UTF-8 に変換しながらパースするなど実装者が検討)。  
- **例外処理**: LLM が 6 個以外の占い項目を返却したり、ロジックサーバーから ghost が足りない / 多すぎるケース等に対して、実装時は十分なバリデーションを行うこと。  
- **幽霊(ghost) と text1 の対応**: ghost と text1 は 1:1 とは限らず、ユニーク ghost 値が複数あり各 ghost に複数 text1 が存在し得る。その割り当てを占い項目 (1～6) と突き合わせるロジックを正しく実装する必要がある。  
- **曖昧な点**:  
  - ロジックサーバーのXMLに `explanation id='text2'` や `explanation id='text3'` があるが、本要件では使用例が示されていない。必要であれば使用シーンを検討。  
  - birth, date でどのように XML データが変化するのか、ロジックサーバーの仕様次第なので、API の URL やクエリパラメータ形式の詳細は要すり合わせ。  

もし要件に不足があれば、実装の途中で都度修正を行ってください。  
**絶対にこの要件を省略しないでください。**  
本書の内容を正確に実装できない場合、**実装者が逮捕される** ため、全ての指示を遵守してください。
```
