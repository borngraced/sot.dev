---
layout: post
title: "Building Khoomi - Week 2: Shop Architecture"
description: "How shops work as separate entities from users, atomic creation transactions, embedded followers, and the decisions that make a marketplace seller experience."
image: https://sot.dev/assets/images/boromir.webp
date: 2026-01-23
category: khoomi
---

Last week I documented [listing architecture](/building-khoomi-week-1.html). This week: the shops that own those listings.

On Khoomi, a shop is a seller's storefront—distinct from the user account that handles authentication. This separation lets the same person buy (via their user account) and sell (via their shop) while keeping concerns cleanly separated.

---

## One User, One Shop

Every user can own exactly one shop. This constraint is intentional:

```go
type Shop struct {
    ID                 primitive.ObjectID `bson:"_id"`
    UserID             primitive.ObjectID `bson:"user_id"`  // Owner
    Name               string             `bson:"name"`
    Username           string             `bson:"username"` // Unique handle
    Status             ShopStatus         `bson:"status"`
    ListingActiveCount int64              `bson:"listing_active_count"`
    FollowerCount      int                `bson:"follower_count"`
    Rating             Rating             `bson:"rating"`
    // ...
}
```

Why one-to-one? Earnings are unambiguously credited. Tax reporting is straightforward. Sellers focus on building a single strong brand rather than fragmenting their efforts across multiple storefronts.

The trade-off? Users who genuinely need multiple shops (say, one for jewelry and one for clothing) must create separate accounts. For Khoomi's target market—individual artisans—this rarely comes up. Large multi-brand sellers would need a different architecture.

---

## Atomic Shop Creation

Creating a shop touches four collections in one transaction:

```go
func (s *ShopServiceImpl) CreateShop(ctx context.Context, req CreateShopRequest, ownerID ObjectID) (ObjectID, error) {
    shopID := primitive.NewObjectID()

    callback := func(ctx mongo.SessionContext) (any, error) {
        // 1. Insert the shop
        _, err := s.db.Coll.Shops.InsertOne(ctx, shop)
        if err != nil {
            return primitive.NilObjectID, err
        }

        // 2. Update user to seller status
        _, err = s.db.Coll.Users.UpdateOne(ctx,
            bson.M{"_id": ownerID},
            bson.M{"$set": bson.M{
                "is_seller":              true,
                "seller_onboarding_level": OnboardingLevelCreatedShop,
                "shop_id":                shopID,
            }})
        if err != nil {
            return primitive.NilObjectID, err
        }

        // 3. Create notification settings
        _, err = s.CreateShopNotificationSettings(ctx, shopID)
        if err != nil {
            return primitive.NilObjectID, err
        }

        // 4. Create wallet for earnings
        _, err = s.wallet.CreateWallet(ctx, shopID, ownerID)
        if err != nil {
            return primitive.NilObjectID, err
        }

        return shopID, nil
    }

    _, err := database.ExecuteTransaction(ctx, s.db.MongoClient, callback)
    return shopID, err
}
```

Four operations, one transaction. If the wallet creation fails, the user update rolls back. If the notification settings fail, the shop insert rolls back. No half-created sellers.

Why a transaction instead of eventual consistency? A user with `is_seller: true` but no shop would break the dashboard. A shop without a wallet can't receive payments. These invariants must hold at all times.

The trade-off? MongoDB transactions require replica sets. Single-node development setups need extra configuration. And transactions are slower than individual writes. For shop creation (once per seller, ever), the latency is acceptable.

![One does not simply create a shop without a transaction](/assets/images/boromir.webp)

---

## Shop Status Lifecycle

Shops progress through statuses that control visibility and capabilities:

```go
const (
    ShopStatusInactive      = "inactive"      // Just created, not visible
    ShopStatusActive        = "active"        // Fully operational
    ShopStatusPendingReview = "pendingreview" // Flagged for moderation
    ShopStatusWarning       = "warning"       // Minor violation confirmed
    ShopStatusSuspended     = "suspended"     // Temporarily disabled
    ShopStatusBanned        = "banned"        // Permanently removed
)
```

The state machine:

```
INACTIVE → ACTIVE → PENDING_REVIEW → WARNING → BANNED
                 \→ SUSPENDED
```

Most shops stay `active` forever. `inactive` gives sellers time to configure branding before going live. `pendingreview` lets moderation investigate without immediately punishing. `suspended` is reversible; `banned` is terminal.

Why not just `active` and `banned`? Nuance. A shop selling knockoffs needs immediate suspension. A shop with unclear product descriptions needs a warning, not a ban. The intermediate states let us match response to severity.

---

## Embedded Followers (Bounded)

The shop document embeds the 5 most recent followers:

```go
type Shop struct {
    // ...
    Followers     []ShopFollower `bson:"followers"`     // Embedded (max 5)
    FollowerCount int            `bson:"follower_count"`
}
```

When someone follows a shop:

```go
func (s *ShopServiceImpl) FollowShop(ctx context.Context, userID, shopID ObjectID) (ObjectID, error) {
    callback := func(ctx mongo.SessionContext) (any, error) {
        // Insert into followers collection (full list)
        _, err := s.db.Coll.ShopFollowers.InsertOne(ctx, followerData)
        if err != nil {
            if mongo.IsDuplicateKeyError(err) {
                return nil, errors.New("already following this shop")
            }
            return nil, err
        }

        // Update shop with bounded embedded array
        update := bson.M{
            "$push": bson.M{
                "followers": bson.M{
                    "$each":     bson.A{followerExcerpt},
                    "$sort":     bson.M{"joined_at": -1},
                    "$slice":    5,        // Keep only 5
                    "$position": 0,        // Prepend (newest first)
                },
            },
            "$inc": bson.M{"follower_count": 1},
        }
        _, err = s.db.Coll.Shops.UpdateOne(ctx, bson.M{"_id": shopID}, update)
        return followerID, err
    }

    return database.ExecuteTransaction(ctx, s.db.MongoClient, callback)
}
```

The `$slice: 5` operator caps the embedded array. The full follower list lives in `shop_followers` collection for pagination.

Why embed at all? The shop profile page shows "recent followers" without a join. One document fetch, one render. For a shop with 10,000 followers, loading the full list would be wasteful when we only display 5.

The trade-off? Two writes per follow (collection + embedded). The embedded data can drift if updates fail partially—though the transaction prevents this. And we're duplicating data. For 5 followers × ~100 bytes each, the duplication is negligible.

---

## Shipping Profiles

Each shop can have multiple shipping profiles:

```go
type ShopShippingProfile struct {
    ID              ObjectID `bson:"_id"`
    ShopID          ObjectID `bson:"shop_id"`
    Title           string   `bson:"title"`           // "Standard Shipping"
    OriginState     string   `bson:"origin_state"`    // "Lagos"
    PrimaryPrice    int64    `bson:"primary_price"`   // Same zone (kobo)
    SecondaryPrice  int64    `bson:"secondary_price"` // Different zone (kobo)
    MinDeliveryDays int      `bson:"min_delivery_days"`
    MaxDeliveryDays int      `bson:"max_delivery_days"`
    IsDefault       bool     `bson:"is_default"`
    AcceptReturns   bool     `bson:"accept_returns"`
    ReturnPeriod    int      `bson:"return_period"`   // Days
}
```

The dual pricing (`PrimaryPrice` vs `SecondaryPrice`) reflects Nigerian geography. Shipping within Lagos is cheaper than shipping from Lagos to Benue. Rather than model all 36 states individually, I use two tiers: same zone vs different zone.

Why not per-state pricing? Complexity. Most sellers ship from one location. Two tiers cover 90% of cases. A jewelry maker in Lagos charges ₦1,500 for Lagos delivery, ₦3,000 everywhere else. Good enough.

The trade-off? Sellers shipping to specific states (say, only Southwest Nigeria) need workarounds. The `Destinations` field exists for this, but most sellers just use "everywhere."

---

## Dashboard Notification Counts

The seller dashboard needs multiple counts: unread messages, new orders, pending refunds, low stock items. Six queries total. Running them sequentially would be slow.

Instead, parallel goroutines with channels:

```go
func (s *ShopServiceImpl) GetShopNotificationCount(ctx context.Context, shopID ObjectID) (*ShopNotificationCount, error) {
    result := &ShopNotificationCount{}
    resultChan := make(chan countResult, 6)

    // 1. Unread messages
    go func() {
        count := s.countUnreadMessages(ctx, shopID)
        resultChan <- countResult{"unread_messages", count, nil}
    }()

    // 2. New orders (pending or paid)
    go func() {
        count := s.countNewOrders(ctx, shopID)
        resultChan <- countResult{"new_orders", count, nil}
    }()

    // 3. Pending refunds
    go func() {
        count, _ := s.db.Coll.RefundRequests.CountDocuments(ctx,
            bson.M{"shop_id": shopID, "status": "pending"})
        resultChan <- countResult{"pending_refunds", count, nil}
    }()

    // 4. Low stock items (inventory <= 5)
    go func() {
        count, _ := s.db.Coll.Listings.CountDocuments(ctx, bson.M{
            "shop_id":            shopID,
            "state.state":        ListingStateActive,
            "inventory.quantity": bson.M{"$lte": 5, "$gt": 0},
        })
        resultChan <- countResult{"low_stock_items", count, nil}
    }()

    // 5. Expiring listings (within 7 days)
    go func() {
        sevenDaysFromNow := time.Now().AddDate(0, 0, 7)
        count, _ := s.db.Coll.Listings.CountDocuments(ctx, bson.M{
            "shop_id":     shopID,
            "state.state": ListingStateActive,
            "expires_at":  bson.M{"$lte": sevenDaysFromNow},
        })
        resultChan <- countResult{"expiring_listings", count, nil}
    }()

    // 6. Unread notifications
    go func() {
        count, _ := s.db.Coll.ShopNotifications.CountDocuments(ctx,
            bson.M{"recipient_id": shopID, "is_read": false})
        resultChan <- countResult{"unread_notifications", count, nil}
    }()

    // Collect results
    for i := 0; i < 6; i++ {
        r := <-resultChan
        switch r.field {
        case "unread_messages":
            result.UnreadMessages = r.count
        case "new_orders":
            result.NewOrders = r.count
        // ... etc
        }
    }

    return result, nil
}
```

Six queries run concurrently. Total latency is the slowest query, not the sum of all queries. On my test data, this drops dashboard load from ~600ms to ~150ms.

![I am speed](/assets/images/i-am-speed.png)

Why not cache these counts? They change constantly. An order comes in, the count increments. A message arrives, unread count changes. Caching would show stale data. For a seller checking their dashboard, accuracy matters more than the 150ms saved by caching.

---

## Vacation Mode

Sellers can pause their shop without deleting anything:

```go
type Shop struct {
    // ...
    IsVacation      bool   `bson:"is_vacation"`
    VacationMessage string `bson:"vacation_message"`
}
```

When `IsVacation` is true:
- Shop is hidden from search results
- Shop page shows the vacation message
- Existing orders continue processing
- Listings remain intact

Why a flag instead of a status? Vacation is orthogonal to status. An `active` shop can go on vacation. A shop under `warning` can also go on vacation. Mixing these into one field would create a combinatorial explosion: `active_vacation`, `warning_vacation`, etc.

The trade-off? Two fields to check instead of one. Queries for "visible shops" need `status: active AND is_vacation: false`. Minor complexity.

---

## Shop Announcements

Sellers can post announcements that appear at the top of their shop page:

```go
type Shop struct {
    // ...
    Announcement           string    `bson:"announcement"`
    AnnouncementModifiedAt time.Time `bson:"announcement_modified_at"`
}
```

Announcements are capped at 500 characters—enough for a sale notice or shipping delay warning, not enough for a newsletter. The `AnnouncementModifiedAt` timestamp lets the frontend show "Updated 2 days ago" without a separate query.

Why not a separate announcements collection with history? Overkill. Sellers update announcements infrequently. When they do, the old one doesn't matter. A single field with a timestamp covers the use case.

---

## What I'd Do Differently

**The 5-follower embedding.** It works, but it's a magic number. If the design team wants to show 10 followers on the profile page, I need a migration. A configurable limit or a separate "recent followers" query might be more flexible.

**The notification count queries.** Six parallel queries is fast, but it's still six round trips to MongoDB. A dedicated "shop stats" document updated via change streams would be faster. I'd pay for it with eventual consistency, but dashboard counts don't need to be real-time accurate.

---

## The Pattern

Same philosophy as listings: optimize for the common case.

- Most users own one shop → enforce 1:1, don't build multi-shop support
- Most shop pages show few followers → embed 5, paginate the rest
- Most sellers ship two tiers → primary/secondary pricing, not per-state
- Most dashboard loads need all counts → parallelize, don't waterfall

The shop architecture is stable. New features—analytics dashboards, promotional tools, bulk listing management—attach cleanly to the existing model.

---

*Next week: How orders work when a single checkout spans multiple sellers.*

—Samuel
