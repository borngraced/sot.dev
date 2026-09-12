---
layout: post
title: "Building Khoomi: Listing Architecture Decisions"
description: "Deep dive into MongoDB document design, stock reservation strategies, and polymorphic category data for an African marketplace."
image: https://sot.dev/assets/images/this-is-fine.jpg
date: 2026-01-15
category: khoomi
---

## Introducing "Building Khoomi"

I've been building [Khoomi](/building-khoomi) for 3.5 years now, a marketplace for Nigerian artisans and creatives. Nigeria's e-commerce landscape has unique constraints: local payment rails, unreliable logistics, and infrastructure gaps that global platforms ignore. Along the way, I've made thousands of technical decisions, some smart, some I'm still fixing.

This series is where I share what I've learned: the actual code, the trade-offs, and what I'd do differently with hindsight.

The series walks through one piece of the system at a time: listings, shops, orders, wallets, shipping, payments. If you're building something similar, or just curious how a marketplace actually works under the hood, this is for you.

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
    Price    *Kobo    // nil = use base price; set prices must be > 0
}
```

The `Price` field is a pointer to a `Kobo` (naira in kobo). `nil` means inherit from the base price. A set value overrides it. Overrides have to be positive, so there's no "free via zero" — the pointer only distinguishes "has an override" from "doesn't."

There are actually two base prices on the inventory now: `inventory.price` for international buyers and `inventory.domestic_price` for local ones. The variation override stacks on whichever applies.

That's how a seller prices a leather bag at ₦15,000 for small and ₦18,000 for large without duplicating the whole listing.

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
func (s *orderService) reserveInventory(ctx context.Context, item CartItem) error {
    filter := bson.M{
        "_id":                item.ListingID,
        "inventory.quantity": bson.M{"$gte": item.Quantity},
    }
    update := bson.M{
        "$inc": bson.M{"inventory.quantity": -item.Quantity},
    }

    result, err := s.db.Coll.Listings.UpdateOne(ctx, filter, update)
    if err != nil {
        return err
    }
    if result.ModifiedCount == 0 {
        return errors.New("insufficient inventory")
    }
    return nil
}
```

The filter ensures stock exists *before* decrementing. If someone else bought it first, the update matches nothing and checkout fails with a clear message.

This takes from the listing-wide `inventory.quantity`, not from a variation row. Variations carry their own `quantity` field, but checkout doesn't decrement per-variation — it was doing that dance, with `$elemMatch` and the `$` positional operator, until it caused more trouble than it saved. Now the guarantee is against the aggregate and everything stays simple for low-volume handmade goods.

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

The job actually does two things now. First it auto-renews: a listing with `should_auto_renew` gets a fresh `expires_at` and stays active — which is what most sellers want, since nobody puts work into a listing just to watch it die. Then it expires the rest. So "30-day expiration" is really "30 days, then renew or die," decided by one flag.

The trade-off? A non-renewing listing might display as "active" for up to 24 hours past its expiration.

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

**Analytics denormalization.** I keep `favorers_count`, `in_carts_count`, and `sold_count` on the listing document itself. A listing is still one document and the numbers ride along: one read, no joins, and the storefront badges don't need anything extra. That part has held up. What I underrated is how easily a denormalized counter rots. A counter that only ever goes up is fine until somebody deletes their account, empties a cart, or cancels an order — and then you're showing "24 people love this" to nobody. I've since wired decrements into account deletion and cart cleanup so the number moves with whatever caused it. The last gap is display: listing responses are cached for up to half an hour, so a badge can trail a little. For social-proof numbers, that's fine. For live inventory, it wouldn't be.

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

*Next: [Shop Architecture](/khoomi-shop-architecture.html)*

—Samuel

---

*Edits:*

- *2026-08-19: Rewrote the analytics-denormalization note to match how the counters actually work now.*
- *2026-08-19: Fixed the price section (no "free via zero" — overrides must be positive, and the base price split into domestic/export), rewrote the reservation sample to the aggregate `inventory.quantity` checkout, and added auto-renew to the expiration story.*
