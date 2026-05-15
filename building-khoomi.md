---
layout: post
title: Building Khoomi
date: 2026-01-07
---

## Why Khoomi?

Khoomi started from a simple frustration: there are too many good makers in Nigeria who are still selling like the internet barely exists.

Crochet artists. Leather workers. Jewelers. Furniture makers. Skincare makers. Fashion designers. People making genuinely good things, but mostly selling through WhatsApp statuses, Instagram DMs, referrals, or physical markets. That works to a point, but it breaks down fast.

A buyer wants to browse, compare, save items, message the seller, pay safely, track an order, and come back later. A seller wants to list products, manage stock, take orders, set shipping, get paid, and not spend the whole day answering the same questions manually.

The tools around them are either too informal or not built for this market. Global marketplaces are not thinking about Nigerian payment habits, local delivery realities, seller trust, small inventory, custom handmade products, or the fact that a lot of sellers are starting from their phone.

So I started building Khoomi.

## What I'm Building

Khoomi is a marketplace for handmade, creative, and small-batch products in Nigeria. The goal is not "Etsy but with naira" or a generic e-commerce clone. The goal is to build the boring infrastructure that lets local sellers run a real online shop without needing to become engineers, logistics operators, payment experts, and customer support teams all at once.

The buyer side is straightforward:

- discover products by category, search, shops, and recommendations
- view listings with images, video, variations, personalization, reviews, and shipping details
- add to cart, buy now, save to wishlist, message sellers, and place orders
- track orders from payment through fulfillment

The seller side is where most of the work is:

- shop creation and onboarding
- listing editor with media, categories, inventory, variations, personalization, and shipping profiles
- order management, shipping updates, cancellations, and receipts
- wallet balances for pending and available earnings
- payout/bank setup
- discounts, analytics, reviews, notifications, and messaging

That sounds like a lot because it is a lot. Marketplaces are not landing pages with checkout buttons. Every small feature touches money, stock, trust, fulfillment, or support.

## The Hard Parts

The most annoying parts are also the most important parts.

Payments have to work the way Nigerians actually pay. Orders can contain products from multiple shops, so one checkout can become multiple seller fulfillment flows. A seller should not receive withdrawable earnings before an order is actually completed, so the wallet needs pending and available balances, not just one number on a screen.

Shipping has to be practical. Nigeria does not have the same logistics assumptions as the US or Europe. Sellers need shipping profiles, delivery estimates, handling fees, and a workflow that accepts the messy parts of local delivery instead of pretending they do not exist.

Listings also get complicated quickly. A handmade bag can have sizes, colors, personalization, limited quantity, video, different prices per variation, and shipping constraints. A product model that works for books will not automatically work for crochet jewelry, skincare, furniture, and digital art.

This is why a lot of Khoomi is backend and product plumbing: carts, checkout, stock checks, order timelines, wallet ledgers, notification routing, messaging, seller dashboards, media uploads, category schemas, and admin moderation.

The visible UI is only the top layer.

## Why I'm Writing About It

I've been building Khoomi in public because the interesting part is not just "I built a marketplace." The interesting part is all the decisions underneath it.

Why embed listing variations instead of making a separate collection? When should stock be reserved? How should a multi-vendor order be represented? What should be in a wallet ledger? What do you cache? What do you never cache? What should be native on mobile and what can wait?

Those decisions are the kind of thing you only really learn by building the system and being forced to live with the trade-offs.

I have started writing the technical notes here:

- [Week 1: Listing Architecture Decisions](/building-khoomi-week-1.html)
- [Week 2: Shop Architecture](/building-khoomi-week-2.html)
- [Week 3: Multi-Vendor Order Architecture](/building-khoomi-week-3.html)

## Where It Is Now

Khoomi is still pre-launch. The platform is being tested, the iOS app is being built alongside the web app, and early sellers are being onboarded.

There is still a lot to tighten: seller onboarding, mobile polish, shipping flows, account settings, order management, and the million small details that decide whether a seller trusts the platform enough to use it every day.

But the direction is clear.

Nigeria does not lack creativity. It lacks better infrastructure around that creativity.

Khoomi is my attempt at building that infrastructure properly.
