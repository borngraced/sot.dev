---
layout: post
title: "Building Khoomi - Week 3: Multi-Vendor Order Architecture"
description: "How orders work when a single checkout spans multiple sellers: parent-child structure, atomic inventory reservation, independent fulfillment, and escrow via wallet."
image: https://sot.dev/assets/images/order-refund-complete.png
date: 2026-01-30
category: khoomi
---

Last week I documented [shop architecture](/building-khoomi-week-2.html). This week, I will be writing on  how orders work when a customer buys from multiple shops in one checkout on Khoomi.

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

The parent `Order` tracks payment. Each `ShopOrder` tracks its own fulfillment. For an example, Shop `A` marking their portion as shipped doesn't affect Shop `B's` status(stays as "processing.", "paid" or whatever status it is)

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

Why reserve at checkout instead of cart add? I covered this decision in [Week 1](/building-khoomi-week-1.html#stock-reservation-later-not-sooner). The short version: carts are abandoned constantly, and reserving at checkout avoids expiry logic and hold timers.

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
    filter := bson.M{
        "_id":                 params.OrderID,
        "shop_orders.shop_id": params.ShopID,
    }

    update := bson.M{
        "$set": bson.M{
            "shop_orders.$.shop_order_status": params.Status,
            "updated_at":                      time.Now(),
        },
    }

    if params.Status == OrderStatusShipped {
        update["$set"].(bson.M)["shop_orders.$.shipped_at"] = time.Now()
    }

    callback := func(sessCtx mongo.SessionContext) (any, error) {
        _, err := s.db.Coll.Orders.FindOneAndUpdate(sessCtx, filter, update).Decode(&order)
        if err != nil {
            return nil, err
        }

        // On delivery, release earnings to seller
        if params.Status == OrderStatusDelivered {
            err := s.wallet.ReleaseEarnings(sessCtx, params.ShopID, params.OrderID)
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

The `shop_orders.$` positional operator updates only the matching shop order within the parent document.

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
}
```

The refund starts as `pending`. A background job picks it up.

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

The refund deducts from both `pending_balance` and `total_earnings`. The seller never actually earned this money. It's going back to the customer.

![Order refund completed](/assets/images/order-refund-complete.png)

The trade-off? If the seller somehow withdrew before delivery (which shouldn't happen, since pending balance can't be withdrawn), the deduction fails. After 5 retries, the refund is marked as failed, and I get an alert to handle it manually.

---

## Cleanup Expired Orders

Pending orders that never get paid should release their inventory:

```go
func (s *orderService) CleanupExpiredPendingOrders(ctx context.Context, expirationHours int) (int64, error) {
    expiredTime := time.Now().Add(-time.Duration(expirationHours) * time.Hour)

    filter := bson.M{
        "status":     OrderStatusPending,
        "created_at": bson.M{"$lt": expiredTime},
    }

    cursor, _ := s.db.Coll.Orders.Find(ctx, filter)
    var expiredOrders []Order
    cursor.All(ctx, &expiredOrders)

    var cleanedCount int64
    for _, order := range expiredOrders {
        callback := func(sessCtx mongo.SessionContext) (any, error) {
            // Cancel the order
            update := bson.M{
                "$set": bson.M{
                    "status":                            OrderStatusCancelled,
                    "payment_status":                    "expired",
                    "internal_note":                     fmt.Sprintf("Auto-cancelled: Payment not received within %d hours", expirationHours),
                    "shop_orders.$[].shop_order_status": OrderStatusCancelled,
                },
            }
            result, err := s.db.Coll.Orders.UpdateOne(sessCtx,
                bson.M{"_id": order.ID, "status": OrderStatusPending}, update)
            if result.ModifiedCount == 0 {
                // Already processed
                return nil, nil 
            }

            // Restore all inventory
            for _, shop := range order.ShopOrders {
                for _, item := range shop.Items {
                    inventoryUpdate := bson.M{
                        "$inc": bson.M{"inventory.quantity": item.Quantity},
                    }
                    _, _ = s.db.Coll.Listings.UpdateOne(sessCtx, bson.M{"_id": item.ListingID}, inventoryUpdate)
                }
            }

            return result.ModifiedCount, nil
        }

        result, _ := database.ExecuteTransaction(ctx, s.db.MongoClient, callback)
        if count, ok := result.(int64); ok {
            cleanedCount += count
        }
    }

    return cleanedCount, nil
}
```

This runs as a background job every 24 hours. That window gives customers enough time to complete bank transfers (which can take hours in Nigeria), but not long enough to hold inventory hostage indefinitely.

The double-check filter `"status": OrderStatusPending` in the update ensures idempotency. If payment came through between the find and update, the order won't be cancelled. The filter simply won't match.

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
shopTotal := shopSubtotal + shippingCost + handlingFee - shopDiscount

platformFee := shopTotal * PLATFORM_FEE_RATE / PERCENT_DIVISOR / 100
transactionFee := shopTotal * TRANSACTION_FEE_RATE / PERCENT_DIVISOR / 100
netAmount := shopTotal - platformFee - transactionFee

shopOrder.SellerPayout = SellerPayout{
    Amount:         shopTotal,
    PlatformFee:    platformFee,
    TransactionFee: transactionFee,
    NetAmount:      netAmount,
    PayoutStatus:   "pending",
}
```

Why calculate at checkout instead of delivery? Transparency. When a seller views an incoming order, they see exactly what they'll receive. No surprises when withdrawal time comes. "You'll get ₦54,000 from this ₦60,000 order" is clear.

The trade-off? Fee changes don't apply retroactively. If I lower the platform fee tomorrow, orders placed today still use today's rate. For accounting consistency, that's actually what I want.

---

## What I'd Do Differently

**The parent order status.** Currently `Order.Status` is supposed to be the "overall" status, but what does "processing" mean when Shop A is shipped and Shop B is still packing? I should probably derive it from shop order statuses instead of storing it separately. Or at least define clearer rules for when the parent status changes.

**Refund status visibility.** Refund status lives inside `ShopOrder.Refund`. That's fine for displaying a single order, but when customer support asks "show me all failed refunds this week," I have to scan every order document. A separate refunds collection with proper indexing would make those queries instant.

---

## The Pattern

Same as previous weeks: optimize for the common case.

- Most orders have 1-2 shops → embed, don't normalize
- Most checkouts succeed → reserve at checkout, not cart add
- Most shops fulfill successfully → escrow by default, release on delivery
- Most cancellations are partial → per-shop status, not all-or-nothing
- Most refunds succeed on first try → async processing with retry, not inline

The multi-vendor order system handles the 90% case elegantly. The 10% (complex disputes, multi-shop returns, refunds that fail after 5 retries) requires me to step in manually. At Khoomi's current scale, that's maybe one or two cases a week. Acceptable.

---

*Next week: Wallets and seller withdrawals.*

—Samuel
