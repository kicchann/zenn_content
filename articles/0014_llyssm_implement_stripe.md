---
title: "FastAPI × Stripe Webhook でサブスクリプションを管理する"
emoji: "🙌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["python", "stripe", "fastapi"]
published: false
---

## はじめに

LLySSMでは、ユーザーの課金状況に応じてプロジェクト数・ストレージ容量の制限を行うため、**Stripeによるサブスクリプション課金**を導入しました。Webhookによるイベント通知を受け取り、**内部API経由で本体システム（llyssm-api）と連携**しています。
これにより、疎結合な設計で、セキュアかつ拡張性のある課金管理を実現しています。

---

## 構成方針と役割分担

| コンポーネント              | 役割                                                |
| -------------------- | ------------------------------------------------- |
| `llyssm-billing-api` | Stripe Webhook受信、subscriptionの状態を整形しllyssm-apiに転送 |
| `llyssm-api`         | Subscription情報をDBに永続化、本体の機能制限判断に使用                |

---

## Webhook連携フロー

1. StripeからWebhookでイベントが通知される
2. `billing-api`が署名を検証し、イベントタイプごとのハンドラを呼び出す
3. Subscription情報を整形し、`llyssm-api/internal/subscription/update` にPOST
4. `llyssm-api`が対応するユーザーのSubscriptionテーブルを更新

---

## リッスン対象のイベント一覧

Stripeの以下のイベントに対応：

* `customer.subscription.created`
* `customer.subscription.updated`
* `customer.subscription.deleted`
* `invoice.payment_succeeded`
* `invoice.payment_failed`

それぞれのイベントごとに `handlers.py` に処理を分離し、責務を明確化。

---

## 🔐 セキュリティ：署名検証 & 内部API認証

* Webhook受信時に `STRIPE_WEBHOOK_SECRET` を用いた署名検証を必須化
* `llyssm-api` の内部APIは `X-API-Key` によるキー認証で保護

---

## 📂 APIエンドポイント

### `billing-api`

```http
POST /webhook/stripe
```

### `llyssm-api`

```http
POST /internal/subscription/update
Authorization: X-API-Key: <secret>
```

受信パラメータ例：

```json
{
  "customer_id": "cus_XXX",
  "subscription_id": "sub_XXX",
  "plan_id": "price_XXX",
  "start_date": 1710000000,
  "expiry_date": 1712688399,
  "status": "active"
}
```

---

## 🛠 運用と開発TIPS

* Webhookの署名検証には `stripe_signature = Header(None, convert_underscores=False)` を忘れずに
* テストイベントはStripe Dashboardの**Webhook UI**から送信可能
* Webhookが常に200を返すようにし、**失敗時もStripeの再送を回避**

---

## 🧪 よくあるハマりポイント

* Webhookで400: Invalid Signature → **Dashboardに表示されたsecretが環境変数に設定されていない**
* subscriptionのカラム追加後、`alembic upgrade` を忘れてDBに反映されていない
* `PUT` を `POST` に変え忘れて `405 Method Not Allowed`
* API Keyの不一致で `401 Unauthorized`

---

## 📝 補足：SubscriptionModel に保持するべきステータス

Stripeの全ステータスのうち、LLySSMでは以下のように解釈：

| ステータス                            | 意味         | 採用 | 備考                      |
| -------------------------------- | ---------- | -- | ----------------------- |
| `active`                         | 有効         | ✅  | 正常に課金されている              |
| `incomplete_expired`             | 支払い完了せず失効  | ✅  | 実質キャンセル扱い               |
| `canceled`                       | 明示キャンセル    | ✅  | ユーザーによる解約               |
| `past_due` / `unpaid` / `paused` | 支払い遅延・一時停止 | △  | 初期MVPでは未対応、通知や警告は将来対応予定 |

---

## 🎉 まとめ

Stripe Webhook + FastAPI による課金管理は、

* **疎結合な設計（Webhook → internal API）**
* **セキュアなAPIキー連携**
* **イベントベースの自動処理**

により、柔軟かつ拡張性のある実装が可能です。今後は、管理画面やStripeポータル連携、エラー通知などを追加していく予定です。