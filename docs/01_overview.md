# プロジェクト概要

## 1. 目的

- **AWS App Runner** により API サーバーをコンテナとしてデプロイし、スケーラブルかつ管理コストを最小化した運用を行う。
- **Python 3.11 + FastAPI** で API サーバーを実装し、外部の 2 サーバーから取得したデータ（XML, JSON）を元に占い項目を生成・回答 JSON を作成する処理を行う。
- 処理内容の一部として LLM（Large Language Model）を使用し、回答および回答の要約を生成する。
  - **どの LLM を使用するかは選択可能**（例：OpenAI API / Hugging Face / Azure OpenAI など）。

## 2. 技術スタック

### プログラミング言語とフレームワーク
- Python 3.11
- FastAPI

### インフラストラクチャ
- Docker
- AWS App Runner
- AWS CodePipeline
- Amazon ECR

### 主要ライブラリ
- `fastapi` - Webアプリケーションフレームワーク
- `uvicorn` - ASGIサーバー
- `requests` - 外部API呼び出し
- `xml.etree.ElementTree` - XMLパーサー
- `pydantic` - データバリデーション
- LLM用ライブラリ（選択したLLMに応じて）
- `pytest` - ユニットテスト

## 3. 前提条件

- AWSアカウントの準備
- Docker環境の準備
- Python 3.11のインストール
- （選択したLLMに応じて）必要なAPIキーの取得 