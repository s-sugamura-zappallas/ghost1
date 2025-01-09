# 開発ガイド

## 1. 開発環境のセットアップ

### 必要なツール
- Python 3.11
- Docker
- AWS CLI
- Git

### 環境構築手順

1. **リポジトリのクローン**
   ```bash
   git clone https://example.com/your-repo.git
   cd your-repo
   ```

2. **Python仮想環境の作成**
   ```bash
   python3.11 -m venv venv
   source venv/bin/activate  # Windows: venv\Scripts\activate
   ```

3. **依存ライブラリのインストール**
   ```bash
   pip install -r requirements.txt
   ```

4. **環境変数の設定**
   ```bash
   cp .env.example .env
   # .envファイルを編集して必要な値を設定
   ```

## 2. ローカル開発

### アプリケーションの起動
```bash
uvicorn src.main:app --reload --host 0.0.0.0 --port 8000
```

### APIエンドポイントの確認
- Swagger UI: http://localhost:8000/docs
- ReDoc: http://localhost:8000/redoc

### 動作確認
```bash
# ヘルスチェック
curl http://localhost:8000/healthcheck

# 占い生成
curl -X POST http://localhost:8000/generateFortune
```

## 3. テスト

### ユニットテストの実行
```bash
# 全テストの実行
pytest

# カバレッジレポートの生成
pytest --cov=src --cov-report=html
```

### テストの作成ガイドライン

1. **テストファイルの配置**
   - `tests/`ディレクトリ配下に配置
   - テスト対象のモジュールと同じ名前で`test_`プレフィックスを付ける

2. **テストケースの作成**
   ```python
   # tests/test_logic_parser.py
   def test_parse_logic_xml():
       sample_xml = """
       <?xml version="1.0" encoding="Windows-31J" ?>
       <uranai>
           <content>
               <explanation id='ghost'>4</explanation>
           </content>
       </uranai>
       """
       result = parse_logic_xml(sample_xml)
       assert result[0]['ghost'] == '4'
   ```

## 4. コーディング規約

### Python コーディングスタイル
- PEP 8に準拠
- 行の最大長: 88文字（black formatter準拠）
- インデント: 4スペース

### コメント規約
- 関数やクラスには必ずdocstringを記述
- 複雑なロジックには適切なインラインコメントを付与

### 型ヒント
- すべての関数とメソッドに型ヒントを付与
```python
def parse_logic_xml(xml_str: str) -> dict:
    """XMLデータをパースしてディクショナリを返す

    Args:
        xml_str (str): パース対象のXML文字列

    Returns:
        dict: パース結果
    """
    pass
```

## 5. デバッグ

### ログ出力
```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

logger.info("Processing request...")
logger.error("Error occurred: %s", error_message)
```

### デバッグツール
- VS Code デバッガー
- Python debugger (pdb)
- FastAPI デバッグモード

### トラブルシューティング
1. ログの確認
2. デバッガーの使用
3. Swagger UIでのAPI動作確認
4. 環境変数の確認 