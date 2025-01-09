```md
# 技術要件定義書

本ドキュメントは、**AWSのApp RunnerでAPIサーバーを構築**するための技術要件定義書です。  
これだけを見て実装されるため、**すべての情報を正確かつ詳細に記述**し、省略はありません。  
記載されている情報をもとに、もし実装が困難な箇所があれば**必ず適切な位置で指摘**して相談してください。  
（本書の指示どおりに実装できない場合、実装者が逮捕されることがあるため、慎重に進めてください）

---

## 1. 目的

- AWS App Runner を用いて、FastAPI ベースの占い用 API サーバーを構築・デプロイする。  
- 閲覧データサーバーから取得した閲覧データ（JSON）と、ロジックサーバーから取得した占い結果（XML）を用いて **占い結果** を返す API を提供する。  
- 占い結果には、LLMを用いたテキスト生成と要約が含まれる。  
- デプロイに際しては、AWS CodePipeline / AWS App Runner / AWS CDK (Python) を利用し、**IaC (Infrastructure as Code)** でパイプラインおよびリソースを構築する。

---

## 2. 全体像

### 2.1 アプリケーションの処理フロー
1. **閲覧データサーバー**から閲覧データ (JSON) を取得する。
2. **ロジックサーバー**からロジック結果データ (XML) を取得する。
3. **閲覧データ**から占い項目6個 (ID=1,2,3,4,5,6) を生成するためのプロンプトを作成し、選択したLLMを用いて占い項目情報を作成する。  
   - 出力形式例：
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
     ]
     ```
4. **ロジック結果データ**内にある `<explanation id='ghost'>` および `<explanation id='text1'>` の情報を参照して、占い項目ごとにLLMへプロンプトを投げ、回答を生成する。  
   - 「ghost」の値が同一の要素をまとめ、そこに含まれる回数分だけ `<explanation id='text1'>` の内容が必要になる。  
   - 例1のパターン（ghost=1,1,2,2,4,4 → ユニークな数字:1,2,4）や 例2 のパターン（ghost=1,3,3,3 → ユニークな数字:1,3）を参照。
   - 出力形式例：
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
       ...
     ]
     ```
5. 4で得られた回答 (`answer`) に対して **再度 LLM** を利用し、要約 (`summary`) を生成して **API レスポンス**として返却する。  
   - 出力形式例：
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

### 2.2 システム構成図（概念レベル）
```
[ロジックサーバー (XML)]
         |
         | (XML形式)
         v
[APIサーバー(FastAPI) - Python3.11 - Docker - AppRunner]
         ^
         | (JSON形式)
[閲覧データサーバー (JSON)]
```
- 上記 API サーバーを AWS App Runner 上にデプロイし、外部からHTTPSでアクセス可能にする。  
- APIサーバーは FastAPI を介してロジックサーバー・閲覧データサーバーと通信し、それぞれのデータを基にLLMを呼び出し、テキスト生成・要約を行う。  
- デプロイとパイプラインの管理は AWS CDK (Python) と CodePipeline, CodeBuild, App Runner を使用して行う。

---

## 3. ディレクトリ構成

本プロジェクトは以下のようなディレクトリ構成を想定する。  
（名前は必要に応じて変更可。ただし、ファイルの役割を保つことを推奨。）

```
fortune_api/
├── app/
│   ├── main.py                 <- FastAPIのエントリポイント
│   ├── logic/
│   │   ├── data_retrieval.md   <- ロジックサーバー/閲覧データサーバーからの取得方法(仕様書)
│   │   ├── fortune_logic.md    <- 占いロジックとLLMの利用方法(仕様書)
│   │   └── data_parsing.md     <- XML/JSONのパース仕様(仕様書)
│   ├── models/
│   │   └── schemas.md          <- API入出力スキーマの仕様書(要件のみ記述)
│   ├── config/
│   │   └── settings.md         <- 環境変数・設定値に関する仕様書
│   ├── Dockerfile.md           <- Dockerビルド手順(仕様書)
│   └── requirements.md         <- Python依存パッケージの記載(仕様書)
├── infrastructure/
│   ├── cdk_app.py.md           <- AWS CDKエントリポイント(仕様書)
│   ├── pipeline_stack.md       <- CodePipeline/CodeBuild/その他IaCの設定(仕様書)
│   └── apprunner_stack.md      <- AppRunner, IAMロールなどの設定(仕様書)
├── README.md                   <- プロジェクトの概要とセットアップ手順、注意事項(仕様書)
└── .gitignore                 <- Git管理対象外ファイルの説明(仕様書)
```

> **重要**: 上記の「.md」ファイルは実装ガイド・仕様書の位置づけであり、「データ形式以外」は具体的なコードではなく**自然言語で詳細に説明**を記述している。実際のプロジェクトでは `.py` や `Dockerfile` として実際のコードを配置するが、ここでは**あくまでも仕様書としての役割**のみ記載する。  
> 実装者は本書の指示に従って `.py` や `Dockerfile` 等を作成すること。

---

## 4. ファイルごとの役割と依存関係

### 4.1 `app/main.py`
- **役割**  
  - FastAPI のエントリポイントとなるファイル。  
  - 以下のような内容を**実装**するためのガイドラインを示す。 
    1. **APIのエンドポイント**（例：`/fortune` など）を定義。  
    2. 閲覧データサーバー・ロジックサーバーへのリクエストメソッド（HTTPクライアント等の使い方）を記述。  
    3. LLM呼び出しロジックを呼ぶためのサービス層の関数を呼び出し。  
    4. JSON形式のレスポンスを作成して返却。  
- **依存関係**  
  - `logic` ディレクトリ（占いロジック、データ取得など）と `models` ディレクトリ（スキーマ定義）に依存。  
  - FastAPI および Python 3.11 が必須。  
  - LLM を使用するための外部ライブラリ（例：OpenAI Python ライブラリ、または Hugging Face トランスフォーマー関連等）への依存が考えられる。

### 4.2 `app/logic/data_retrieval.md`
- **役割**  
  - ロジックサーバー (XML) と 閲覧データサーバー (JSON) からデータを取得する方法を詳細に記述。  
    - HTTP リクエストパラメータ、エンドポイントURL (例: 環境変数で与えられる場合がある)  
    - 認証ヘッダやトークンが必要であればその記述  
    - XML/JSON の取得例
- **依存関係**  
  - PythonのHTTPクライアント (requestsなど) に依存。  
  - 取得したデータをパースする処理を `data_parsing.md` が参照。

### 4.3 `app/logic/data_parsing.md`
- **役割**  
  - XMLおよびJSON形式データをどのようにパース（解析）し、どのようなPythonデータ構造に格納するかを記述。  
  - 具体的な要素名のマッピング (例：`<explanation id="ghost">` → `ghost_id`) などを定義。  
- **依存関係**  
  - `data_retrieval.md` で取得したデータを解析するために呼び出される。  
  - XMLライブラリ (例：`xml.etree.ElementTree` や `lxml`) に依存。  
  - JSON は標準ライブラリ `json` を想定。

### 4.4 `app/logic/fortune_logic.md`
- **役割**  
  - 閲覧データ(6項目)の生成フロー、ロジック結果データ(ghost, text1)をもとにLLMへプロンプトを投げて占い結果を生成するフロー、要約生成フローの詳細を記述。  
  - **重要な処理**: 
    1. 閲覧データサーバーから取得した JSON を使い、LLM へのプロンプトを作成 → 占い項目6個を生成  
    2. ロジックサーバーから取得した XML 内の `<explanation id="ghost">` や `<explanation id="text1">` を用いて、各項目に紐づく回答をLLMで生成  
       - ghost の値の出現回数（同じ値が複数あれば text1 も複数回）に応じたロジック  
    3. 生成した回答を再度 LLM で要約  
  - **回答例**で示された JSON 形式の出力例を、**それぞれのステップでどう組み立てるか**を詳述する。  
- **依存関係**  
  - LLM呼び出し用のモジュール/ライブラリ (OpenAI, Hugging Face, Azure OpenAI, etc.)  
  - `data_parsing.md` で取り出した構造体を引数に受け取り、APIサーバー(`main.py`) へ返す。

### 4.5 `app/models/schemas.md`
- **役割**  
  - APIでやりとりする **入力データ / 出力データ** のスキーマを明確に定義する。  
  - Pydantic モデルや Dataclass 等を想定した形で、どんなキーを持ち、どんな型を持つかを自然言語で記述。  
  - 例:  
    - **閲覧データ**: `category: str`, `title: str`  
    - **占い項目**: `id: int`, `fortune_items: str`  
    - **占い回答**: `ghost_id: int`, `fortune_items: str`, `answer: str`  
    - **最終レスポンス**: `ghost_id: int`, `summary: str`, `answer: str`

### 4.6 `app/config/settings.md`
- **役割**  
  - 各種環境変数や設定値をどのように管理するかを記述。  
  - 例:  
    - LLM API KEY (例: `OPENAI_API_KEY`)  
    - ロジックサーバーのエンドポイントURL (例: `LOGIC_SERVER_URL`)  
    - 閲覧データサーバーのエンドポイントURL (例: `VIEW_SERVER_URL`)  
    - デバッグフラグ (例: `DEBUG`)  
  - これらを `.env` ファイルで管理するのか、AWS Systems Manager Parameter Store や Secrets Manager で管理するのかなどの方針もここに書く。

### 4.7 `app/Dockerfile.md`
- **役割**  
  - Docker イメージをビルドする手順を自然言語で記述する（実際のDockerfileではない）。  
  - Python 3.11 ベースイメージを使用し、FastAPIやLLMライブラリの必要な依存関係をインストールし、`main.py` を起動する流れ。  
  - 例:  
    1. python:3.11-slim等のイメージをFROMで指定  
    2. pipでrequirementsをインストール  
    3. FastAPI を起動するコマンド(例: `uvicorn app.main:app --host 0.0.0.0 --port 8080`) でコンテナをスタート  
  - App Runner がデフォルトでポート8080を使う想定であるため、FastAPIのポート指定を8080に統一する方針などをここで示す。

### 4.8 `app/requirements.md`
- **役割**  
  - Python の主要ライブラリのバージョンと依存関係を自然言語で記述。  
  - 例:  
    - `fastapi==0.88.0`  
    - `uvicorn==0.20.0`  
    - `requests==2.28.1`  
    - `openai==0.XX.X` (or `transformers==4.X.X` etc. 選択したいLLMに応じて)  
    - `lxml==4.9.1` (または `xml.etree.ElementTree` のみ使うなら不要)  
    - `pydantic==1.10.2` (FastAPI依存)  
    - `python-dotenv==0.21.0` (必要に応じて)

---

## 5. インフラ構築 (Infrastructure ディレクトリ)

### 5.1 `infrastructure/cdk_app.py.md`
- **役割**  
  - AWS CDK のエントリポイントとなるファイル構成を自然言語で定義。  
  - スタック（`pipeline_stack.md` や `apprunner_stack.md`）をここで読み込み、アプリケーションをデプロイする流れを説明。  
- **依存関係**  
  - `pipeline_stack.md` (CodePipeline, CodeBuild)、`apprunner_stack.md` (App Runner)  
  - AWS CDK (Python)  
  - AWSアカウントID、リージョン設定などの環境変数が必要。

### 5.2 `infrastructure/pipeline_stack.md`
- **役割**  
  - AWS CodePipeline と CodeBuild の構成を自然言語で詳述。  
  - **パイプライン**:  
    1. ソースステージ（GitHub 等からソースを取得）  
    2. ビルドステージ（CodeBuild で Docker イメージをビルドし ECR にプッシュ）  
    3. デプロイステージ（App Runner を更新）  
  - **CodeBuild**:  
    - Docker ビルド コマンド実行。  
    - `requirements.md` に基づいてイメージ内で `pip install` する流れ。  
  - **ECR**:  
    - 生成されたDockerイメージをECRに格納。  
- **依存関係**  
  - AWS CodePipeline / CodeBuild / ECR / IAM など。  
  - `apprunner_stack.md` と連携し、App Runner がECR イメージを取得できるようにする。

### 5.3 `infrastructure/apprunner_stack.md`
- **役割**  
  - App Runner サービスの詳細設定を自然言語で定義。  
  - ECR の Docker イメージをもとに稼働する設定を行い、環境変数やポート設定を記述。  
  - **注意点**: 
    - App Runner はビルド済みのイメージを直接ECRから取得する方法と、ソースコードからビルドさせる方法がある。  
    - 今回は**CodePipeline -> ECR -> App Runner** の構成とし、App Runner ではイメージを取り込むだけにする。  
- **依存関係**  
  - ECR にあるイメージと IAM ロール (ECR からPullする権限)。  
  - CodePipeline / CodeBuild との連動。

---

## 6. AWSリソースの具体的な設定例

※以下は**概念的なIaC構成**の説明であり、実際のCDKコードではなく自然言語での詳細設定例である。

1. **App Runner サービス**  
   - **サービス名**: `fortune-api-service` (例)  
   - **ソース**: ECR の特定タグ (例: `latest` or `build-*`)  
   - **ポート**: 8080  
   - **環境変数**:  
     - `LOGIC_SERVER_URL`  
     - `VIEW_SERVER_URL`  
     - `OPENAI_API_KEY` (など)  
   - **オートスケーリング**: 同時接続数に応じて、最小インスタンス数を1、最大インスタンス数を3程度に設定可能(要件に応じて)。  

2. **ECR リポジトリ**  
   - **リポジトリ名**: `fortune-api-repo` など  
   - CodeBuild から Docker イメージを push する。  
   - App Runner から pull するロールを設定。  

3. **CodePipeline**  
   - **ステージ1 (Source)**: GitHub または CodeCommit などでソースコードを取得。  
   - **ステージ2 (Build)**: CodeBuild プロジェクトを実行して Docker イメージをビルドし ECR に push。  
     - ビルド完了後にイメージのタグをパイプラインに渡す。  
   - **ステージ3 (Deploy)**: App Runner のコンフィグを更新し、新しいイメージタグを適用。  

4. **IAMロール**  
   - CodeBuild が ECR へプッシュするための権限。  
   - App Runner が ECR からイメージを取得するための権限。

> **注意**: App Runnerを使用する場合、ECR Pull 後のデプロイを自動的にトリガーさせる構成が選択可能な場合があるため、**CodePipeline** でのDeployステージをどう扱うかは要件・予算・運用ポリシーに応じて決定すること。

---

## 7. LLM選択の考慮

- **OpenAI GPT** (例: `gpt-3.5-turbo` や `gpt-4`)  
- **Hugging Face Transformers** (例: `distilgpt2` など自己ホストまたはInference APIを利用)  
- **Azure OpenAI**  
- **その他** (Google PaLM APIなど)  

本プロジェクトでは、**複数のLLMを選択的に使えるようにする**ため、LLM呼び出し部分は抽象化したインタフェースを用意するなど、要件定義で注意。  
- LLMが異なる場合、APIキーやエンドポイントが異なる点を設定ファイル (`app/config/settings.md`) で管理する。  
- ロジックは**「入力(プロンプト)」「出力(回答)」**が得られればよい設計とし、特定のLLM固有のパラメータ（例: `max_tokens`, `temperature` 等）は**設定ファイルか環境変数**で変更可能にする。

---

## 8. ロジックの詳細

この章では、**実装者が誤解なく実装できるよう**に、**各ステップをさらに詳細に**記述する。  

### 8.1 閲覧データ（JSON）の取得と占い項目作成

1. **閲覧データの取得**  
   - `VIEW_SERVER_URL` (例: `https://example.com/viewdata`) に対して GET リクエストを送信する。  
   - 返却される JSON は以下のような形式を想定。  
     ```json
     [
       {
         "category": "誕生日が分からないあの人との恋",
         "title": "友達だけど、少しでも異性を意識してくれてる？"
       },
       {
         "category": "人間関係について",
         "title": "あの人は『都合のいい女』だって思ってる？"
       },
       {
         "category": "あの人の本音",
         "title": "あの人が「この人は大切にしなきゃ」って想う人"
       }
     ]
     ```  
   - データが複数件返ってくる場合は、**必要に応じて6件(またはそれ以上)を利用**するなどのビジネスロジックを決める。  
2. **占い項目（id=1..6）の生成**  
   - 受け取った閲覧データをLLMへ渡し、**「恋の秘訣」「運気が上がる秘訣」等の文言を生成**させる。  
   - たとえば、`category` や `title` をまとめて、  
     ```
     「以下の閲覧データから6個の占い項目を作ってください。各項目はユニークなIDと占い項目名を含むように出力してください。」
     ```  
     というプロンプトをLLMに投げて生成する。  
   - LLMの応答を解析し、以下の形式に整形する。  
     ```json
     [
       {"id": 1, "fortune_items": "恋の秘訣"},
       {"id": 2, "fortune_items": "運気が上がる秘訣"},
       ...
       {"id": 6, "fortune_items": "何かの秘訣"}
     ]
     ```  
   - **注意**: 具体的な「秘訣」内容はLLMで生成されるため、**実装者がカスタムプロンプト**を書く必要あり。

### 8.2 ロジックデータ（XML）の取得と解析

1. **ロジック結果データの取得**  
   - `LOGIC_SERVER_URL` (例: `https://example.com/logicdata`) に対して GET リクエストを送信する。  
   - 返却されるデータの例：  
     ```xml
     <?xml version="1.0" encoding="Windows-31J" ?>
     <uranai>
       <title>【ギャル霊媒師◆飯塚唯】AI守護霊メッセージ</title>
       <reader>gal</reader>
       <admin>ZAPPALLAS</admin>
       <version>1.0</version>
       <link>http://www.zappallas.com/</link>
       <content>
         <title>0</title>
         <explanation id='num'>3</explanation>
       </content>
       <content>
         <title>1</title>
         <explanation id='menu'>2002</explanation>
         <explanation id='id'>13</explanation>
         <explanation id='text1'>頭をいっぱい使える人なんだね...[長文]...</explanation>
       </content>
       <content>
         <title>26.守護霊ID[4]x運気ID[9](N:金星/T:火星/180度)</title>
         <explanation id='menu'>1</explanation>
         <explanation id='id'>9</explanation>
         <explanation id='text1'>menu[1]-id[9]-text1</explanation>
         <explanation id='text2'>menu[1]-id[9]-text2</explanation>
         <explanation id='text3'>menu[1]-id[9]-text3</explanation>
         <explanation id='ghost'>4</explanation>
         <explanation id='sdate'>20241226</explanation>
         <explanation id='edate'>20250215</explanation>
         <explanation id='angle'>180.0</explanation>
       </content>
       ...
       <result>2000</result>
     </uranai>
     ```  
2. **XMLの構造解析**  
   - `<uranai>` 直下に複数の `<content>` 要素がある。  
   - `<content>` 内の `<explanation id='ghost'>` や `<explanation id='text1'>` を取得。  
   - これらを Python の dictリスト等に格納する。  
     - 例:  
       ```python
       {
         "title": "26.守護霊ID[4]x運気ID[9](N:金星/T:火星/180度)",
         "ghost": 4,
         "text1": "menu[1]-id[9]-text1",
         ...
       }
       ```  
   - ghost値のユニークな集合を取得し、同じ ghost値を持つエントリーの数を把握する（後述する回答生成のために使用）。

### 8.3 占い回答生成

1. **ghostID と text1 の出現回数に応じた回答生成**  
   - 例1: `<explanation id='ghost'>` の値が `[1, 1, 2, 2, 4, 4]` → ユニーク値は 1,2,4 でそれぞれ2回ずつ。  
     - **占い項目 id=1,2** は ghost=1 の text1 (2件ぶん) を使って回答  
     - **占い項目 id=3,4** は ghost=2 の text1 (2件ぶん) を使って回答  
     - **占い項目 id=5,6** は ghost=4 の text1 (2件ぶん) を使って回答  
   - 例2: `<explanation id='ghost'>` の値が `[1,3,3,3]` → ユニーク値は 1,3 で ghost=1 は1回、ghost=3 は3回。  
     - **占い項目 id=1,2,3** は ghost=1 の text1 (1件ぶん) を使って回答  
     - **占い項目 id=4,5,6** は ghost=3 の text1 (3件ぶん) を使って回答  
2. **LLM へのプロンプト設計**  
   - `ghost_id` や `text1`、および前ステップで作成した「占い項目 (fortune_items)」を組み合わせてプロンプトを生成。  
   - たとえば:  
     ```
     "Ghost IDが{ghost_id}で、テキストは{text1}です。以下の占い項目「{fortune_items}」に対して、ユーザーを励ますようなアドバイスを生成してください。"
     ```  
   - これを ghost値の登場回数分だけ繰り返して回答を受け取り、最終的に**6個の回答**になるよう組み立てる。  
3. **回答形式**  
   - 結果をまとめ、以下の形式のリストを返す:  
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
       ...
     ]
     ```
   - ghost_id と fortune_items の組合せをキーとし、LLMからの出力を `answer` に格納する。

### 8.4 要約生成

1. **要約のための追加LLM呼び出し**  
   - 各回答 (answer) を **再度 LLM** へ投げる。  
   - 例:  
     ```
     "以下の文を要約してください: {answer}"
     ```
2. **要約結果の格納**  
   - 返却された要約を `summary` フィールドとして持つ JSON を生成。  
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
3. **最終レスポンス**  
   - APIからのレスポンスとしてこのリスト全体を JSON で返却する。

---

## 9. セキュリティと運用上の注意

- **LLM APIキー**は**環境変数**または**AWS Secrets Manager**で安全に管理する。  
- **ロジックサーバーURL**・**閲覧データサーバーURL**も設定ファイルや環境変数にして、**実行環境ごとに切り替えられる**ようにする。  
- **App Runner** はHTTPSが標準で付与される (awsアカウントID.app.run.aws-region.amazonaws.com のURLなど)。カスタムドメインを使う場合はRoute53やCloudFrontの設定が必要になる場合がある。  
- **スケーリング**: アクセスが少ない場合でも最低稼働数を1インスタンスにしておくと応答が早い。コスト最適化のため0にもできるがコールドスタートが発生する。  
- **LLMコスト**: 生成文字数やトークン数に応じて課金されるLLMを使う場合、**適切なトークン数制御**や**要約呼び出し回数**を管理する必要がある。

---

## 10. 疑問や不明点

- **LLMの具体的なAPI**: OpenAIかHugging FaceかAzureかなど、まだ明確でない場合は**優先度高く検討**し、プロンプト形式・APIキー管理方式が異なるため、早期に仕様決定して対応すること。  
- **認証/認可**: APIサーバー自体を外部公開する際、IP制限や認証を行うかどうか。必要ならばJWTやAPI Key AuthなどをFastAPIで実装する必要がある。  
- **XMLの文字コード**: 例に `encoding="Windows-31J"` とあるが、Pythonでのパース時にUTF-8以外は文字化けのリスクがあるため、**正しくデコード**できるように設定を行う。  
- **GhostIDの出現パターン**: 現在の例では複数回出現したghostIDを占い項目に対して重複して割り当てる仕様だが、**実際にビジネスロジックでどう割り当てるか**は詳細設計段階で詰める必要がある。

---

## 11. 実装およびテスト手順 (概要)

1. **ローカル環境での開発**  
   1. Python3.11 環境構築  
   2. 仮想環境を作成し、`app/requirements.md` を参考にライブラリをインストール  
   3. `main.py` 内のFastAPI サーバーを起動 (uvicorn使用)  
   4. 閲覧データサーバー・ロジックサーバーのURLをテスト用モックに向けて動作検証  
   5. LLM APIキーをテストキーに設定し動作確認  
2. **Dockerイメージの作成とテスト**  
   1. `Dockerfile.md` の内容をもとに Dockerfile を作成  
   2. `docker build` → `docker run -p 8080:8080`  
   3. ブラウザやcurl等で FastAPI エンドポイントにリクエストを送り、レスポンスを確認  
3. **AWSデプロイ (手動)**  
   1. `cdk_app.py.md` を参考にCDKの実装ファイルを用意  
   2. `cdk bootstrap` → `cdk synth` → `cdk deploy`  
   3. CodePipeline, AppRunner, ECR などが作成される  
4. **動作確認**  
   1. CodePipeline がトリガーされ、Dockerイメージがビルドされ ECR にプッシュ  
   2. AppRunner がECR からイメージを取得し起動  
   3. AppRunner のURL (または独自ドメイン) にアクセスし、APIが動作するか検証

---

## 12. 今後の拡張

- **要約精度の向上**: LLMを高度なものにアップグレードする、もしくはテンプレートエンジンを導入してユーザー向けにカスタマイズされた要約を生成。  
- **データベース接続**: 将来的に閲覧データやロジック結果をキャッシュまたは保存する場合、RDSやDynamoDBを組み込む。  
- **ログ / モニタリング**: CloudWatch Logs, X-Rayなどを活用し、LLMの呼び出し回数や遅延を可視化。  
- **CI/CDの拡充**: テストステージを追加 (pytestなど) し、自動テストを実行。  
- **国際化対応**: 日本語以外の言語にも対応する場合、LLMのプロンプトや応答言語設定、文字コードの考慮が必要。

---

## 13. まとめ

本ドキュメントは、**AWS App Runner 上で FastAPI サーバーを動かす**ための**技術要件定義書**であり、**閲覧データサーバーからのJSON**・**ロジックサーバーからのXML** を用いて **LLMを活用した占い項目/回答/要約** を生成する手順を示した。  
- **ディレクトリ構成**・**ファイルの役割**・**依存関係**・**デプロイのためのIaC**、すべてを網羅した。  
- ここで示した仕様は**実装上の最低限の前提条件**を含むが、実際の要件に合わせて細部を追加・修正していく必要がある。  

もし不足や曖昧な点があれば、**必ず適切な位置で指摘**・相談し、逮捕されないように注意して実装を進めていただきたい。  

以上。  
```
