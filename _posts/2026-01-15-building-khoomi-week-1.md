---
layout: post
title: "Building Khoomi - Week 1: Listing Architecture Decisions"
description: "Deep dive into MongoDB document design, stock reservation strategies, and polymorphic category data for an African marketplace."
image: https://sot.dev/assets/images/this-is-fine.jpg
date: 2026-01-15
category: khoomi
---

## Introducing "Building Khoomi"

I've been building [Khoomi](/building-khoomi) for 3.5 years now, a marketplace for Nigerian artisans and creatives. Nigeria's e-commerce landscape has unique constraints: local payment rails, unreliable logistics, and infrastructure gaps that global platforms ignore. Along the way, I've made thousands of technical decisions, some smart, some I'm still fixing.

This series is where I share what I've learned: the actual code, the trade-offs, and what I'd do differently with hindsight.

Each week, I'll cover one piece of the system: listings, wallets, shipping, payments, all of it. If you're building something similar, or just curious how a marketplace actually works under the hood, this is for you.

---

Every marketplace lives or dies by how it handles listings. After 3.5 years developing Khoomi (MVP coming soon), I've made dozens of decisions about how listings work. Some were obvious. Most weren't.

Here's what I learned.

---

## The Core Model

A listing in Khoomi is a single MongoDB document. One document = one product. This sounds obvious until you consider the alternatives.

```go
type Listing struct {
    ID          primitive.ObjectID
    Code        string              // "ABCD-1234"
    Slug        string              // URL-friendly
    ShopID      primitive.ObjectID
    Title       string
    Description string
    Inventory   Inventory
    Variations  []Variation
    Details     ListingDetails      // Category-specific data
    State       ListingState
    Rating      Rating
    // ...
}
```

I chose embedded documents over references. A listing with 10 color variations stores all 10 in a single `variations` array, not in a separate `variations` collection.

Why? Atomic updates. When a customer buys "Large / Red", I decrement that variation's quantity in one database operation. No distributed transactions. No race conditions between collections.

The trade-off? MongoDB's 16MB document limit caps me at roughly 5,000 variations per listing. For handmade goods, that's never a problem.

---

## Two Prices, Not One

Every variation can override the base price:

```go
type Variation struct {
    ID       string
    Name     string   // "Size"
    Value    string   // "Large"
    Quantity int
    Price    *int64   // nil = use base price, 0 = free
}
```

The `Price` field is a pointer. If `nil`, inherit from `inventory.price`. If set, use the override.

This lets a seller price a leather bag at ₦15,000 for small and ₦18,000 for large without duplicating the entire listing. Simple idea, but the pointer-vs-value distinction matters. An explicit zero means free. A `nil` means "same as base."

---

## Stock Reservation: Later, Not Sooner

When should you reserve inventory? Two options:

1. **Reserve on cart add** — Item is "held" while in cart
2. **Reserve on checkout** — Item remains available until payment

I chose option 2.

Why? Carts are abandoned constantly. Reserving on cart add means expiry logic: "Release after 15 minutes of inactivity." That creates edge cases. What if someone's mid-checkout when the hold expires? What if they have unreliable internet?

![Abandoned cart meme](/assets/images/abandoned-cart.jpg)

Instead, multiple customers can cart the same item. At checkout, I verify and reserve atomically:

```go
func (s *CheckoutService) ReserveStock(ctx context.Context, item CartItem) error {
    result, err := s.listings.UpdateOne(ctx,
        bson.M{
            "_id": item.ListingID,
            "variations": bson.M{
                "$elemMatch": bson.M{
                    "id":       item.VariationID,
                    "quantity": bson.M{"$gte": item.Quantity},
                },
            },
        },
        bson.M{
            "$inc": bson.M{
                "variations.$.quantity": -item.Quantity,
            },
        },
    )
    if result.MatchedCount == 0 {
        return ErrInsufficientStock
    }
    return err
}
```

The query filter ensures stock exists *before* decrementing. If someone else bought it first, the update matches nothing and checkout fails with a clear message.

The trade-off? Worse UX when items sell out while browsing. But that's better than the alternative: customers whose "held" items vanish mid-checkout.

---

## Expiration via Background Job

Listings expire after 30 days. I could check this on every read:

```go
// Option A: Check on read
if listing.ExpiresAt.Before(time.Now()) {
    return ErrListingExpired
}
```

Instead, I run a daily job that bulk-updates expired listings:

```go
// Option B: Background job
func (s *ListingService) MarkExpiredListings(ctx context.Context) error {
    _, err := s.collection.UpdateMany(ctx,
        bson.M{
            "state":      "active",
            "expires_at": bson.M{"$lt": time.Now()},
        },
        bson.M{"$set": bson.M{"state": "expired"}},
    )
    return err
}
```

Why? Reads are frequent. Writes are rare. Pushing expiration logic into reads adds latency to every listing view. A background job handles it once, in bulk, off the critical path.

The trade-off? A listing might display as "active" for up to 24 hours past its expiration.

![This is fine meme](/assets/images/this-is-fine.jpg)

For a marketplace selling handmade goods, that's acceptable. For concert tickets, it wouldn't be.

---

## Polymorphic Category Data

Clothing needs size charts. Furniture needs dimensions. Jewelry needs materials. How do you model this?

I use a typed dynamic field:

```go
type ListingDetails struct {
    Category      Category
    DynamicType   string                 // "clothing", "furniture"
    Dynamic       map[string]interface{} // Raw data
    ClothingData  *Clothing              // Typed struct
    FurnitureData *Furniture             // Typed struct
    // ...
}
```

The `Dynamic` map stores whatever the frontend sends. The typed struct (`ClothingData`, `FurnitureData`) is populated by parsing that map at runtime.

Why not separate collections per category? Because listings change categories. A seller might realize their "home decor" item belongs under "art". With embedded polymorphic data, that's a field update. With separate collections, it's a migration.

The trade-off? I can't index inside the dynamic fields. Searching "all listings with cotton fabric" requires application-level filtering, not a database index. For Khoomi's scale, that's fine. For Jumia, it wouldn't be.

---

## What I'd Do Differently

**Slugs.** I generate URL slugs from titles. When titles change, slugs don't update automatically—to preserve existing links. This creates stale URLs over time. I should have made slugs immutable from day one, or built a redirect system earlier.

**Analytics denormalization.** I store `views` and `favorers_count` directly on the listing document for fast reads. But updating these counters on every view creates write contention. I'm now migrating to a separate analytics collection with periodic sync back to listings.

---

## The Pattern

Most of these decisions follow a pattern: optimize for the common case, accept trade-offs for edge cases.

- Most listings have <20 variations → embed them
- Most carts are abandoned → don't reserve stock early
- Most reads don't need millisecond-accurate expiration → use background jobs
- Most category changes are rare → use polymorphic embedding

The key is knowing your domain. Khoomi sells handmade goods. Low volume per listing. High variety across listings. Patient buyers. These constraints shaped every decision.

Your marketplace might be different. That's the point.

---

*Next: [Week 2 - Shop Architecture](/building-khoomi-week-2.html)*

—Samuel
