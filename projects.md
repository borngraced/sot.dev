---
layout: page
title: Projects
---

## What I've built

* **[Building Khoomi](/building-khoomi.html)**: an African marketplace for handmade goods. I built the backend from scratch in Go, covering multi-vendor orders, inventory, shipping, seller wallets, payouts, notifications, and moderation. More on the architecture: [orders](/khoomi-multi-vendor-orders.html), [shops](/khoomi-shop-architecture.html), and [listings](/khoomi-listing-architecture.html).
* **[DDW](https://www.dontdiewondering.com)**: a magazine platform built with Next.js and TypeScript, with memberships, gated content, and CMS-backed publishing.
* **[awari](https://github.com/borngraced/awari)**: a Wayland app launcher built on GPUI, Zed's UI framework. I profiled its startup and memory use and brought idle memory down from ~141 MiB to ~39 MiB. [I wrote about the work here](/cutting-a-rust-gpui-launcher-idle-memory.html).

## Low-level libraries

* **[packed_bits](https://github.com/borngraced/packed_bits)**: a `no_std`, const-friendly bit-packing library for Rust.
* **[paged-small-vec](https://github.com/borngraced/paged-small-vec)**: an experiment with an inode-style container using inline storage and fixed-size heap chunks. It mostly lost to `Vec` and `SmallVec`; [I wrote about why](/inode-style-vector-in-rust.html).

## Open source

* [**REVM**](https://github.com/bluealloy/revm): improved JumpTable bit lookups by 7.9x ([#2618](https://github.com/bluealloy/revm/pull/2618)) and improved benchmarks by 8.65% by avoiding unnecessary warm-address cloning ([#2634](https://github.com/bluealloy/revm/pull/2634)).
* [**Rust Clippy**](https://github.com/rust-lang/rust-clippy): added the [`needless_type_cast`](https://github.com/rust-lang/rust-clippy/pull/16139) lint.
* [**Slint**](https://github.com/slint-ui/slint): changed WGPU instance creation so unused GL/Mesa backends aren't loaded at startup ([#13261](https://github.com/slint-ui/slint/pull/13261)).
* [**WalletConnect Rust**](https://github.com/GLEECBTC/WalletConnectRust): implemented the [Core Pairing API](https://github.com/GLEECBTC/WalletConnectRust/pull/3) and [WASM WebSocket support](https://github.com/GLEECBTC/WalletConnectRust/pull/1) for Komodo's integration.
* [**librustzcash**](https://github.com/GLEECBTC/librustzcash): made Komodo's wallet backend async and WASM-compatible ([#8](https://github.com/GLEECBTC/librustzcash/pull/8)), and fixed unconfirmed change-note tracking for shielded balances ([#10](https://github.com/GLEECBTC/librustzcash/pull/10)).

## Experiments

* **[lc3-vm-rust](https://github.com/borngraced/lc3-vm-rust)**: an LC-3 virtual machine written in Rust.
* **[rv32i-vm-rust](https://github.com/borngraced/rv32i-vm-rust)**: an educational RV32I virtual machine in Rust.
* **[DoomHelix](https://github.com/borngraced/doom-helix)**: my Helix fork with a native ACP agent panel for explaining, fixing, and refactoring code inside the editor.
