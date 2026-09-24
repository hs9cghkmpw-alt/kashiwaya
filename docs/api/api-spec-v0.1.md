# かしわや API仕様 v0.1

## 1. 方針

APIは店舗LAN内で利用する。顧客公開APIとスタッフ/管理APIを論理的に分離する。

- JSON
- UTF-8
- 金額は整数円
- 全書き込みAPIはサーバー側で権限・状態・整合性を検証
- 注文/会計など再送され得る操作にはIdempotency-Keyを要求
- エラー形式を統一する

## 2. 認証

### 顧客
QRから発行されるcustomer_session_tokenを使用。テーブルの内部IDやTicket IDを直接信用しない。

### スタッフ
ユーザー名/パスワードによるログインを基本とする。管理画面ではセッションCookie等のサーバー管理セッションを使用する。

## 3. エラー形式

```json
{
  "error": {
    "code": "ORDER_ALREADY_SUBMITTED",
    "message": "注文はすでに受け付けられています。",
    "request_id": "..."
  }
}
```

## 4. 顧客API

### GET /api/customer/session

QRセッションの状態を取得。

### GET /api/customer/menu

販売可能なメニューを取得。

### POST /api/customer/orders

注文を確定。

Headers:
- Idempotency-Key: 必須

Body:
```json
{
  "items": [
    {"menu_item_id":"...", "quantity":2, "note":"..."}
  ]
}
```

サーバーはセッションのTicketへ紐付け、価格をDBから取得してOrderItemへスナップショット保存する。クライアント送信価格は信用しない。

### POST /api/customer/calls

店員呼び出し。

### GET /api/customer/order-status

自席Ticketの注文状態を取得。

## 5. スタッフAPI

### POST /api/auth/login
ログイン。

### POST /api/auth/logout
ログアウト。

### GET /api/staff/tables
テーブル一覧・状態。

### GET /api/staff/tables/{table_id}/ticket
現在の伝票。

### POST /api/staff/orders
店員注文作成。

### POST /api/staff/orders/{order_id}/cancel
注文取消。理由必須。

### PATCH /api/staff/order-items/{item_id}
調理/提供状態変更。

### POST /api/staff/tickets/{ticket_id}/checkout/preview
会計前再計算。ここでは会計を確定しない。

### POST /api/staff/tickets/{ticket_id}/payments
現金会計確定。Idempotency-Key必須。

Body:
```json
{"received_yen":5000}
```

サーバーは最新の有効明細からamount_dueを再計算し、received_yenからchange_yenを計算する。

### POST /api/staff/tickets/{ticket_id}/receipt/reprint
レシート再印刷。

### POST /api/staff/cash-out
現金出金。金額・理由必須。

### GET /api/staff/kitchen/orders
厨房未処理注文。

### POST /api/staff/kitchen/orders/{order_id}/status
厨房状態更新。

### GET /api/staff/notifications
呼び出し一覧。

### POST /api/staff/notifications/{id}/ack
呼び出し対応。

## 6. 管理API

### POST /api/admin/business-days/open
営業日開始。

### POST /api/admin/register-sessions/open
レジ開始・釣銭準備金登録。

### POST /api/admin/business-days/close
日次締め。

締め前検査:
- 未会計Ticket=0
- 未処理注文=0
- レジ実査入力済み
- 理論現金算出済み

### GET /api/admin/reports/daily
日次売上。

### GET /api/admin/reports/monthly
月次集計。

### GET /api/admin/reports/items
商品別集計。

### GET /api/admin/reports/cash
現金集計。

### GET /api/admin/audit-logs
監査ログ。

### CRUD /api/admin/menu/*
メニュー管理。

### CRUD /api/admin/tables/*
テーブル管理。

### CRUD /api/admin/staff/*
スタッフ管理。

## 7. API状態検証

サーバー側で以下を拒否する。

- CLOSED Ticketへの新規注文
- PAID Ticketへの新規注文
- SOLD_OUT/available=false商品の顧客注文
- 存在しない/無効なmenu_item_id
- quantity <= 0
- received_yen < amount_due
- 無権限者による取消/締め/出金
- 日次締め済み営業日への直接変更
- 同一Idempotency-Keyの別内容再利用

## 8. 会計の原子性

Payment作成、CashMovement作成、Ticket=PAID、AuditLogを同一DBトランザクションで確定する。

失敗した場合、全てrollbackする。

## 9. 冪等性

Idempotency-Keyはリクエスト内容と紐付ける。同一Key・同一内容は前回結果を返す。同一Key・異なる内容は409 Conflict。

## 10. 将来拡張

payment_methodは内部モデルで拡張可能だが、v0.1の公開APIはCASHのみ。
