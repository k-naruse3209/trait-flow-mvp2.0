<div align="center">
<img width="1200" height="475" alt="GHBanner" src="https://github.com/user-attachments/assets/0aa67016-6eaf-458a-adb2-6e31a0763ed6" />
</div>

# Trait Flow v2.0 プロトタイプ — 実行とデプロイ

このリポジトリは Trait Flow v2.0 の Web UI プロトタイプを含みます。ユーザーの TIPI 回答や日次チェックインを受け付け、AI 介入カードを表示するフローをローカル/Cloud Run で検証できます。システム全体の要件は `docs/system_spec_ja.md` を参照してください。

## ローカルでの実行方法

**前提条件:** Node.js がインストールされていること

1. 依存関係をインストール  
   `npm install`
2. （任意）LLM の API キーを `.env.local` の `GEMINI_API_KEY` に設定  
   ※現在はモック応答を返すため、未設定でも動作します。環境変数は Vite の define でビルド時に注入されます。
3. 開発サーバーを起動  
   `npm run dev`

## Google Cloud Run へのデプロイ

このリポジトリには、本番運用を想定した Dockerfile が含まれており、Vite でビルドした成果物を Nginx で配信する構成になっています。以下の手順で Cloud Run へデプロイできます。

1. **認証とプロジェクトの指定**
   ```bash
   gcloud auth login
   gcloud config set project YOUR_PROJECT_ID
   ```

2. **必要なサービスの有効化（初回のみ）**
   ```bash
   gcloud services enable run.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com
   ```

3. **コンテナイメージのビルドとプッシュ**  
   Cloud Run では `PORT` 環境変数で渡されるポートで待ち受ける必要があります。`start.sh` と `nginx.conf.template` が自動で対応します。
   ```bash
   gcloud builds submit --tag REGION-docker.pkg.dev/YOUR_PROJECT_ID/trait-flow/trait-flow-ui
   ```
   - `REGION` は利用可能な Artifact Registry のリージョン（例: `asia-northeast1`）に置き換えてください。
   - 初回実行時は `trait-flow` というリポジトリが自動作成されます。

4. **Cloud Run へデプロイ**
   ```bash
   gcloud run deploy trait-flow-ui \
     --image REGION-docker.pkg.dev/YOUR_PROJECT_ID/trait-flow/trait-flow-ui \
     --platform managed \
     --region REGION \
     --allow-unauthenticated
   ```
   数十秒待つと Cloud Run からサービス URL が表示されます。アプリは静的サイトなので、実行時に追加の環境変数を設定する必要はありません（必要ならビルド時に `.env.local` を反映させます）。

5. **（任意）Docker でローカル確認**
   ```bash
   docker build -t trait-flow-ui .
   docker run -p 8080:8080 trait-flow-ui
   ```
   http://localhost:8080 にアクセスして、デプロイ前に挙動を確認できます。

## ドキュメント
- システム仕様: `docs/system_spec_ja.md`
- メタデータ: `metadata.json`
