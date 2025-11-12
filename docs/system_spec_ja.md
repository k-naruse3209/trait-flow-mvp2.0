# Trait Flow v2.0 プロトタイプ仕様書

## 0. 背景
- 既存の AI Studio UI（React/Vite）をベースに、ユーザーの回答を保存・分析しながら AI からのフィードバックが動的に変化する **プロトタイプ** を短期間で成立させる。
- 目標は「Big Five × 日次チェックイン × AI 介入」の一連フローを実ユーザーデータで動かし、運用手順と拡張余地を確認すること。定量 KPI は設定せず、レスポンス体感や安定性をオペレーターが確認する。
- 基盤には Supabase（Auth + PostgreSQL + Edge Functions）を用い、フロントエンドは Cloud Run 上でホストする。介入生成に利用する LLM API はエンジニアが選定し、Structured Output/Moderation/フォールバックを実装可能なものを採用する。

## 1. プロトタイプ方針
- 検証仮説や成功指標は本仕様では扱わず、**ユーザー回答→処理→AI 応答が変化すること**を一貫したフローとして成立させることに専念する。
- 実データを Supabase に追記保存し、後から分析・リプレイできる構造を必須とする。
- LLM API の選定・チューニングはエンジニアに委任する（要件: Structured Output/Moderation/監査ログ/フォールバックを実装可能、SLA やコストは運用判断）。
- 将来的なプロダクション展開を見据えているが、プロトタイプ段階ではオペレーター監視・小規模ユーザー（5〜10 名）での安定動作に集中する。

## 2. スコープ
### 今回実装するもの
| レイヤ | 内容 |
| --- | --- |
| フロント | 既存 React UI を日本語のまま利用。Auth/Auth 状態管理、API 呼び出し、リアルデータ表示のための改修を追加。 |
| バックエンド | Supabase Auth, PostgreSQL スキーマ、Edge Functions 3 本（TIPI登録、チェックイン作成、介入生成）。 |
| AI サービス | エンジニアが選定する LLM API（REST/Responses 互換）。Structured Output で `{title, body, tone}` を取得できること。 |
| 運用 | Cloud Run デプロイ、Supabase モニタリング、Slack 通知（失敗時）。 |

### スコープ外
 - Symanto / SNS 連携、Push 通知、ネイティブアプリ、A/B テスト、決済系。

## 3. ユーザーフロー（プロトタイプ）
1. **アクセス & 認証**  
   Cloud Run 上の UI へアクセス → Supabase Auth（メールリンク）でログイン。
2. **TIPI オンボーディング**  
   10 問回答 → `baseline_traits` へ保存 → LLM 呼び出しなしの簡易スコア結果を即時表示。
3. **ホーム**  
   API から当日のメッセージを取得。未読なら「チェックインしてメッセージを受け取る」を促す。
4. **チェックイン**  
   気分/エネルギー/メモを入力 → `/functions/v1/checkins` へ送信 → DB 保存後、キュー `intervention_jobs` にメッセージ生成を投入。
5. **AI メッセージ生成**  
   Edge Function Worker がジョブを処理し 選定した LLM API を呼び出し → `interventions` に保存 → フロントは SSE/ポーリングで結果取得。
6. **履歴 & フィードバック**  
   メッセージカードを開き評価（1〜5）を送信 → `feedback_score` 更新。

## 4. システム構成
```mermaid` 
flowchart LR
  subgraph Client[React / Cloud Run]
    Landing --> Auth --> Onboarding --> App
    App --> Home
    App --> History
    App --> Settings
    Home --> CheckinModal
    History --> MessageDrawer
  end

  subgraph Supabase["Supabase Project"]
    AuthSvc[Auth]
    DB[(PostgreSQL)]
    funcOnboard[Edge Fn: saveTipi]
    funcCheckin[Edge Fn: createCheckin]
    funcPoll[Edge Fn: fetchMessages]
    funcWorker[Edge Fn: processIntervention]
    Queue[(intervention_jobs)]
  end

  subgraph Google["Google Cloud"]
    LLM[LLM API - engineer-selected]
    Slack[Slack Webhook]
  end

  Client <--HTTP--> AuthSvc
  Client <--JWT REST--> funcOnboard & funcCheckin & funcPoll
  funcOnboard --> DB
  funcCheckin --> DB
  funcCheckin --> Queue
  funcWorker --> Queue
  funcWorker --> DB
  funcWorker --> LLM
  funcWorker --> Slack
  funcPoll --> DB
```

## 5. 機能仕様
### 5.1 フロントエンド（主要コンポーネント）
| コンポーネント | 主な API / 挙動 |
| --- | --- |
| Auth Gate | Supabase Auth UI をラップし JWT を取得。React Query で認証状態をキャッシュ。 |
| TIPI Form (`OnboardingPage`) | `POST /functions/v1/tipi` に `{ answers: Record<id,1-7> }` を送信。レスポンス `{ scores: BigFiveScores }` を表示して `/app/home` へ遷移。 |
| Home | `GET /functions/v1/messages/today` を呼び、`status: generating` ならローディング、`completed` なら本文と評価 UI を表示。 |
| CheckinModal | `POST /functions/v1/checkins` を送信し HTTP 202 を受領後、`GET /functions/v1/messages/latest?checkin_id=` を 5 秒間隔でポーリング。 |
| History | `GET /functions/v1/history?limit=20&cursor=...` で一覧取得。詳細で `PATCH /functions/v1/messages/{id}` を呼び `feedback_score` を更新。 |

### 5.2 Edge Functions
| 関数 | 説明 |
| --- | --- |
| `saveTipi` | TIPI 回答を受け取り、Big Five スコアを算出して `baseline_traits` に UPSERT。 |
| `createCheckin` | チェックインを `checkins` に保存、`intervention_jobs` にジョブを enqueue、HTTP 202 を返す。 |
| `fetchMessages` | 今日のメッセージ / 履歴 / 評価更新を扱う REST エンドポイント集合。 |
| `processIntervention` | キューからジョブ取得 → 選定した LLM API を呼び出し → `interventions` へ保存 → Slack へ失敗通知。 |

### 5.3 LLM 呼び出し仕様
- **モデル選定**: エンジニアがレスポンス速度・コスト・日本語出力品質に基づき選択（例: Gemini 1.5 Flash, OpenAI Responses, Claude Haiku など）。  
- **必須要件**:  
  - Structured Output で `{title, body, tone}` を返せる（JSON Schema or function call）。  
  - Moderation/安全制御 API、もしくは同等のコンテンツフィルタを適用できる。  
  - 4 秒以内に応答できるレイテンシ特性（プロトタイプ目安）。  
- **入力**:  
  - 上位/下位各 1 特性（数値）  
  - 直近 3 回の気分平均、最新チェックイン詳細  
  - ユーザーの週次目標（任意フィールド）  
- **アウトプット要件**: `title`（20 文字）、`body`（200 文字以内）、`tone`（`reflective | actionable | compassionate`）。  
- **フォールバック**: タイムアウト/エラー時はテンプレート文を使用し `interventions.use_fallback = true`。LLM 切替・再試行ポリシーは I モジュール（信頼性）で管理する。

## 6. データモデル
| テーブル | 主な列 |
| --- | --- |
| `users` | `id`, `email`, `created_at` |
| `baseline_traits` | `user_id`, `traits_avg` (JSON), `instrument`, `administered_at` |
| `checkins` | `id`, `user_id`, `mood_score`, `energy_level`, `note`, `created_at` |
| `intervention_jobs` | `id`, `user_id`, `checkin_id`, `status`, `attempts`, `payload`, `created_at` |
| `interventions` | `id`, `user_id`, `checkin_id`, `title`, `body`, `tone`, `feedback_score`, `use_fallback`, `created_at` |
| `user_settings` | `user_id`, `notification_channel`, `quiet_hours`, `weekly_goal`, `updated_at` |

RLS（Row Level Security）で `user_id = auth.uid()` の行のみ CRUD 可とする。

## 7. 外部サービス & 依存関係
| 種別 | 内容 |
| --- | --- |
| Supabase | Auth、PostgreSQL、Edge Functions、Storage（将来のメディア用途）。 |
| Google Cloud | Cloud Run（UI）、Secret Manager（API キー）、選定した LLM API への接続、Cloud Scheduler（定期 Worker 起動）、Slack Webhook（運用通知）。 |
| npm 依存 | React 19, React Router 7, React Query, Supabase JS, Tailwind（CDN）, Recharts, Vite 6。 |

## 8. 非機能要件
| 項目 | 要件 |
| --- | --- |
| レイテンシ | チェックイン送信からメッセージ表示まで平均 4 秒以内、p95 7 秒以内。 |
| 可用性 | UI/Edge Functions の稼働率 99%（Cloud Run / Supabase SLA）。 |
| 監視 | Supabase Logs + Cloud Logging。失敗時 Slack 通知。LLM API 使用量を日次エクスポート。 |
| セキュリティ | Supabase Auth + HTTPS。Edge Functions には JWT 必須。LLM / Slack キーは Secret Manager 管理。 |
| バックアップ | DB は Supabase PITR を有効化。`interventions` は 90 日でアーカイブ。 |

## 9. オペレーション
1. **デプロイ**  
   - `npm run build` → Cloud Build → Cloud Run。  
   - Edge Functions は `supabase functions deploy`。  
2. **ランブック**  
   - Edge Function 5xx 増加 → Slack アラート → Cloud Logging で原因特定 → 必要ならキューを手動リプレイ。  
   - LLM API の使用量制限に到達 → Secret Manager でキー切替 or プラン変更。  
3. **サポート**  
   - パイロット参加者用に Slack/LINE グループを用意し、障害時は即時連絡。  

## 10. ロードマップ（6 週間）
1. Week1: Supabase プロジェクト初期化、Auth/Fn/DB スキーマ設計。
2. Week2: TIPI API とホーム画面のデータ取得を実装、CI/CD 整備。
3. Week3: チェックイン API + intervention_jobs → processIntervention Worker。
4. Week4: LLM API 連携、エラーハンドリング、Slack 通知。
5. Week5: QA（iOS/Android）・ロギング・観測性強化・小規模ドッグフーディング。
6. Week6: パイロットローンチ、週次で利用者フィードバックをレビュー。

## 11. リスクと対策
| リスク | 対策 |
| --- | --- |
| LLM API 遅延/失敗 | タイムアウト＆テンプレート fallback、Queue 再試行（最大 3 回）、必要に応じて代替モデルへフェイルオーバー。 |
| Supabase 障害 | Cloud Run 停止＆Slack 通知。緊急時はローカルテンプレートでオフラインモードを提供。 |
| 個人情報保護 | 取得 PII をメールアドレスのみに限定。利用規約・プライポリを公開。 |
| モバイル UX 破綻 | Storybook / Percy でビジュアルリグレッションテスト、主要端末で QA。 |

## 12. 参考資料
- 旧仕様: `trait-flow-mvp/docs/prototype_spec_ja.md`
- `trait-flow-mvp2.0/App.tsx`（UI 実装）
- Supabase Edge Functions & Auth ドキュメント
- LLM API ドキュメント（例: Google Gemini API）
- Cloud Run / Cloud Build / Secret Manager ベストプラクティス
