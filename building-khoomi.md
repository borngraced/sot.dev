---
layout: page
title: Building Khoomi
---

## Why Khoomi?

Nigeria is home to a thriving community of artisans and small business owners who create unique products that reflect the country's rich cultural heritage: leather goods, textiles, jewelry, art, handcrafted furniture, fashion pieces, and crafts passed down through generations. But most of these creators remain invisible beyond their local markets.

The problem isn't talent. It's access.

Many of these businesses struggle to reach a wider audience beyond their immediate communities: limited access to technology, no user-friendly platforms to list their products, and no resources to market effectively. This doesn't just limit their ability to generate income but also restricts the wider public's access to their unique products.

Meanwhile, the platforms that do exist aren't built for this market. Global marketplaces don't serve Nigeria well: wrong payment rails, no local shipping infrastructure, international fees that eat into already thin margins, and an experience designed for Western sellers and buyers. Nigerian creators deserve better than being an afterthought on someone else's platform.

## What I'm Building

Khoomi is a marketplace built from the ground up for Nigerian artisans and creatives. Not a clone of existing platforms with Naira slapped on top, but actual infrastructure designed for how this market works.

That means:

- Local payment processing that handles bank transfers, cards, and mobile money; the ways Nigerians actually pay
- Integrated shipping across all 36 states plus FCT, working with carriers like GIG to handle waybill generation, tracking, and the complexity of last-mile delivery in Nigeria
- A wallet system with proper ledger-based transactions, so sellers can track their earnings, pending payments, and payouts with full transparency
- Discovery tools that help buyers find incredible work that's been hidden in local markets; search, categories, and eventually personalized recommendations

The platform allows artisans and small business owners to create their own shops, showcase their products with descriptions and images, set their own pricing and shipping options, and manage orders. All through a simple interface that doesn't require technical expertise.

## Why This Matters

I believe that everyone should have an equal opportunity to showcase their talents and reach a wider audience. The goal isn't just to build another e-commerce platform, it's to drive economic growth by supporting local businesses and creating opportunities for individuals who've been locked out of the digital economy.

By connecting talented artisans and small business owners with customers across Nigeria, Khoomi can help promote local entrepreneurship while giving buyers access to products they simply can't find anywhere else. The long-term vision extends beyond Nigeria, once we've built something that works here, there's an opportunity to expand across Africa, connecting creative economies that face similar challenges.

## Why Me

I've spent years building complex systems, blockchain infrastructure, performance-critical code, the kind of engineering where getting the details wrong means real consequences. Building a marketplace that handles people's money and livelihoods requires that same rigor.

Khoomi's backend is built in Go with a clear architecture separating business logic from infrastructure. The wallet service uses a two-balance model (pending and available) with ledger-based transactions for complete audit trails. Payment processing integrates with Paystack for both buyer payments and seller payouts. Shipping uses a semi-integrated approach where Khoomi handles waybill generation and tracking while sellers manage physical handoffs, a pragmatic solution for Nigeria's logistics reality.

This isn't a weekend project or a pitch deck waiting for someone else to build it. It's real infrastructure, being built piece by piece, designed to scale.

## The Opportunity

Nigeria's e-commerce market is growing rapidly, but the creative economy remains massively underserved. There's no dedicated platform doing for Nigerian artisans what the best marketplaces have done elsewhere, providing not just a listing page, but the full stack of tools needed to run a real business online.

Khoomi is my bet on what happens when you give talented people the right tools. The cultural heritage is already there. The creativity is already there. The demand is already there. What's missing is the infrastructure to connect it all.

That's what I'm building.
