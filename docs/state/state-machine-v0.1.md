# かしわや 状態遷移設計 v0.1

## 1. Table

```
AVAILABLE -> OCCUPIED
OCCUPIED -> CHECKOUT_PENDING
CHECKOUT_PENDING -> AVAILABLE
AVAILABLE -> OUT_OF_SERVICE
OUT_OF_SERVICE -> AVAILABLE
```

不正な逆遷移は禁止。

## 2. TableSession

```
OPEN -> CLOSED
```

CLOSED後の注文追加は禁止。

## 3. Ticket

```
OPEN -> CHECKOUT_PENDING -> PAID -> CLOSED
  |           |
  +---------->VOIDED
```

- OPEN: 注文受付可能
- CHECKOUT_PENDING: 会計処理中
- PAID: 支払確定
- CLOSED: 業務上クローズ
- VOIDED: 無効化。理由・操作者必須

PAID/CLOSEDからOPENへ戻さない。

## 4. Order

```
SUBMITTED -> ACCEPTED -> IN_PROGRESS -> PARTIALLY_SERVED -> SERVED
     |            |             |
     +------------+-------------+--> CANCELLED
```

取消は履歴を残す。提供済み商品の取消は通常の注文取消とは別権限・別処理を検討する。

## 5. OrderItem

```
ORDERED -> PREPARING -> PARTIALLY_SERVED -> SERVED
   |           |               |
   +-----------+---------------+--> CANCELLED
```

数量の一部取消・一部提供を表現できるようにする。

## 6. Payment

```
未作成 -> COMPLETED
             |
             +-> VOIDED（訂正等の正式処理時のみ）
```

同一Ticketに複数の有効Paymentを初期版では許可しない。

## 7. BusinessDay

```
OPEN -> CLOSING -> CLOSED
```

CLOSEDからOPENへ戻さない。締め誤りは訂正記録として扱う。

## 8. RegisterSession

```
OPEN -> CLOSED
```

## 9. 呼び出し

```
OPEN -> ACKNOWLEDGED -> CLOSED
```

## 10. 不正遷移

APIはDB値を書き換えるだけでなく、現在状態から許可された遷移かをドメイン層で検証する。

例:
- PAID Ticketへの注文追加 → reject
- CLOSED BusinessDayでの会計 → reject
- SERVED OrderItemをPREPARINGへ戻す → 原則reject
- CLOSED通知への再ACK → reject

## 11. 状態遷移と監査

状態変更はAuditLog対象。少なくとも、actor、entity、旧状態、新状態、時刻、request_idを記録する。
