---
layout: post
title: "Building Khoomi: Shop Architecture"
description: "How shops work as separate entities from users, atomic creation transactions, follow counters, and the decisions that make a marketplace seller experience."
image: https://sot.dev/assets/images/boromir.webp
date: 2026-01-23
category: khoomi
---

Earlier in this series I covered [listing architecture](/khoomi-listing-architecture.html). This one is about the shops that own those listings.

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

Creating a shop touches three collections in one transaction:

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

        // 3. Create wallet for earnings
        _, err = s.wallet.CreateWallet(ctx, models.CreateWalletParams{ShopID: shopID, UserID: ownerID})
        if err != nil {
            return primitive.NilObjectID, err
        }

        return shopID, nil
    }

    _, err := database.ExecuteTransaction(ctx, s.db.MongoClient, callback)
    return shopID, err
}
```

Three operations, one transaction. If the wallet creation fails, the shop insert and the user update roll back. No half-created sellers.

Why a transaction instead of eventual consistency? A user with `is_seller: true` but no shop would break the dashboard. A shop without a wallet can't receive payments. These invariants must hold at all times.

The trade-off? MongoDB transactions require replica sets. Single-node development setups need extra configuration. And transactions are slower than individual writes. For shop creation (once per seller, ever), the latency is acceptable.

After the transaction commits, we purge the seller's caches and publish a `ShopCreatedEvent` for the follow-up work—fan-out, analytics, order routing. That's the stuff that can safely happen outside the atomic boundary.

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
    ShopStatusRejected      = "rejected"      // Review refused onboarding
)
```

The state machine:

```
INACTIVE → ACTIVE → PENDING_REVIEW → WARNING → BANNED
                 \→ SUSPENDED
                 \→ REJECTED
```

Most shops stay `active` forever. `inactive` gives sellers time to configure branding before going live. `pendingreview` lets moderation investigate without immediately punishing. `suspended` is reversible; `banned` is terminal.

Why not just `active` and `banned`? Nuance. A shop selling knockoffs needs immediate suspension. A shop whose profiles got flagged at review needs a `rejected`, not a ban. The intermediate states let us match response to severity.

Status isn't the only visibility gate anymore. Shops also carry `is_live`—a seller-controlled toggle, gated on identity verification—and `ready_to_sell`, a system-computed flag that's true only when the address, payment info, and a shipping profile all exist on an active shop. Public discovery requires both. So `status` says what the platform allows; the flags say what the seller has actually finished setting up.

---

## Following (Collection + Counter)

Followers live in a `shop_followers` collection. The shop document only keeps a counter:

```go
type Shop struct {
    // ...
    FollowerCount int `bson:"follower_count"`
}
```

When someone follows a shop, two writes happen in one transaction:

```go
func (s *service) FollowShop(ctx context.Context, params models.UserShopParams) (bson.ObjectID, error) {
    followerID := bson.NewObjectID()

    callback := func(sessCtx context.Context) (any, error) {
        // 1. Insert the follower record
        _, err := s.db.Coll.ShopFollowers.InsertOne(sessCtx, shopMemberData)
        if err != nil {
            if mongo.IsDuplicateKeyError(err) {
                return bson.NilObjectID, errors.New("already following this shop")
            }
            return bson.NilObjectID, err
        }

        // 2. Bump the counter
        _, err = s.db.Coll.Shops.UpdateOne(sessCtx,
            bson.M{"_id": shopID},
            bson.M{"$set": bson.M{"modified_at": now}, "$inc": bson.M{"follower_count": 1}})
        if err != nil {
            return bson.NilObjectID, err
        }

        return followerID, nil
    }

    _, err := s.db.WithTransaction(ctx, callback)
    if err != nil {
        return bson.NilObjectID, err
    }
    return followerID, nil
}
```

A duplicate follow hits MongoDB's unique index and fails the whole transaction. Unfollow is the mirror image—delete the record, decrement the counter. Because both writes always move together, the counter can't drift from the list.

"Recent followers" on the profile page is a paginated query against `shop_followers`, newest first. The `Followers` field on the `Shop` struct is `bson:"-"`—populated on read, never stored.

Why a collection instead of embedding a bounded array (as I'd originally planned)? A per-shop `shop_id, joined_at` query is indexed and cheap, and it's not a hot read path like listing views. And the dedicated collection earns its keep elsewhere: the feed subsystem queries it to figure out who to notify whenever a shop posts.

The trade-off? The profile page is technically two queries instead of one. Against everything else a shop page loads, that's a rounding error.

---

## Shipping Profiles

Each shop can have multiple shipping profiles:

```go
type ShopShippingProfile struct {
    ID                bson.ObjectID `bson:"_id"`
    ShopID            bson.ObjectID `bson:"shop_id"`
    Title             string        `bson:"title"`           // "Standard Shipping"
    OriginState       string        `bson:"-"`               // "Lagos" — joined, not stored
    DestinationBy     string        `bson:"destination_by"`  // "state" | "zone" | "everywhere"
    Destinations      []string      `bson:"destinations"`
    Shipping          Shipping      `bson:"service"`         // "fez" | "shipbubble"
    Processing        ShippingProcessing `bson:"processing"`
    PrimaryPrice      int64         `bson:"primary_price"`   // Same zone (kobo)
    SecondaryPrice    int64         `bson:"secondary_price"` // Different zone (kobo)
    MinDeliveryDays   int           `bson:"min_delivery_days"`
    MaxDeliveryDays   int           `bson:"max_delivery_days"`
    HandlingFee       int64         `bson:"handling_fee"`
    OffersFreeShipping bool         `bson:"offers_free_shipping"`
    IsDefault         bool          `bson:"is_default"`
}
```

The dual pricing (`PrimaryPrice` vs `SecondaryPrice`) still reflects Nigerian geography. Shipping within Lagos is cheaper than shipping from Lagos to Benue. Rather than model all 36 states individually, the base calculation uses two tiers: same zone vs different zone. A jewelry maker in Lagos charges ₦1,500 for Lagos delivery, ₦3,000 everywhere else.

The two tiers are the *fallback*, though. Each profile rides on a shipping service (`FEZ` or Shipbubble), and at cart/checkout the service computes live rates from the destination state and the package weight when it can. The profile's prices are what you get when the service can't—or when the seller just prefers flat rates.

Why not per-state pricing? Complexity. Most sellers ship from one location. Two tiers cover 90% of cases. For sellers who do need targeting, `DestinationBy` restricts a profile to a state, a zone, or everywhere, and `Destinations` names the targets. But most sellers just use "everywhere."

The trade-off? All this is a lot of surface area on one document. Two pieces of polish keep it sane: `MinDeliveryDays`/`MaxDeliveryDays` give the buyer an honest "arrives in 2–4 days," and returns are *not* here anymore—they moved to a separate shop policy (`accepts_return`, `accepts_exchanges`, `return_deadline`), with individual listings able to override.

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

**The follower counter.** `follower_count` is a denormalized number that has to be incremented in the same transaction as the follow insert. It works, and the transaction keeps it honest. But it's the kind of invariant that silently corrupts the moment someone forgets the unit of work. Deriving it from the collection—or better, from the stats document below—would remove the coupling entirely.

**The notification count queries.** Six parallel queries is fast, but it's still six round trips to MongoDB. A dedicated "shop stats" document updated via change streams would be faster. I'd pay for it with eventual consistency, but dashboard counts don't need to be real-time accurate.

---

## The Pattern

Same philosophy as listings: optimize for the common case.

- Most users own one shop → enforce 1:1, don't build multi-shop support
- Most shop pages show few followers → query the recent few, paginate the rest
- Most sellers ship two tiers → primary/secondary pricing, not per-state
- Most dashboard loads need all counts → parallelize, don't waterfall

The shop architecture is stable. New features—analytics dashboards, promotional tools, bulk listing management—attach cleanly to the existing model.

---

*Next: [Multi-Vendor Order Architecture](/khoomi-multi-vendor-orders.html)*

—Samuel

---

*Edits:*

- *2026-08-19: Brought the article in line with the current code: shop creation is three collections (notification settings are gone) plus a post-commit event; added `rejected` and the `is_live`/`ready_to_sell` visibility gates; followers are a separate collection with a transactional counter instead of an embedded array; shipping profiles grew `destination_by`, services, handling fees, and free-shipping — and returns moved to a shop policy.*
