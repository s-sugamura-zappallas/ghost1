# AWS App Runner FastAPI プロジェクト

このプロジェクトは、AWS App Runnerを用いたAPIサーバーの構築を目的としています。
Python 3.11 + FastAPIで実装され、外部の2サーバーから取得したデータ（XML, JSON）を元に
占い項目を生成・回答JSONを作成する処理を行います。

## ドキュメント構成

- [プロジェクト概要](docs/01_overview.md)
- [アーキテクチャ](docs/02_architecture.md)
- [ディレクトリ構成](docs/03_directory_structure.md)
- [デプロイガイド](docs/04_deployment.md)
- [開発ガイド](docs/05_development.md)
- [セキュリティガイド](docs/06_security.md)

## クイックスタート

```bash
# リポジトリのクローン
git clone https://example.com/your-repo.git

# Python仮想環境の作成
python3.11 -m venv venv
source venv/bin/activate

# 依存ライブラリのインストール
pip install -r requirements.txt

# ローカル実行
uvicorn src.main:app --reload --host 0.0.0.0 --port 8000
```

## ライセンス

このプロジェクトは独自のライセンスの下で提供されています。 