## 1. システム概要

- **目的**：キャプチャデバイスや配信サーバーからの映像をリアルタイムに解析し、JSON形式で結果を外部システムへプッシュ  
- **利用者**：解析結果を受け取るAIエージェント（最大2セッション想定）および管理者／運用担当者  

## 2. 映像入出力仕様

| 項目                | 詳細                                                         |
|--------------------|------------------------------------------------------------|
| 入力解像度／fps      | 1080p / 60 fps                                              |
| コーデック           | H.264                                                       |
| ローカルファイル    | MP4コンテナ（H.264映像＋AAC音声）                             |
| ライブストリーミング | RTSP over TCP, MPEG-TS コンテナ上の H.264                      |
| 認証方式（RTSP）     | 初期：Basic 認証<br>後日対応：Digest 認証                      |
| ポート番号           | デフォルト 554                                             |

## 3. 前処理〜解析パイプライン

1. **デコーディング**  
    - FFmpeg／GStreamer などのライブラリで H.264 → 生フレーム  
2. **前処理**  
    - リサイズ、色空間変換など  
3. **抽出処理**  
    - GPU（CUDA）活用による機械学習モデル推論  
4. **レイテンシ要件**  
    - 前処理→抽出：500 ms以内  

## 4. ハードウェア要件

- **GPU**：CUDA対応型1基以上  
- **CPU**：4コア以上 @2.5 GHz以上  
- **メモリ**：16 GB以上  

## 5. 認証モジュール設計

- **Strategy パターン**で抽象化  
    interface AuthStrategy {  
        applyAuthHeaders(Request req): void  
    }  
- **BasicAuthStrategy**（MVP）／**DigestAuthStrategy**（後日追加）  
- **設定切替**：YAML/JSON または CLI オプションで `auth: basic|digest` を指定  

## 6. バックエンドアーキテクチャ

- **言語／フレームワーク**：Python + FastAPI  
- **WebSocket API**  
    - エンドポイント：`wss://<your-domain>/ws/analysis`  
    - サブプロトコル：`json`  
    - 認証：`Authorization: Bearer <JWT>` ヘッダー  
    - メッセージ形式：テキストフレームにプレーンJSON  
        ```json
        {
          "sessionId": "string",
          "timestamp": "2025-06-26T15:04:05Z",
          "event": "frameAnalyzed",
          "payload": { /* 任意の解析結果 */ }
        }
        ```  
    - Ping/Pong：30 秒間隔  
    - 再接続：指数バックオフ（1 → 2 → 5 → …秒、最大60秒）  

## 7. 永続化／データベース設計

```sql
CREATE TABLE analysis_results (
  id          SERIAL PRIMARY KEY,
  session_id  VARCHAR(64) NOT NULL,
  timestamp   TIMESTAMPTZ NOT NULL,
  event       VARCHAR(32) NOT NULL,
  payload     JSONB        NOT NULL
);
CREATE INDEX ON analysis_results (session_id);
CREATE INDEX ON analysis_results (timestamp);
```

- **接続**：SQLAlchemy or asyncpg

## 8. ユーザー管理

```sql
CREATE TABLE users (
  id          SERIAL PRIMARY KEY,
  username    VARCHAR(64) UNIQUE NOT NULL,
  email       VARCHAR(255) UNIQUE NOT NULL,
  password    VARCHAR(255) NOT NULL,  -- bcrypt
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE TABLE roles (
  id   SERIAL PRIMARY KEY,
  name VARCHAR(32) UNIQUE NOT NULL
);
CREATE TABLE user_roles (
  user_id INT REFERENCES users(id),
  role_id INT REFERENCES roles(id),
  PRIMARY KEY(user_id, role_id)
);
```

- **認証／認可**：
  - OAuth2 Password + JWT（FastAPI 標準）
  - bcrypt ハッシュ
  - エンドポイント：`/signup`, `/login`, `/password-reset`, 管理者用ユーザー・ロール管理

## 9. バックアップ・リカバリ

- **フルバックアップ**：毎日 02:00 に `pg_dump`
- **WAL アーカイブ**：リアルタイムに S3 互換ストレージへ送信
- **保持期間**：バックアップ7日分
- **リカバリ手順**：
  1. フルバックアップリストア
  2. WAL リプレイ
  3. 手順書化／演習実施

## 10. デプロイ／CI-CD

- **コンテナ化**：Docker Compose
- **CI/CD**：GitHub Actions
  1. Lint & ユニットテスト
  2. Docker イメージビルド
  3. レジストリへプッシュ
  4. SSH 接続 → `docker-compose pull && docker-compose up -d`

## 11. モニタリング／アラート

- **ログ収集**：
  - アプリケーション → JSON ログ → Fluentd → Elasticsearch → Kibana
- **メトリクス**：
  - `prometheus_client` → Prometheus → Grafana ダッシュボード
- **アラート**：
  - Alertmanager で GPU/CPU or レイテンシ閾値超過 → Slack 通知
  - Elasticsearch Watcher／ElastAlert でエラーログ急増 → メール／Slack

## 12. スケーリング要件

- **同時セッション数**：2
- **処理レート**：100 fps／セッション
- **ノード追加条件**：GPU使用率 > 80 %
