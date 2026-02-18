# Dify チャットフロー API 連携 調査仕様書

**作成日**: 2026-02-18  
**目的**: Difyチャットフローの出力再現性テストを自動化するための技術調査

---

## 1. 背景と目的

チャットフローの開発・デバッグにおいて、同一入力に対する出力の再現性を繰り返し検証する必要がある。Difyプレビュー画面での手動テスト（コピペによる入力）は実施済みだが、回数増加に伴い手作業の限界に達した。これを自動化する手段を調査する。

---

## 2. Dify チャットフロー API 仕様

### 2.1 エンドポイント

| 項目 | 値 |
|---|---|
| メソッド | `POST` |
| パス | `/v1/chat-messages` |
| Content-Type | `application/json` |
| 認証 | `Authorization: Bearer {API_KEY}` |

### 2.2 必要な情報（3点）

| 項目 | 説明 | 取得方法 |
|---|---|---|
| Base URL | DifyインスタンスのAPIエンドポイント | クラウド版: `https://api.dify.ai/v1`、セルフホスト版: `http://<host>/v1` |
| API Key | アプリ固有の認証キー（`app-xxxx` 形式） | Dify管理画面 → 対象アプリ →「概要」or「APIアクセス」→「APIキー」で生成 |
| User ID | 任意のユーザー識別子 | 呼び出し側で自由に設定（例: `"user-001"`） |

**注意**: API Keyはチャットフローごとに固有。別アプリのキーは使用不可。

### 2.3 リクエストボディ

| パラメータ | 型 | 必須 | 説明 |
|---|---|---|---|
| `query` | string | ✅ | ユーザーの入力メッセージ |
| `user` | string | ✅ | ユーザー識別子 |
| `inputs` | object | — | チャットフローStartノードで定義した変数のkey/value |
| `response_mode` | string | — | `"blocking"`（同期）or `"streaming"`（SSE）。デフォルト: streaming |
| `conversation_id` | string | — | 会話継続時に指定 |
| `files` | array | — | 画像等のファイル添付（Vision対応モデル用） |
| `auto_generate_name` | boolean | — | 会話タイトルの自動生成。デフォルト: true |

**制約**: blockingモードのCloudflareタイムアウトは100秒。

### 2.4 レスポンス（blockingモード）

```json
{
  "event": "message",
  "task_id": "uuid",
  "message_id": "uuid",
  "conversation_id": "uuid",
  "mode": "chat",
  "answer": "応答テキスト",
  "metadata": {
    "usage": {
      "prompt_tokens": 123,
      "completion_tokens": 456,
      "total_tokens": 579
    }
  },
  "created_at": 1234567890
}
```

### 2.5 Python 最小実装例

```python
import requests

DIFY_BASE_URL = "http://<host>/v1"
DIFY_API_KEY = "app-xxxxxxxxxxxxxxxx"

def chat(query: str, user: str = "user-001", conversation_id: str = "") -> dict:
    resp = requests.post(
        f"{DIFY_BASE_URL}/chat-messages",
        headers={
            "Authorization": f"Bearer {DIFY_API_KEY}",
            "Content-Type": "application/json",
        },
        json={
            "query": query,
            "user": user,
            "inputs": {},
            "response_mode": "blocking",
            "conversation_id": conversation_id,
        },
    )
    resp.raise_for_status()
    return resp.json()
```

---

## 3. 環境制約：ネットワーク問題

### 3.1 現状

| 項目 | 状況 |
|---|---|
| Difyの運用形態 | 親会社によるセルフホスト（プライベートIP: `10.x.x.x`） |
| 自身のネットワーク | 子会社（親会社とは別ネットワーク） |
| DifyのWeb UI | ブラウザからアクセス可能（前提） |
| Dify API（`/v1/chat-messages`） | **未確認** — プライベートIPへの到達可否がボトルネック |

### 3.2 接続パターンと可能性

| パターン | 説明 | 確認先 |
|---|---|---|
| 拠点間VPN / 専用線 | 親子会社間でネットワーク接続済みなら到達可能 | 自社 情シス |
| クライアントVPN | 親会社ネットワークへのVPNアカウント発行 | 親会社 管理者 |
| リバースプロキシ | Difyが外部公開されている可能性（低） | Dify管理者 |

### 3.3 疎通確認コマンド

```bash
curl -s -o /dev/null -w "%{http_code}" http://10.60.xxx.xxx/v1/parameters \
  -H "Authorization: Bearer app-your-key"
```

- **200 or 401** → ネットワーク到達可（API利用可能）
- **タイムアウト / 接続拒否** → 経路なし（管理者へ依頼が必要）

---

## 4. 対応方針

### 4.1 方針の選択肢

| # | 方式 | 前提条件 | 即時性 | 長期運用 |
|---|---|---|---|---|
| A | **Dify API + Python** | ネットワーク経路の開通 | △（管理者依頼待ち） | ◎ |
| B | **Playwright UI自動化** | Web UIにアクセス可能 | ◎（即日着手可） | △（UI変更に弱い） |

### 4.2 推奨アクション

1. **即時**: Playwright によるプレビュー画面自動化で、テスト作業をまず開始
2. **並行**: 管理者へ API 経路の開通を依頼（下記テンプレート参照）
3. **経路開通後**: Python + API に移行し、本格的な自動テスト環境を構築

### 4.3 管理者への依頼テンプレート

> チャットフローの開発・デバッグで、同一入力に対する出力再現性を繰り返しテストする必要があります。手動では限界があるため、子会社ネットワークからDify APIへのアクセス経路を開通いただけないでしょうか。対象エンドポイントは `POST /v1/chat-messages` のみです。

---

## 5. 補足情報

### 5.1 既存Pythonライブラリ

PyPIに `dify-api-python` パッケージが存在し、streaming/blocking両対応。ただしrequests直接呼び出しで十分簡素なため、依存追加は任意。

### 5.2 API Key に関する注意

- API Keyはチャットフローごとに個別生成
- 管理画面でEditor以上のロールがあれば自身で発行可能
- 機密情報として扱い、ソースコードへの直接埋め込みは非推奨（環境変数等で管理）
- アプリが「公開（Published）」状態でないとAPIは機能しない
