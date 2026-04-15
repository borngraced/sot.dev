---
layout: post
title: "Everything Should Be Typed: Scalar Types Are Not Enough"
description: "Why scalar types give us a false sense of safety, and how wrapping primitives in domain types catches bugs the compiler never could."
image: https://sot.dev/assets/images/everything-should-be-typed.png
date: 2026-04-13
---

Recently while working in a codebase, I had a serious bug originating from a very small and minor misuse of arguments to a function which also led to my issue in [rust-clippy](https://github.com/rust-lang/rust-clippy/issues/16735). I was ranting about it on X too:

{::nomarkdown}
<blockquote class="twitter-tweet"><p><a href="https://twitter.com/sotdev_/status/2034796732366667932">Tweet</a></p></blockquote>
{:/nomarkdown}

So I said why not write on why developers shouldn't stop at scalar types. Imagine a function takes a `string`, returns a `number`, and we call it "typed." But this is a shallow form of type safety, one that gives us a false sense of security while letting entire categories of bugs slip through unnoticed.

Let me walk you through why scalar types fail us, and what I believe a truly typed codebase should look like.

## The Positional Parameter Problem

Say I have a function that processes a seller payout after an order is delivered. It takes a shop ID, a customer ID, an order ID, the gross amount, platform fee, transaction fee, and net amount.

**JavaScript:**
```js
function processOrderPayout(shopId, customerId, orderId, amount, platformFee, txFee, netAmount) {
  // ...
}
```

**Go:**
```go
func ProcessOrderPayout(shopID string, customerID string, orderID string, amount int64, platformFee int64, txFee int64, netAmount int64) error {
    // ...
}
```

**Rust:**
```rust
fn process_order_payout(shop_id: String, customer_id: String, order_id: String, amount: i64, platform_fee: i64, tx_fee: i64, net_amount: i64) {
    // ...
}
```

Seven parameters. Three IDs that are all strings. Four money values that are all integers. Now imagine somewhere in my codebase, a caller writes this:

```rust
process_order_payout(customer_id, shop_id, order_id, net_amount, tx_fee, platform_fee, amount);
```

The customer ID went in place of the shop ID. The net amount went in place of the gross amount. The fees are swapped. The compiler doesn't complain. Tests probably pass too. The application runs, pays out the wrong entity, credits the wrong amount, and nobody notices until a seller asks why they received ₦350 instead of ₦54,000.

This is what bit me. The compiler checks the *shape* of the data, not the *meaning*. A `String` is a `String` is a `String`, and an `i64` is an `i64` is an `i64`. The type system has no way to tell a shop ID apart from a customer ID, or a gross amount from a net amount, when they share the same underlying type.

## "Just Use a Struct." Better, But Not Enough

The natural next step is grouping parameters into a struct or object.

**JavaScript:**
```js
function processOrderPayout({ shopId, customerId, orderId, amount, platformFee, txFee, netAmount }) {
  // ...
}
```

**Go:**
```go
type OrderPayoutParams struct {
    ShopID      string
    CustomerID  string
    OrderID     string
    Amount      int64
    PlatformFee int64
    TxFee       int64
    NetAmount   int64
}

func ProcessOrderPayout(params OrderPayoutParams) error {
    // ...
}
```

**Rust:**
```rust
struct OrderPayoutParams {
    shop_id: String,
    customer_id: String,
    order_id: String,
    amount: i64,
    platform_fee: i64,
    tx_fee: i64,
    net_amount: i64,
}

fn process_order_payout(params: OrderPayoutParams) {
    // ...
}
```

This is better. Named fields eliminate positional confusion. You can't accidentally swap `shop_id` and `customer_id` when you're explicitly naming them at the call site.

But we've only solved one problem. Look at what the struct *doesn't* prevent:

```rust
let params = OrderPayoutParams {
    shop_id: customer_id,       // oops, customer ID assigned to shop field
    customer_id: shop_id,       // oops, shop ID assigned to customer field
    order_id: order_id,
    amount: net_amount,         // oops, net amount assigned to gross amount field
    platform_fee: tx_fee,       // oops, fees are swapped
    tx_fee: platform_fee,
    net_amount: amount,         // oops, gross amount assigned to net field
};
```

The compiler is perfectly happy. Every string field got a `String`. Every integer field got an `i64`. The fact that `customer_id` contains a customer identifier and not a shop identifier? Invisible to the type system.

This might seem contrived, but it happens all the time in real codebases. Variables get renamed. Data flows through multiple layers. A function receives values from a database row and passes them into a struct, and nobody remembers which column mapped to which field. Someone refactors and swaps two fields, and the compiler catches zero of the call sites that now pass data into the wrong slots.

The struct gave us named assignment, but not semantic correctness. The types are still lying. They say "this field accepts a string" when what we actually mean is "this field accepts a *shop identifier*." They say "this field accepts an integer" when what we actually mean is "this field accepts a *platform fee in kobo*."

## The Deeper Problem: Primitives Erase Meaning

This goes way beyond my payout example. Scalar types like `string`, `int`, `float`, and `bool` are building blocks, but they carry no domain meaning. When your codebase passes around raw primitives everywhere, you lose the ability to reason about what data actually *means* at the type level.

Here are bugs I've either hit or seen others hit that scalar types will never catch:

**Passing a user ID where an order ID is expected.** Both are `int` or `string`. Both represent identifiers. But mixing them up means you're querying the wrong table, charging the wrong customer, or deleting the wrong record.

**Mixing up units.** A distance in meters passed to a function expecting kilometers. A price in cents passed to a function expecting dollars. A duration in seconds stored in a field labeled "minutes." All the same type, `f64` or `int`, and the compiler won't say a word.

**Confusing sanitized and unsanitized input.** A raw user-provided string passed directly into a SQL query or HTML template. The type system sees `String`. It doesn't know "this string hasn't been escaped yet." This is how injection vulnerabilities happen.

**Swapping latitude and longitude.** Both `f64`. Both coordinates. Swap them and your map renders on the wrong continent.

All of these compile. All of them might pass tests. All of them have caused real production incidents. And all of them are preventable.

## The Solution: Make Invalid States Unrepresentable

The fix is simpler than you'd think. Stop using scalar types for domain concepts. Wrap every meaningful value in its own type.

**Rust:**
```rust
struct ShopId(String);
struct CustomerId(String);
struct OrderId(String);
struct Amount(i64);
struct PlatformFee(i64);
struct TxFee(i64);
struct NetAmount(i64);

struct OrderPayoutParams {
    shop_id: ShopId,
    customer_id: CustomerId,
    order_id: OrderId,
    amount: Amount,
    platform_fee: PlatformFee,
    tx_fee: TxFee,
    net_amount: NetAmount,
}

fn process_order_payout(params: OrderPayoutParams) {
    // ...
}
```

Now try to swap them:

```rust
let params = OrderPayoutParams {
    shop_id: customer_id,       // ERROR: expected `ShopId`, found `CustomerId`
    customer_id: shop_id,       // ERROR: expected `CustomerId`, found `ShopId`
    order_id: order_id,
    amount: net_amount,         // ERROR: expected `Amount`, found `NetAmount`
    platform_fee: tx_fee,       // ERROR: expected `PlatformFee`, found `TxFee`
    tx_fee: platform_fee,       // ERROR: expected `TxFee`, found `PlatformFee`
    net_amount: amount,         // ERROR: expected `NetAmount`, found `Amount`
};
```

The compiler refuses. Not because the data is shaped wrong, but because the *meaning* is wrong. `CustomerId` is not `ShopId`, even though both wrap a `String` underneath. `NetAmount` is not `Amount`, even though both wrap an `i64`.

**Go:**
```go
type ShopID string
type CustomerID string
type OrderID string
type Amount int64
type PlatformFee int64
type TxFee int64
type NetAmount int64

type OrderPayoutParams struct {
    ShopID      ShopID
    CustomerID  CustomerID
    OrderID     OrderID
    Amount      Amount
    PlatformFee PlatformFee
    TxFee       TxFee
    NetAmount   NetAmount
}

func ProcessOrderPayout(params OrderPayoutParams) error {
    // ...
}
```

Go's type definitions are not aliases. `ShopID` and `CustomerID` are distinct types. Passing one where the other is expected is a compile-time error. Same for `Amount` vs `NetAmount` vs `PlatformFee`.

**TypeScript:**
```ts
type ShopId = string & { readonly __brand: "ShopId" };
type CustomerId = string & { readonly __brand: "CustomerId" };
type OrderId = string & { readonly __brand: "OrderId" };
type Amount = number & { readonly __brand: "Amount" };
type PlatformFee = number & { readonly __brand: "PlatformFee" };
type TxFee = number & { readonly __brand: "TxFee" };
type NetAmount = number & { readonly __brand: "NetAmount" };

interface OrderPayoutParams {
  shopId: ShopId;
  customerId: CustomerId;
  orderId: OrderId;
  amount: Amount;
  platformFee: PlatformFee;
  txFee: TxFee;
  netAmount: NetAmount;
}

function processOrderPayout(params: OrderPayoutParams) {
  // ...
}
```

TypeScript doesn't have native newtype wrappers, so we use branded types. It's a well-established pattern that adds a phantom property to prevent accidental interchange. A small amount of ceremony that pays for itself immediately.

## Living With Newtypes

The first thing people ask when they see this is "okay but now I can't do anything with my data." That's fair. A `ShopId(String)` doesn't have `.len()` or `.contains()` or any of the methods you're used to calling on `String`. You'd have to write `shop_id.0.len()` everywhere, and that's ugly.

This is where `Deref` comes in. In Rust, you can implement `Deref` to let your newtype transparently expose the inner type's methods:

```rust
use std::ops::Deref;

struct ShopId(String);

impl Deref for ShopId {
    type Target = String;

    fn deref(&self) -> &String {
        &self.0
    }
}

let shop = ShopId("shop_abc123".to_string());
println!("{}", shop.len());        // works, delegates to String::len()
println!("{}", shop.to_uppercase()); // works too
```

You get full access to all `String` methods without unwrapping. But the type system still prevents you from passing a `ShopId` where a `CustomerId` is expected. Best of both worlds.

You'll also want `Display` and `From` so your types play nicely with the rest of your code:

```rust
use std::fmt;

impl fmt::Display for ShopId {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}", self.0)
    }
}

impl From<String> for ShopId {
    fn from(s: String) -> Self {
        ShopId(s)
    }
}

// Now you can do:
let shop: ShopId = "shop_abc123".to_string().into();
println!("Processing payout for {shop}");
```

And here's where it gets really powerful. You can add validation directly in the constructor, so invalid data can never become a `ShopId` in the first place:

```rust
impl ShopId {
    pub fn new(id: String) -> Result<Self, String> {
        if id.is_empty() {
            return Err("Shop ID cannot be empty".into());
        }
        if !id.starts_with("shop_") {
            return Err("Shop ID must start with 'shop_'".into());
        }
        Ok(ShopId(id))
    }
}
```

Once a `ShopId` exists in your system, you *know* it's valid. Every function that receives a `ShopId` can skip validation entirely. The constructor already did the work.

In Go, defined types start with an empty method set, but built-in operations like `len()` still work and you can add your own methods:

```go
type ShopID string

id := ShopID("shop_abc123")
fmt.Println(len(id)) // works, len() is a built-in function

// Add your own methods
func (id ShopID) Validate() error {
    if id == "" {
        return errors.New("shop ID cannot be empty")
    }
    return nil
}
```

In TypeScript, branded types are just structural, so all `string` or `number` operations work without any extra code. The brand only exists at compile time.

## What You Actually Gain

This isn't just about catching swapped arguments. It changes how you think about your code.

**Self-documenting code.** When a function takes `ShopId` instead of `String`, you don't need a doc comment explaining what that parameter is. The type *is* the documentation.

**Refactoring confidence.** When you rename a field or change a data flow, the compiler traces every usage of that type across your entire codebase. Nothing slips through.

**Validation at the boundary.** When you construct a `ShopId`, you can enforce invariants: must be a valid ObjectID format, can't be empty, must exist in the database. Every `ShopId` in your system is guaranteed valid. Not because every function checks, but because the constructor checked once and the type system carries that guarantee forward.

**Grep-ability.** Searching for `ShopId` in your codebase shows you every place a shop identifier is created, passed, stored, or transformed. Searching for `String` shows you everything.

**Security.** A `RawUserInput` type that must be explicitly converted to `SanitizedHtml` before rendering? That's injection prevention enforced by the compiler, not by code review discipline.

## The Cost Is Lower Than You Think

The most common objection is ceremony. "I don't want to wrap every string in a newtype." But think about the alternative: you're trusting that every developer on your team, across every PR, in every late-night hotfix, will correctly match unnamed strings to their intended purpose. That's not engineering. That's hope.

The wrapper types are typically two to five lines each. You write them once. The compiler enforces them forever.

---

Scalar types describe what data *looks like*. A sequence of characters, a 64-bit integer, a boolean flag. Domain types describe what data *means*. A title, a price in USD, a sanitized HTML fragment, a user ID.

The gap between these two is where bugs live. I learned that the hard way. Wrap your primitives. Make your types mean something. Let the compiler do the work that code review and testing never will.

—Samuel

---

*Edits:*
- *2026-04-15: Corrected Go section on defined types and method sets based on readers feedback.*
