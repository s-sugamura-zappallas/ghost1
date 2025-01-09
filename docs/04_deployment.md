# デプロイガイド

## 1. デプロイフロー

1. ソースコード (GitHub等) → CodePipelineがトリガー
2. CodePipeline内でビルドフェーズ (AWS CodeBuild) で `docker build` → ECRへpush
3. App RunnerがECRのイメージを参照して自動デプロイ

## 2. Dockerfile

```dockerfile
# ベースイメージ: python3.11
FROM python:3.11-slim

# 作業ディレクトリ
WORKDIR /app

# 依存ライブラリのコピー & インストール
COPY requirements.txt /app
RUN pip install --no-cache-dir -r requirements.txt

# ソースコードのコピー
COPY src /app/src

# ポート解放
EXPOSE 8000

# 起動コマンド (uvicorn)
CMD ["uvicorn", "src.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

## 3. AWS リソース構成

### 必須リソース
- AWS CodePipeline
- Amazon ECR
- AWS App Runner
- IAMロール（ECRアクセス用）

### CloudFormation テンプレート例

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: CodePipeline + App Runner Example

Resources:
  # ECR Repository
  MyEcrRepo:
    Type: AWS::ECR::Repository
    Properties:
      RepositoryName: my-fastapi-app-runner-repo

  # App Runner Service
  MyAppRunnerService:
    Type: AWS::AppRunner::Service
    Properties:
      ServiceName: my-fastapi-app-runner-service
      SourceConfiguration:
        ImageRepository:
          ImageIdentifier: !Sub "${MyEcrRepo}.dkr.ecr.${AWS::Region}.amazonaws.com/my-fastapi-app-runner-repo:latest"
          ImageRepositoryType: ECR
        AutoDeploymentsEnabled: true
      InstanceConfiguration:
        Cpu: '1024'
        Memory: '2048'
        EnvironmentVariables:
          - Name: LOGIC_SERVER_URL
            Value: "http://logicserver.example.com/logic"
          - Name: BROWSING_SERVER_URL
            Value: "http://browsing.example.com/data"

  # ECR Access Role
  MyEcrAccessRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
              Service: "build.apprunner.amazonaws.com"
            Action: "sts:AssumeRole"
      ManagedPolicyArns:
        - "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
```

## 4. 環境変数設定

### 必須環境変数
- `LOGIC_SERVER_URL`: ロジック結果データ(XML)取得用URL
- `BROWSING_SERVER_URL`: 閲覧データ(JSON)取得用URL
- `OPENAI_API_KEY`: OpenAI API使用時のAPIキー

### 環境変数の管理
- 開発環境: `.env`ファイル
- 本番環境: AWS Secrets Manager

## 5. デプロイ手順

1. **ECRリポジトリの作成**
   ```bash
   aws ecr create-repository --repository-name my-fastapi-app-runner-repo
   ```

2. **Dockerイメージのビルドとプッシュ**
   ```bash
   # ECRログイン
   aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account_id>.dkr.ecr.<region>.amazonaws.com

   # イメージビルド
   docker build -t my-fastapi-app .

   # タグ付け
   docker tag my-fastapi-app:latest <account_id>.dkr.ecr.<region>.amazonaws.com/my-fastapi-app:latest

   # プッシュ
   docker push <account_id>.dkr.ecr.<region>.amazonaws.com/my-fastapi-app:latest
   ```

3. **CloudFormationテンプレートのデプロイ**
   ```bash
   aws cloudformation deploy \
     --template-file iac/codepipeline_app_runner.yaml \
     --stack-name my-fastapi-app-stack \
     --capabilities CAPABILITY_IAM
   ``` 