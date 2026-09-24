# かしわや DB詳細設計 v0.1

## 1. DB方針

- DBMS: SQLite
- 文字コード: UTF-8
- 金額: INTEGER（日本円、1円単位）
- 時刻: UTC保存または日本時間固定のどちらかを実装規約として統一する。初期運用は日本国内のみのため、API/DBで扱いを明文化する。
- ID: UUID/ULID等の衝突しにくい文字列IDを基本候補とする。
- 表示用連番はDB内部IDと分離する。
- 外部キー制約を有効化する。
- トランザクションを必要な業務単位で使用する。
- 物理削除はマスタの非使用化等を除き原則禁止。

## 2. テーブル一覧

| Table | 目的 |
|---|---|
| business_days | 営業日 |
| register_sessions | レジ営業セッション |
| tables | 物理テーブル |
| table_sessions | 着席単位 |
| menu_categories | 商品カテゴリ |
| menu_items | 商品マスタ |
| staff_users | 店員アカウント |
| tickets | 伝票 |
| orders | 注文ヘッダ |
| order_items | 注文明細 |
| payments | 会計 |
| cash_movements | 現金移動 |
| daily_closings | 日次締め |
| notifications | 呼び出し等 |
| print_jobs | 印刷要求履歴 |
| audit_logs | 監査ログ |
| system_settings | 店舗設定 |

## 3. business_days

```sql
id TEXT PRIMARY KEY
business_date TEXT NOT NULL UNIQUE
status TEXT NOT NULL CHECK(status IN ('OPEN','CLOSING','CLOSED'))
opened_at TEXT NOT NULL
closed_at TEXT
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```

## 4. register_sessions

```sql
id TEXT PRIMARY KEY
business_day_id TEXT NOT NULL REFERENCES business_days(id)
register_code TEXT NOT NULL
status TEXT NOT NULL CHECK(status IN ('OPEN','CLOSED'))
opening_cash_yen INTEGER NOT NULL CHECK(opening_cash_yen >= 0)
expected_cash_yen INTEGER
counted_cash_yen INTEGER
variance_yen INTEGER
opened_by TEXT NOT NULL REFERENCES staff_users(id)
opened_at TEXT NOT NULL
closed_at TEXT
UNIQUE(business_day_id, register_code)
```

## 5. tables

```sql
id TEXT PRIMARY KEY
code TEXT NOT NULL UNIQUE
name TEXT NOT NULL
capacity INTEGER NOT NULL CHECK(capacity > 0)
status TEXT NOT NULL CHECK(status IN ('AVAILABLE','OCCUPIED','CHECKOUT_PENDING','OUT_OF_SERVICE'))
display_order INTEGER NOT NULL DEFAULT 0
active INTEGER NOT NULL DEFAULT 1 CHECK(active IN (0,1))
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```

## 6. table_sessions

```sql
id TEXT PRIMARY KEY
business_day_id TEXT NOT NULL REFERENCES business_days(id)
table_id TEXT NOT NULL REFERENCES tables(id)
status TEXT NOT NULL CHECK(status IN ('OPEN','CLOSED'))
started_at TEXT NOT NULL
ended_at TEXT
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```

INDEX:
- `(business_day_id, table_id, status)`

## 7. menu_categories

```sql
id TEXT PRIMARY KEY
name TEXT NOT NULL
sort_order INTEGER NOT NULL DEFAULT 0
active INTEGER NOT NULL DEFAULT 1 CHECK(active IN (0,1))
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```

## 8. menu_items

```sql
id TEXT PRIMARY KEY
category_id TEXT NOT NULL REFERENCES menu_categories(id)
name TEXT NOT NULL
description TEXT
price_yen INTEGER NOT NULL CHECK(price_yen >= 0)
tax_rate INTEGER NOT NULL CHECK(tax_rate >= 0)
available INTEGER NOT NULL DEFAULT 1 CHECK(available IN (0,1))
kitchen_station TEXT
sort_order INTEGER NOT NULL DEFAULT 0
active INTEGER NOT NULL DEFAULT 1 CHECK(active IN (0,1))
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```

価格変更履歴が必要な場合は別menu_price_historiesを追加する。

## 9. staff_users

```sql
id TEXT PRIMARY KEY
username TEXT NOT NULL UNIQUE
display_name TEXT NOT NULL
role TEXT NOT NULL CHECK(role IN ('STAFF','MANAGER','ADMIN'))
password_hash TEXT NOT NULL
active INTEGER NOT NULL DEFAULT 1 CHECK(active IN (0,1))
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```

## 10. tickets

```sql
id TEXT PRIMARY KEY
ticket_number TEXT NOT NULL UNIQUE
business_day_id TEXT NOT NULL REFERENCES business_days(id)
table_session_id TEXT NOT NULL REFERENCES table_sessions(id)
status TEXT NOT NULL CHECK(status IN ('OPEN','CHECKOUT_PENDING','PAID','VOIDED','CLOSED'))
subtotal_yen INTEGER NOT NULL DEFAULT 0 CHECK(subtotal_yen >= 0)
tax_yen INTEGER NOT NULL DEFAULT 0 CHECK(tax_yen >= 0)
total_yen INTEGER NOT NULL DEFAULT 0 CHECK(total_yen >= 0)
opened_at TEXT NOT NULL
closed_at TEXT
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```

## 11. orders

```sql
id TEXT PRIMARY KEY
ticket_id TEXT NOT NULL REFERENCES tickets(id)
order_number TEXT NOT NULL
source TEXT NOT NULL CHECK(source IN ('CUSTOMER_QR','STAFF'))
status TEXT NOT NULL CHECK(status IN ('SUBMITTED','ACCEPTED','IN_PROGRESS','PARTIALLY_SERVED','SERVED','CANCELLED'))
idempotency_key TEXT NOT NULL UNIQUE
created_by_staff_id TEXT REFERENCES staff_users(id)
submitted_at TEXT NOT NULL
accepted_at TEXT
completed_at TEXT
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
UNIQUE(ticket_id, order_number)
```

## 12. order_items

```sql
id TEXT PRIMARY KEY
order_id TEXT NOT NULL REFERENCES orders(id)
menu_item_id TEXT NOT NULL REFERENCES menu_items(id)
item_name_snapshot TEXT NOT NULL
unit_price_yen INTEGER NOT NULL CHECK(unit_price_yen >= 0)
tax_rate INTEGER NOT NULL CHECK(tax_rate >= 0)
quantity INTEGER NOT NULL CHECK(quantity > 0)
cancelled_quantity INTEGER NOT NULL DEFAULT 0 CHECK(cancelled_quantity >= 0 AND cancelled_quantity <= quantity)
subtotal_yen INTEGER NOT NULL CHECK(subtotal_yen >= 0)
tax_yen INTEGER NOT NULL CHECK(tax_yen >= 0)
status TEXT NOT NULL CHECK(status IN ('ORDERED','PREPARING','PARTIALLY_SERVED','SERVED','CANCELLED'))
note TEXT
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```

## 13. payments

```sql
id TEXT PRIMARY KEY
ticket_id TEXT NOT NULL REFERENCES tickets(id)
payment_method TEXT NOT NULL CHECK(payment_method = 'CASH')
amount_due_yen INTEGER NOT NULL CHECK(amount_due_yen >= 0)
received_yen INTEGER NOT NULL CHECK(received_yen >= amount_due_yen)
change_yen INTEGER NOT NULL CHECK(change_yen >= 0)
status TEXT NOT NULL CHECK(status IN ('COMPLETED','VOIDED'))
idempotency_key TEXT NOT NULL UNIQUE
staff_id TEXT NOT NULL REFERENCES staff_users(id)
completed_at TEXT NOT NULL
created_at TEXT NOT NULL
```

初期仕様ではTicket 1件につき有効なCOMPLETED Paymentは1件を基本とする。分割会計を導入する場合は制約を見直す。

## 14. cash_movements

```sql
id TEXT PRIMARY KEY
register_session_id TEXT NOT NULL REFERENCES register_sessions(id)
type TEXT NOT NULL CHECK(type IN ('OPENING_FLOAT','CASH_SALE','CASH_OUT','CASH_IN','ADJUSTMENT'))
amount_yen INTEGER NOT NULL CHECK(amount_yen >= 0)
reason TEXT
reference_type TEXT
reference_id TEXT
staff_id TEXT NOT NULL REFERENCES staff_users(id)
created_at TEXT NOT NULL
```

金額の符号は列のtypeで意味を表し、負数を入れない。集計時にtypeごとの符号を決定する。

## 15. daily_closings

```sql
id TEXT PRIMARY KEY
business_day_id TEXT NOT NULL UNIQUE REFERENCES business_days(id)
register_session_id TEXT NOT NULL REFERENCES register_sessions(id)
total_sales_yen INTEGER NOT NULL CHECK(total_sales_yen >= 0)
total_cash_sales_yen INTEGER NOT NULL CHECK(total_cash_sales_yen >= 0)
total_cash_out_yen INTEGER NOT NULL CHECK(total_cash_out_yen >= 0)
expected_cash_yen INTEGER NOT NULL CHECK(expected_cash_yen >= 0)
counted_cash_yen INTEGER NOT NULL CHECK(counted_cash_yen >= 0)
variance_yen INTEGER NOT NULL
status TEXT NOT NULL CHECK(status IN ('DRAFT','CONFIRMED'))
confirmed_by_staff_id TEXT REFERENCES staff_users(id)
confirmed_at TEXT
created_at TEXT NOT NULL
updated_at TEXT NOT NULL
```

## 16. notifications

```sql
id TEXT PRIMARY KEY
table_id TEXT NOT NULL REFERENCES tables(id)
type TEXT NOT NULL CHECK(type IN ('CALL_STAFF','SYSTEM'))
status TEXT NOT NULL CHECK(status IN ('OPEN','ACKNOWLEDGED','CLOSED'))
created_at TEXT NOT NULL
acknowledged_at TEXT
closed_at TEXT
acknowledged_by TEXT REFERENCES staff_users(id)
```

## 17. print_jobs

```sql
id TEXT PRIMARY KEY
type TEXT NOT NULL CHECK(type IN ('KITCHEN_ORDER','RECEIPT','INVOICE','DAILY_CLOSING','REPORT'))
reference_type TEXT
reference_id TEXT
status TEXT NOT NULL CHECK(status IN ('QUEUED','PRINTING','COMPLETED','FAILED','CANCELLED'))
printer_name TEXT
attempt_count INTEGER NOT NULL DEFAULT 0
error_message TEXT
created_at TEXT NOT NULL
printed_at TEXT
```

再印刷を可能にするため、印刷要求履歴を保持する。

## 18. audit_logs

```sql
id TEXT PRIMARY KEY
actor_type TEXT NOT NULL CHECK(actor_type IN ('STAFF','CUSTOMER','SYSTEM'))
actor_id TEXT
action TEXT NOT NULL
entity_type TEXT NOT NULL
entity_id TEXT NOT NULL
before_json TEXT
after_json TEXT
reason TEXT
request_id TEXT
occurred_at TEXT NOT NULL
```

INDEX:
- `(entity_type, entity_id, occurred_at)`
- `(actor_id, occurred_at)`
- `(occurred_at)`

## 19. system_settings

```sql
key TEXT PRIMARY KEY
value_json TEXT NOT NULL
updated_at TEXT NOT NULL
updated_by TEXT REFERENCES staff_users(id)
```

## 20. 主要インデックス

- orders(ticket_id, created_at)
- order_items(order_id)
- payments(ticket_id, completed_at)
- cash_movements(register_session_id, created_at)
- tickets(business_day_id, status)
- table_sessions(table_id, status)
- audit_logs(entity_type, entity_id, occurred_at)

## 21. トランザクション必須処理

### 注文確定

1. Ticket状態確認
2. idempotency_key確認
3. Order作成
4. OrderItem作成
5. Ticket再計算
6. AuditLog
7. commit

### 注文取消

1. 権限確認
2. 対象OrderItem確認
3. cancelled_quantity/status更新
4. Ticket再計算
5. AuditLog
6. commit

### 会計確定

1. Ticketロック相当の競合制御
2. 金額再計算
3. received >= due確認
4. Payment作成
5. CashMovement(CASH_SALE)作成
6. Ticket=PAID
7. AuditLog
8. commit

### 日次締め

1. BusinessDay=CLOSING
2. 未会計Ticket=0確認
3. 未処理注文=0確認
4. 理論現金計算
5. 実査金額登録
6. DailyClosing作成
7. BusinessDay=CLOSED
8. RegisterSession=CLOSED
9. AuditLog
10. commit

## 22. 禁止事項

- 会計済みTicketのtotalを直接UPDATEして帳尻を合わせる。
- OrderItemを会計後にDELETEする。
- CashSaleをPaymentなしで作る。
- Paymentを二重登録する。
- 締め済み営業日の取引を直接書き換える。
- 金額をfloat/REALで保持する。
- 顧客から受け取ったtable_idだけを信用して他テーブルのTicketへアクセスさせる。
