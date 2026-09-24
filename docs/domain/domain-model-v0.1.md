# かしわや ドメインモデル設計 v0.1

## 1. 設計目的

本書は、店舗業務を「テーブル」「来店/伝票」「注文」「注文明細」「会計」「現金」「営業日」「レジセッション」に分離し、後続のDB/API/画面設計の正本となるドメインモデルを定義する。

## 2. 設計上の最重要原則

1. **テーブルと伝票を同一概念にしない。** テーブルは物理的な場所、伝票は会計単位。
2. **注文と注文明細を分離する。** 1回の送信が注文、商品単位が注文明細。
3. **会計は伝票に対する確定イベント。** 会計確定後の売上金額を直接編集しない。
4. **金額は整数の円で保持する。浮動小数点を金額計算に使わない。**
5. **価格は注文時点で注文明細にスナップショット保存する。** 後日のメニュー価格変更は過去伝票へ影響させない。
6. **取消・訂正は削除ではなく履歴として扱う。**
7. **現金移動は売上と分離する。** 売上、釣銭準備金、出金、入金を混同しない。
8. **営業日とカレンダー日を分離する。** 深夜営業等に備え、業務上の営業日を持つ。
9. **日次締め後の確定取引を直接更新しない。** 訂正取引を新たに記録する。
10. **顧客QRセッションはテーブル番号そのものではない。** 顧客からの他卓アクセス・なりすましを防ぐ。

## 3. エンティティ

### 3.1 BusinessDay（営業日）

店舗の業務上の1日。開店から日次締めまでを1単位とする。

主属性:
- id
- business_date
- status: OPEN / CLOSING / CLOSED
- opened_at
- closed_at

### 3.2 RegisterSession（レジセッション）

営業日のレジ運用単位。原則、営業日につき1セッション。将来複数レジへ拡張可能な構造を残す。

主属性:
- id
- business_day_id
- register_code
- status: OPEN / CLOSED
- opening_cash_yen
- expected_cash_yen
- counted_cash_yen
- variance_yen
- opened_at
- closed_at

### 3.3 Table（テーブル）

物理的な席/卓。

主属性:
- id
- code
- name
- capacity
- status: AVAILABLE / OCCUPIED / CHECKOUT_PENDING / OUT_OF_SERVICE
- display_order

**重要:** Tableは伝票を所有しない。現在紐付いているopen ticketを参照する。

### 3.4 TableSession（着席セッション）

1回の来店・着席を表す。テーブル移動や同一卓の次客を区別するために導入する。

主属性:
- id
- business_day_id
- table_id
- status: OPEN / CLOSED
- started_at
- ended_at

1 TableSession に1つ以上の Ticket を紐付けられる設計を基本候補とするが、初期運用では通常1伝票とする。

### 3.5 Ticket（伝票）

**会計単位。** テーブルとは独立したドメインオブジェクト。

主属性:
- id
- ticket_number
- business_day_id
- table_session_id
- status: OPEN / CHECKOUT_PENDING / PAID / VOIDED / CLOSED
- opened_at
- closed_at
- subtotal_yen
- tax_yen
- total_yen

初期運用では1 TableSession = 1 Ticketを標準とする。ただし将来の別会計・分割会計に備え、モデル上は独立させる。

### 3.6 Order（注文）

顧客または店員が1回の送信/登録で作成した注文ヘッダ。

主属性:
- id
- ticket_id
- order_number
- source: CUSTOMER_QR / STAFF
- status: SUBMITTED / ACCEPTED / IN_PROGRESS / PARTIALLY_SERVED / SERVED / CANCELLED
- idempotency_key
- submitted_at
- accepted_at
- completed_at
- created_by_staff_id nullable

### 3.7 OrderItem（注文明細）

実際の商品1種類についての数量・価格・税情報。

主属性:
- id
- order_id
- menu_item_id
- item_name_snapshot
- unit_price_yen
- tax_rate
- quantity
- subtotal_yen
- tax_yen
- status: ORDERED / PREPARING / PARTIALLY_SERVED / SERVED / CANCELLED
- cancelled_quantity
- note

商品名・価格・税率をスナップショット保存する。

### 3.8 MenuItem（商品）

現在販売している商品マスタ。

主属性:
- id
- category_id
- name
- description
- price_yen
- tax_rate
- available
- kitchen_station
- display_order

価格変更はMenuItem更新で行うが、既存OrderItemには影響しない。

### 3.9 Payment（会計）

伝票の金額を確定し、現金を受領した記録。

主属性:
- id
- ticket_id
- payment_method: CASH
- amount_due_yen
- received_yen
- change_yen
- status: COMPLETED / VOIDED
- completed_at
- staff_id
- idempotency_key

`received_yen >= amount_due_yen` を必須条件とする。

### 3.10 CashMovement（現金移動）

レジ内現金の増減を記録する。売上Paymentとは別オブジェクト。

種別:
- OPENING_FLOAT
- CASH_SALE
- CASH_OUT
- CASH_IN
- ADJUSTMENT

主属性:
- id
- register_session_id
- type
- amount_yen
- reason
- reference_type
- reference_id
- staff_id
- created_at

CASH_SALEはPayment確定から自動生成し、通常手動入力させない。

### 3.11 DailyClosing（日次締め）

営業日を確定する記録。

主属性:
- id
- business_day_id
- register_session_id
- total_sales_yen
- total_cash_sales_yen
- total_cash_out_yen
- expected_cash_yen
- counted_cash_yen
- variance_yen
- status: DRAFT / CONFIRMED
- confirmed_by_staff_id
- confirmed_at

### 3.12 AuditLog（監査ログ）

重要操作の追跡記録。

主属性:
- id
- actor_type: STAFF / CUSTOMER / SYSTEM
- actor_id nullable
- action
- entity_type
- entity_id
- before_json nullable
- after_json nullable
- reason nullable
- occurred_at
- request_id

## 4. ドメイン関係

```text
BusinessDay
  |
  +-- RegisterSession
  |
  +-- TableSession -- Table
  |        |
  |        +-- Ticket
  |              |
  |              +-- Order
  |              |     |
  |              |     +-- OrderItem -- MenuItem
  |              |
  |              +-- Payment
  |
  +-- CashMovement
  |
  +-- DailyClosing
```

## 5. テーブル・伝票の扱い

### 通常営業

```text
TABLE-05
  |
  +-- TableSession #123
          |
          +-- Ticket #20260924-005
                 |
                 +-- Order #001
                 +-- Order #002
                 +-- Order #003
                 +-- Payment #001
```

### 席移動

TableSessionまたはTicketのtable_id変更を監査可能なイベントとして扱う。履歴を消さない。

### 将来の別会計

1つのTableSessionに複数Ticketを許容できるモデルにしておく。ただし初期UIでは複雑性を露出させず、必要になった時点で機能追加する。

## 6. 注文・取消モデル

注文確定後のOrderItemを物理削除しない。

数量を減らす場合も、元の注文履歴と取消数量を追跡可能にする。会計対象金額は有効数量から算出する。

例:

```text
注文: かしわ 5個 x 100円 = 500円
取消: 2個
有効数量: 3個
売上対象: 300円
```

取消理由と実行者をAuditLogへ記録する。

## 7. 会計モデル

会計直前にTicketの有効明細から金額を再計算する。

```text
subtotal = 有効明細の税抜/税込ルールに従った合計
 tax = 税計算ルールに従う
 total = subtotal + tax 等、採用する価格表示方式に従う
```

実際の税込/税抜表示、軽減税率、端数処理は店舗ヒアリングで確定する。

Payment確定時には同一トランザクション内で、以下を行うことを要求する。

1. TicketをCHECKOUT_PENDINGへ遷移
2. 金額を再計算
3. Payment作成
4. CASH_SALEをCashMovementへ作成
5. TicketをPAIDへ遷移
6. AuditLog作成
7. commit

途中失敗時は全体をrollbackする。

## 8. 現金管理モデル

理論現金:

`opening_float + cash_sales + cash_in - cash_out + adjustments`

会計時の釣銭はレジ理論残高から引くのではなく、顧客から受領した金額と請求額の差で計算する。

閉店時に実査金額と理論現金を比較し、varianceを保存する。

## 9. 営業日モデル

カレンダー日と営業日を混同しない。初期実装では日本時間を基準とする。

日次締めCONFIRMED後は、そのBusinessDayに属する確定売上・CashMovement・Paymentの直接編集を禁止する。

## 10. QR注文セッション

QRには単純なtable_idだけでなく、店舗・テーブルを識別する安全な公開識別子を利用する。

客側セッションは、QRからアクセスしたテーブルに限定して操作可能とする。客に管理APIや他テーブルのTicket IDを露出しない。

注文確定にはidempotency_keyを使用し、スマートフォンの二重タップ・通信再送による二重注文を防止する。

## 11. 権限モデル

### CUSTOMER

- 自分の注文画面表示
- 商品閲覧
- 注文送信
- 店員呼び出し

### STAFF

- テーブル閲覧
- 注文登録
- 注文取消/訂正（理由必須）
- 厨房状態更新
- 会計
- レシート再印刷
- 現金出金（必要に応じて権限制限）

### MANAGER

- STAFF機能
- 売切れ
- メニュー価格変更
- レジ開始/終了
- 日次締め
- 売上出力

### ADMIN

- MANAGER機能
- システム設定
- ユーザー管理
- バックアップ/復旧

## 12. 不変条件

1. Ticketのtotalは有効OrderItemから再現可能である。
2. Payment.completedの金額はTicketの確定金額と一致する。
3. Paymentの二重確定を許さない。
4. CashMovement.CASH_SALEはPaymentと1対1で追跡できる。
5. CLOSEDなBusinessDayの取引を直接更新できない。
6. 取消されたOrderItemを売上集計へ二重計上しない。
7. 顧客は他テーブルのTicketを参照できない。
8. 金額は円単位整数で保存する。
9. 重要操作にはAuditLogが存在する。
10. 削除によって会計履歴を失わせない。

## 13. 未確定事項

実装前に店舗ヒアリングで確定する。

- テーブル/席構成
- 席移動の実態
- 別会計・割り勘の有無
- 商品の税込/税抜表示
- 税率・軽減税率の扱い
- 端数処理
- レシート/領収書の要件
- 厨房の注文伝達方法
- ラストオーダー運用
- 深夜営業の有無
- 現金出金の実態
- 店員アカウント運用
- プリンター機種
