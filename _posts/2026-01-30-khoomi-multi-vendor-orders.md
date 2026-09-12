---
layout: post
title: "Building Khoomi: Multi-Vendor Order Architecture"
description: "How orders work when a single checkout spans multiple sellers: parent-child structure, atomic inventory reservation, independent fulfillment, and escrow via wallet."
image: https://sot.dev/assets/images/order-refund-complete.png
date: 2026-01-30
category: khoomi
---

Earlier in this series I covered [shop architecture](/khoomi-shop-architecture.html). This one is about how orders work when a customer buys from multiple shops in one checkout.

On Khoomi, a single cart can contain items from different sellers. At checkout, that becomes one payment but multiple fulfillment flows. Shop A might ship tomorrow while Shop B takes a week. The order system needs to handle this gracefully.

---

## The Parent-Child Model

An order has embedded shop orders. One document, multiple fulfillment units:

```go
type Order struct {
    ID            primitive.ObjectID `bson:"_id"`
    OrderNumber   string             `bson:"order_number"`
    CustomerID    primitive.ObjectID `bson:"customer_id"`
    CustomerEmail string             `bson:"customer_email"`

    ShopOrders []ShopOrder `bson:"shop_orders"` // Embedded

    Pricing         OrderPricing       `bson:"pricing"`
    ShippingAddress UserAddressExcerpt `bson:"shipping_address"`

    Status        OrderStatus `bson:"status"`        // Overall
    PaymentStatus string      `bson:"payment_status"`

    CreatedAt   time.Time  `bson:"created_at"`
    PaidAt      *time.Time `bson:"paid_at,omitempty"`
    CancelledAt *time.Time `bson:"cancelled_at,omitempty"`
}

type ShopOrder struct {
    OrderID         primitive.ObjectID `bson:"order_id"`
    ShopID          primitive.ObjectID `bson:"shop_id"`
    ShopName        string             `bson:"shop_name"`
    Items           []OrderItem        `bson:"items"`

    Subtotal     int64 `bson:"subtotal"`      // in kobo
    ShippingCost int64 `bson:"shipping_cost"` // in kobo
    ShopTotal    int64 `bson:"shop_total"`    // in kobo

    ShopOrderStatus OrderStatus `bson:"shop_order_status"` // Independent
    TrackingNumber  string      `bson:"tracking_number,omitempty"`

    SellerPayout SellerPayout      `bson:"seller_payout"`
    Refund       *ShopOrderRefund  `bson:"refund,omitempty"` // Created on cancellation
}
```

The parent `Order` tracks payment. Each `ShopOrder` tracks its own fulfillment. When Shop A marks its portion as shipped, Shop B's status doesn't budge — it stays exactly where it was.

Why embed instead of separate collections? Atomic reads. Fetching an order for display is one query, not a join. The customer sees everything at once: their items, each shop's status, the overall payment state.

The trade-off? Updating a single shop's status requires updating the entire order document. MongoDB's positional operator (`$[elem]`) handles this efficiently, but it's still a larger write than updating a separate document.

---

## Atomic Checkout

Creating an order touches multiple collections. Inventory must be reserved. The cart must be cleared. The order must be inserted. All or nothing:

```go
func (s *orderService) CreateOrderFromCart(ctx context.Context, userID ObjectID, req CreateOrderRequest) (*Order, error) {
    // Build order from cart items (grouping by shop)
    order := buildOrderFromCart(...)

    callback := func(sessCtx mongo.SessionContext) (any, error) {
        // Reserve inventory for each item
        for _, shop := range order.ShopOrders {
            for _, item := range shop.Items {
                filter := bson.M{
                    "_id":                item.ListingID,
                    "inventory.quantity": bson.M{"$gte": item.Quantity},
                }
                update := bson.M{
                    "$inc": bson.M{"inventory.quantity": -item.Quantity},
                }

                result, err := listingColl.UpdateOne(sessCtx, filter, update)
                if err != nil {
                    return nil, err
                }
                if result.ModifiedCount == 0 {
                    return nil, fmt.Errorf("insufficient inventory for '%s'", item.Title)
                }
            }
        }

        _, err := orderColl.InsertOne(sessCtx, order)
        if err != nil {
            return nil, err
        }

        _, err = s.cart.ClearCartItems(sessCtx, userID)
        if err != nil {
            return nil, err
        }

        return order, nil
    }

    result, err := database.ExecuteTransaction(ctx, s.db.MongoClient, callback)
    if err != nil {
        return nil, err
    }

    return result.(*Order), nil
}
```

The filter `"inventory.quantity": bson.M{"$gte": item.Quantity}` is critical. It ensures stock exists *before* decrementing. If two customers checkout simultaneously and only one item remains, exactly one succeeds. The other gets a clear error.

The snippet hides the rest of the unit of work. The same transaction records coupon usage (so a code can't be drained twice) and enqueues stock-change events for analytics and low-stock alerts. After it commits, the order triggers an outbox dispatch and a cache purge. Same shape — reserve, insert, clear — but the transaction is the whole checkout, not just three writes.

Why reserve at checkout instead of cart add? I covered this decision in the [listing article](/khoomi-listing-architecture.html#stock-reservation-later-not-sooner). The short version: carts get abandoned constantly, and reserving at checkout means no expiry timers and no holds to babysit.

---

## Independent Fulfillment

Each shop order has its own lifecycle:

```
PENDING → PAID → PROCESSING → SHIPPED → DELIVERED
                                ↓
                            CANCELLED
```

When a shop updates their status, only the order belonging to their shop gets updated:

```go
func (s *orderService) UpdateShopOrderStatus(ctx context.Context, params UpdateShopOrderParams) error {
    currentStatus, err := extractCurrentShopOrderStatus(snapshot, params.ShopID)
    if err != nil {
        return err
    }

    // Include current status in the filter: two workers can't both win
    filter := bson.M{
        "_id": params.OrderID,
        "shop_orders": bson.M{
            "$elemMatch": bson.M{
                "shop_id":           params.ShopID,
                "shop_order_status": currentStatus,
            },
        },
    }

    callback := func(sessCtx context.Context) (any, error) {
        result, err := os.db.Coll.Orders.UpdateOne(sessCtx, filter,
            bson.M{"$set": buildShopOrderStatusSetFields(params.Status, now, params)})
        if err != nil {
            return nil, err
        }
        if result.ModifiedCount == 0 {
            return nil, errors.New("shop order status has already changed, please retry")
        }

        updatedOrder, err := os.GetOrderByID(sessCtx, params.OrderID)
        if err != nil {
            return nil, err
        }

        if params.Status == OrderStatusDelivered {
            // Release this shop's earnings to its wallet
            err := s.wallet.ReleaseEarningsInTx(sessCtx, params.ShopID, params.OrderID)
            if err != nil {
                return nil, err
            }
        }

        // Derived parent status: when every shop order reaches the same
        // status, the parent follows automatically.
        if allShopOrdersInStatus(updatedOrder, params.Status) {
            _, err = os.db.Coll.Orders.UpdateOne(sessCtx,
                bson.M{"_id": params.OrderID},
                bson.M{"$set": buildMainOrderStatusSetFields(params.Status, now)})
            if err != nil {
                return nil, err
            }
        }

        return nil, nil
    }

    _, err := database.ExecuteTransaction(ctx, s.db.MongoClient, callback)
    return err
}
```

The transitions themselves are validated before the write — `paid→processing/shipped/cancelled`, `processing→shipped/cancelled`, `shipped→delivered`. And because the filter carries the current status, two concurrent requests can't double-advance a shop order; one of them always loses. When Shop B marks a delivered order, only Shop B's earnings are released and its status moves.

The interesting bit is the last step: the parent status is *derived*. Whenever every shop order reaches the same status, the empty parent follows automatically. All shops delivered → the order is delivered. Shop A shipped while Shop B is still processing → the parent keeps its previous label, because there's no honest single answer for "processing with one shipped component."

Why not separate order documents per shop? The customer paid once. They expect one order number, one receipt, one tracking page. If I split orders into separate documents per shop, I'd need to reassemble them for every customer-facing view. Extra queries, extra complexity.

---

## Escrow via Wallet

Sellers don't get paid immediately. The money sits in escrow until delivery:

```go
// Package wallet manages seller earnings on Khoomi.
//
// The wallet has two balances:
//   - Pending: from orders not yet delivered (cannot withdraw)
//   - Available: from delivered orders (can withdraw)
//
// Money Flow:
//  1. CreditPending: Payment received -> +pending, +total_earnings
//  2. ReleaseEarnings: Order delivered -> -pending, +available
//  3. ProcessWithdrawal: Seller withdraws -> -available, +total_withdrawn
```

When payment succeeds, the seller's pending balance increases:

```go
func (s *walletService) CreditPendingBalance(ctx context.Context, shopID, orderID ObjectID, amount int64) error {
    callback := func(sessCtx mongo.SessionContext) (any, error) {
        // Check idempotency - already credited for this order?
        count, _ := s.db.Coll.WalletTransactions.CountDocuments(sessCtx, bson.M{
            "shop_id":  shopID,
            "order_id": orderID,
            "type":     TxTypeOrderEarning,
        })
        if count > 0 {
             // Already credited
            return nil, nil
        }

        tx := WalletTransaction{
            ShopID:  shopID,
            Type:    TxTypeOrderEarning,
            Amount:  amount,
            OrderID: &orderID,
            // ...
        }
        _, err := s.db.Coll.WalletTransactions.InsertOne(sessCtx, tx)
        if err != nil {
            return nil, err
        }

        update := bson.M{
            "$inc": bson.M{
                "pending_balance": amount,
                "total_earnings":  amount,
            },
        }
        _, err = s.db.Coll.Wallets.UpdateOne(sessCtx, bson.M{"shop_id": shopID}, update)
        return nil, err
    }

    _, err := database.ExecuteTransaction(ctx, s.db.MongoClient, callback)
    return err
}
```

When the order is delivered, pending becomes available:

```go
func (s *walletService) ReleaseEarnings(ctx context.Context, shopID, orderID ObjectID) error {
    callback := func(sessCtx mongo.SessionContext) (any, error) {
        // Find the pending transaction
        var pendingTx WalletTransaction
        err := s.db.Coll.WalletTransactions.FindOne(sessCtx, bson.M{
            "shop_id":  shopID,
            "order_id": orderID,
            "type":     TxTypeOrderEarning,
        }).Decode(&pendingTx)
        if err != nil {
            return nil, ErrPendingEarningsNotFound
        }

        amount := pendingTx.Amount

        releaseTx := WalletTransaction{
            ShopID:      shopID,
            Type:        TxTypeEarningReleased,
            Amount:      amount,
            OrderID:     &orderID,
            Description: "Earnings released - order delivered",
        }
        _, err = s.db.Coll.WalletTransactions.InsertOne(sessCtx, releaseTx)
        if err != nil {
            return nil, err
        }

        // Move from pending to available
        walletUpdate := bson.M{
            "$inc": bson.M{
                "pending_balance":   -amount,
                "available_balance": amount,
            },
        }
        _, err = s.db.Coll.Wallets.UpdateOne(sessCtx, bson.M{"shop_id": shopID}, walletUpdate)
        return nil, err
    }

    _, err := database.ExecuteTransaction(ctx, s.db.MongoClient, callback)
    return err
}
```

Why escrow? Buyer protection. If a seller never ships or order status is "paid", the money can be refunded. If the item arrives damaged, there's a dispute window before the seller can withdraw.

The trade-off? Sellers wait for their money. Cash flow suffers. But for a marketplace with unknown sellers, trust requires holding funds until delivery is confirmed.

### The settlement split

The wallet view hides the rest of the ledger. At payment time, the settlement engine recomputes a three-pot split for each shop order and verifies it against what was stamped at checkout:

- **Seller earnings** — credited to the wallet's pending balance. This is the only slice that ever enters the wallet.
- **Shipping escrow** — locks the full carrier cost in a separate account, released to the carrier on fulfillment (or clawed back on refund). If a promo funded the shipping, a `ShippingFundedBy` flag records who covers the gap.
- **Platform fees** — credited to the platform account in the same transaction.

So "release earnings on delivery" means: the seller's slice moves pending → available, and the shipping escrow pays the carrier. Two locks, two releases, one delivery event.

---

## Partial Cancellation

One shop can cancel their order without affecting others:

```go
func (s *orderService) CancelOrderByShop(ctx context.Context, orderID, shopID ObjectID, reason string) error {
    order, _ := s.GetOrderByID(ctx, orderID)

    // Find this shop's order
    var shopOrder *ShopOrder
    for i := range order.ShopOrders {
        if order.ShopOrders[i].ShopID == shopID {
            shopOrder = &order.ShopOrders[i]
            break
        }
    }

    needsRefund := order.Status == OrderStatusPaid || order.Status == OrderStatusProcessing

    callback := func(sessCtx mongo.SessionContext) (any, error) {
        updateFields := bson.M{
            "shop_orders.$[elem].shop_order_status": OrderStatusCancelled,
            "shop_orders.$[elem].updated_at":        time.Now(),
        }

        // Create refund record if order was paid
        if needsRefund {
            refund := ShopOrderRefund{
                Status:      RefundStatusPending,
                Amount:      shopOrder.ShopTotal,
                InitiatedBy: RefundByShop,
                Reason:      reason,
                RequestedAt: time.Now(),
                RetryCount:  0,
            }
            updateFields["shop_orders.$[elem].refund"] = refund
        }

        update := bson.M{"$set": updateFields}
        arrayFilters := options.Update().SetArrayFilters(options.ArrayFilters{
            Filters: []any{bson.M{"elem.shop_id": shopID}},
        })

        _, err := s.db.Coll.Orders.UpdateOne(sessCtx, bson.M{"_id": orderID}, update, arrayFilters)
        if err != nil {
            return nil, err
        }

        // Restore inventory
        for _, item := range shopOrder.Items {
            inventoryUpdate := bson.M{
                "$inc": bson.M{"inventory.quantity": item.Quantity},
            }
            _, _ = s.db.Coll.Listings.UpdateOne(sessCtx, bson.M{"_id": item.ListingID}, inventoryUpdate)
        }

        // Check if ALL shops are now cancelled
        updatedOrder, _ := s.GetOrderByID(sessCtx, orderID)
        allCancelled := true
        for _, so := range updatedOrder.ShopOrders {
            if so.ShopOrderStatus != OrderStatusCancelled {
                allCancelled = false
                break
            }
        }

        // If all cancelled, cancel the whole order
        if allCancelled {
            mainUpdate := bson.M{
                "$set": bson.M{
                    "status":         OrderStatusCancelled,
                    "payment_status": "refund_pending",
                    "cancelled_at":   time.Now(),
                },
            }
            _, err = s.db.Coll.Orders.UpdateOne(sessCtx, bson.M{"_id": orderID}, mainUpdate)
        }

        return nil, err
    }

    _, err := database.ExecuteTransaction(ctx, s.db.MongoClient, callback)
    return err
}
```

The `$[elem]` array filter lets me target a specific shop order by its `shop_id`. If Shop A cancels but Shop B is still fulfilling, the parent order stays active. Only Shop A's portion shows as cancelled.

Why allow partial cancellation? Stock issues happen. A seller might realize they can't fulfill an item after accepting the order. Cancelling the entire order (including items from other shops that are ready to ship) would punish buyers and other sellers for one shop's mistake.

The real method goes through a cancel plan, not raw field edits. The plan answers a few questions up front: can this shop cancel at all, may it cancel from `shipped` (only before the carrier confirms pickup), does inventory need restoring, which shops get refund records, and what should the parent become if every shop cancels. After the transaction, applied coupons for the cancelled branch get rolled back, and the refund records the plan creates carry the reverse-split amounts — seller earnings clawback, shipping-escrow debit, and platform-fee clawback are tracked separately, not rolled into one number.

When a paid order is cancelled, it also creates a refund record:

```go
type ShopOrderRefund struct {
    Status        RefundStatus    `bson:"status"`         // pending/processing/completed/failed
    Amount        int64           `bson:"amount"`         // in kobo
    InitiatedBy   RefundInitiator `bson:"initiated_by"`   // customer/shop/system
    Reason        string          `bson:"reason,omitempty"`
    RequestedAt   time.Time       `bson:"requested_at"`
    ProcessedAt   *time.Time      `bson:"processed_at,omitempty"`
    CompletedAt   *time.Time      `bson:"completed_at,omitempty"`
    Reference     string          `bson:"reference,omitempty"`
    FailureReason string          `bson:"failure_reason,omitempty"`
    RetryCount    int             `bson:"retry_count"`
    LastRetryAt   *time.Time      `bson:"last_retry_at,omitempty"`

    // Reverse settlement split: each pot clawed back separately.
    Scope               RefundScopeName `bson:"refund_scope,omitempty"`
    WalletDeductAmount  int64           `bson:"wallet_deduct_amount,omitempty"`   // seller earnings
    EscrowDebitAmount   int64           `bson:"escrow_debit_amount,omitempty"`    // shipping escrow
    PlatformDebitAmount int64           `bson:"platform_debit_amount,omitempty"`  // fee clawback
}
```

---

## Automated Refund Processing

Refunds run on a scheduler, not inline with cancellation:

```go
func (j *RefundsJob) Run(ctx context.Context) {
    orders, _ := j.orderService.GetPendingRefunds(ctx)

    for _, order := range orders {
        for _, shopOrder := range order.ShopOrders {
            if shopOrder.Refund == nil || shopOrder.Refund.Status != RefundStatusPending {
                continue
            }

            // Max 5 retries before marking as failed
            if shopOrder.Refund.RetryCount >= MaxRefundRetries {
                j.orderService.MarkRefundAsFailed(ctx, order.ID, shopOrder.ShopID,
                    "Maximum retry attempts exceeded")
                continue
            }

            j.orderService.ProcessShopOrderRefund(ctx, order.ID, shopOrder.ShopID)
        }
    }
}
```

Why async? Payment provider APIs fail. Network timeouts happen. If I processed refunds inline with cancellation, a Paystack outage would block the entire cancellation flow. With a scheduler, the cancellation succeeds immediately. The customer sees "cancelled" right away, and the actual money movement retries in the background until it works.

Processing a refund deducts from the seller's pending balance:

```go
func (s *orderService) ProcessShopOrderRefund(ctx context.Context, orderID, shopID ObjectID) error {
    // Mark as processing (with retry count increment)
    updateProcessing := bson.M{
        "$set": bson.M{
            "shop_orders.$.refund.status":       RefundStatusProcessing,
            "shop_orders.$.refund.processed_at": time.Now(),
        },
        "$inc": bson.M{
            "shop_orders.$.refund.retry_count": 1,
        },
    }
    s.db.Coll.Orders.UpdateOne(ctx, filter, updateProcessing)

    // Deduct from seller wallet
    err := s.wallet.DeductForRefund(ctx, shopID, orderID, shopOrder.Refund.Amount)
    if err != nil {
        // Revert to pending for retry
        revertUpdate := bson.M{
            "$set": bson.M{
                "shop_orders.$.refund.status":         RefundStatusPending,
                "shop_orders.$.refund.failure_reason": fmt.Sprintf("Wallet deduction failed: %v", err),
            },
        }
        s.db.Coll.Orders.UpdateOne(ctx, ...)
        return err
    }

    completeUpdate := bson.M{
        "$set": bson.M{
            "shop_orders.$.refund.status":       RefundStatusCompleted,
            "shop_orders.$.refund.completed_at": time.Now(),
        },
    }
    s.db.Coll.Orders.UpdateOne(ctx, ...)
    return nil
}
```

The wallet deduction reverses the original credit:

```go
func (s *walletService) DeductForRefund(ctx context.Context, shopID, orderID ObjectID, amount int64) error {
    callback := func(sessCtx mongo.SessionContext) (any, error) {
        // Idempotency check
        count, _ := s.db.Coll.WalletTransactions.CountDocuments(sessCtx, bson.M{
            "shop_id":  shopID,
            "order_id": orderID,
            "type":     TxTypeRefundDeduction,
        })
        if count > 0 {
            return nil, ErrRefundAlreadyProcessed
        }

        // Verify sufficient pending balance
        var w SellerWallet
        s.db.Coll.Wallets.FindOne(sessCtx, bson.M{"shop_id": shopID}).Decode(&w)
        if w.PendingBalance < amount {
            return nil, ErrInsufficientPendingBalance
        }

        tx := WalletTransaction{
            ShopID:      shopID,
            Type:        TxTypeRefundDeduction,
            Amount:      amount,
            OrderID:     &orderID,
            Description: "Refund deduction for cancelled order",
        }
        s.db.Coll.WalletTransactions.InsertOne(sessCtx, tx)

        // Deduct from pending and total earnings
        walletUpdate := bson.M{
            "$inc": bson.M{
                "pending_balance": -amount,
                "total_earnings":  -amount,
            },
        }
        s.db.Coll.Wallets.UpdateOne(sessCtx, bson.M{"shop_id": shopID}, walletUpdate)
        return nil, nil
    }

    _, err := database.ExecuteTransaction(ctx, s.db.MongoClient, callback)
    return err
}
```

The refund deducts from `pending_balance` and `total_earnings` — the seller never actually earned this money, it's going back to the customer. But the deduction is only the *seller earnings* slice. The same refund also debits the shipping escrow and claws back the platform fees, each pot tracked separately in the refund record. The buyer-facing refund amount is the sum; the bookkeeping keeps the parts.

![Order refund completed](/assets/images/order-refund-complete.png)

The trade-off? If the seller somehow withdrew before delivery (which shouldn't happen, since pending balance can't be withdrawn), the deduction fails. After 5 retries, the refund is marked as failed, and I get an alert to handle it manually.

---

## Cleanup Expired Orders

Pending orders that never get paid should release their inventory:

```go
// GetPendingOrdersForCleanup finds them; CancelExpiredOrder cancels one.
func (s *orderService) CancelExpiredOrder(ctx context.Context, orderID ObjectID) error {
    expiredTime := time.Now().Add(-24 * time.Hour)
    var order models.Order

    callback := func(sessCtx mongo.SessionContext) (any, error) {
        // Cancel the order (guarded: already-cancelled or paid won't match)
        update := bson.M{
            "$set": bson.M{
                "status":                            OrderStatusCancelled,
                "payment_status":                    "expired",
                "internal_note":                     "Auto-cancelled: Payment not received within expiration period",
                "cancelled_at":                      time.Now(),
                "shop_orders.$[].shop_order_status": OrderStatusCancelled,
            },
        }
            result, err := s.db.Coll.Orders.UpdateOne(sessCtx,
                bson.M{"_id": orderID, "status": OrderStatusPending}, update)
            if result.ModifiedCount == 0 {
                // Already cancelled or paid — nothing to do
                return nil, nil
            }

            // Restore all inventory
            order, _ = s.GetOrderByID(sessCtx, orderID)
            for _, shop := range order.ShopOrders {
                for _, item := range shop.Items {
                    inventoryUpdate := bson.M{
                        "$inc": bson.M{"inventory.quantity": item.Quantity},
                    }
                    _, _ = s.db.Coll.Listings.UpdateOne(sessCtx, bson.M{"_id": item.ListingID}, inventoryUpdate)
                }
            }

            return nil, nil
        }
    }

    _, err := os.db.WithTransaction(ctx, callback)
    return err
}
```

A `PendingOrdersJob` runs every six hours and publishes an expiration event that a consumer acts on. Orders expire after 24 hours of unpaid `pending` — and a reminder fires at the 12-hour mark so the customer can still pay before the cutoff. That window gives customers enough time to complete bank transfers (which can take hours in Nigeria), but not long enough to hold inventory hostage indefinitely.

The handler is careful not to kill a paid order that *looks* pending. It re-syncs each order's payment status first (`VerifyAndUpdatePaidOrder`): if the payment actually landed, the order gets marked paid and skipped. Only truly unpaid orders are cancelled. And the `"status": OrderStatusPending` inside the update filter keeps the whole thing idempotent — if payment came through between the scan and the update, the filter simply won't match.

Each cancellation runs in its own transaction: flip the status, restore inventory, enqueue the expired-order event, and after the commit, roll back any applied coupons.

---

## Seller Payout Calculation

Each shop order tracks exactly what the seller receives:

```go
type SellerPayout struct {
    Amount         int64 `bson:"amount"`          // Customer paid (kobo)
    PlatformFee    int64 `bson:"platform_fee"`    // Khoomi's cut (kobo)
    TransactionFee int64 `bson:"transaction_fee"` // Payment processor (kobo)
    NetAmount      int64 `bson:"net_amount"`      // Seller receives (kobo)
    PayoutStatus   string `bson:"payout_status"`
}
```

Calculated at checkout:

```go
shopTotal := shopSubtotal + buyerFacingShipping + handlingFee - shopDiscount

// Fees carve off the item subtotal, not the shop total
platformFee := shopSubtotal * PLATFORM_FEE_RATE / PERCENT_DIVISOR / 100
transactionFee := shopSubtotal * TRANSACTION_FEE_RATE / PERCENT_DIVISOR / 100
netAmount := shopTotal - platformFee - transactionFee
```

Stamped at checkout so a seller viewing an incoming order sees roughly what they'll get — "You'll get ₦54,000 from this ₦60,000 order." But the *authoritative* number comes later: at payment, the settlement engine recomputes the three-pot split (earnings, shipping escrow, fees) and asserts it matches the stamped breakdown before crediting the wallet. The `NetAmount` above is display-only; the wallet is credited the settlement's earnings slice.

Why calculate at checkout instead of delivery? Transparency. No surprises when withdrawal time comes.

The trade-off? Fee changes don't apply retroactively. If I lower the platform fee tomorrow, orders placed today still use today's rate. For accounting consistency, that's actually what I want.

---

## What I'd Do Differently

**The parent order status — done.** This used to be a wishlist item: `Order.Status` mixed meanings whenever shops diverged. The code now derives the parent status — when every shop order reaches the same status, the parent follows. The genuinely mixed case (one shipped, one processing) still has no single label, and that's correct: there isn't one.

**Refund settlement coupling.** The reverse split (wallet, escrow, platform fees) is correct now, but it lives spread across three services and a hand-written invariant check that fails closed on mismatch. If the money model evolves again, that's the file I expect to touch. I'd like the split rules in one place with property tests over the invariants.

**Refund status visibility — mostly solved.** Refund requests now live in their own collection, so support's "all failed refunds this week" query is a normal indexed query instead of a scan. The remaining smell is that the *processing* state still lives embedded on the shop order — a reasonable place for single-order views, but it means the scheduler and the request collection must agree on whose refund is whose.

---

## The Pattern

Same philosophy as the earlier posts: optimize for the common case.

- Most orders have 1-2 shops → embed, don't normalize
- Most checkouts succeed → reserve at checkout, not cart add
- Most shops fulfill successfully → escrow by default, release on delivery
- Most cancellations are partial → per-shop status, not all-or-nothing
- Most refunds succeed on first try → async processing with retry, not inline

The multi-vendor order system handles the 90% case elegantly. The 10% (complex disputes, multi-shop returns, refunds that fail after 5 retries) requires me to step in manually. At Khoomi's current scale, that's maybe one or two cases a week. Acceptable.

---

*Next: Wallets and seller withdrawals.*

—Samuel

---

*Edits:*

- *2026-08-19: Brought the post in line with the current code: the checkout transaction also records coupon usage and stock events; "independent fulfillment" now shows the concurrency guard and the derived parent status (the old "what I'd do differently" item, now done); added the settlement split — seller earnings, shipping escrow, and platform fees are kept in separate pots and refunds claw back each one; partial cancellation runs through a cancel plan with a carrier-pickup guard; expired orders are handled by a six-hour scheduler with a 12-hour reminder and a payment re-check first; fees are computed on the item subtotal and `net_amount` is display-only, since the wallet is credited the settlement's earnings slice.*
