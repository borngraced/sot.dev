---
layout: home
---

Hey! I'm Sami; I write Rust, Go, and sometimes C for fun. I like building tools and understanding how things work at a low level.

Started out building digital publications at DDW, then transitioned to systems programming at Komodo working on DeFi backends. Now I'm contributing to rust-clippy and building [Khoomi](/building-khoomi), an African marketplace for handmade goods.

## Notable Contributions

### Zcash/ARRR WASM Implementation - Komodo DeFi Framework
Ported the Zcash protocol (ARRR coin) to WASM for browser-based transactions. Refactored librustzcash for async, added IndexedDB wallet storage, and built a WASM-compatible gRPC transport. [View PR](https://github.com/KomodoPlatform/komodo-defi-framework/pull/1957) | [librustzcash](https://github.com/KomodoPlatform/librustzcash/pull/8)

### WalletConnect v2 Integration - Komodo DeFi Framework
Integrated WalletConnect v2 Protocol in Rust and implemented for EVM and Cosmos chains. Handles multi-session, persistent storage, and Sign & Pairing APIs. [View PR](https://github.com/KomodoPlatform/komodo-defi-framework/pull/2223) | [WalletConnectRust](https://github.com/KomodoPlatform/WalletConnectRust/pull/3)

### REVM - Performance Optimization
7.9x speedup in JumpTable lookups (9.4M → 74.6M ops/sec). [View PR](https://github.com/bluealloy/revm/pull/2618)

### Rust - needless_type_cast lint
New lint that detects bindings cast to the same type at every usage site, suggesting the correct type at definition. [View PR](https://github.com/rust-lang/rust-clippy/pull/16139)


## Professional Experience

### Rust Software Engineer @ Komodo Platform
*May 2022 - 2025*

Worked on Komodo's DeFi core backend, building blockchain applications in Rust and Golang:
- Integrated the Z/Pirate Coin protocol for web-based trading using WebAssembly
- Integrated WalletConnect protocol into Komodefi
- Enhanced Bitcoin Light Client (SPV) with critical fixes and improvements
- Developed EnumFromStringify, an internal Rust macro tool for improved enum and error handling
- Contributed to various blockchain protocol implementations (UTXO, ETH, etc.)

### Fullstack Engineer @ DDW
*April 2020 - May 2022*

- Built magazine websites with WordPress, NextJS & TypeScript
- Developed membership platform for UK/US users
- CSSDA UI/UX Award (June 2020)

### Some Open Source

- **rustlibzcash** - Zcash protocol in Rust
- **WalletConnectRust** - WalletConnect protocol in Rust
- **Rust** - cargo fmt, rust-clippy
- **rustaceanvim** - Neovim Rust plugin
- **eza** - modern ls replacement
- **MongoDB** - BSON Rust implementation
- **Penumbra** - Cosmos privacy blockchain
- **Khoomi** - Golang e-commerce API

## When I'm Not Coding

**Games:** CoD, Genshin Impact, FIFA

**Music:** Pop Country, CCM, nightcore, Afrobeats, Pop — Sasha Sloan, Amanda Cook, Tauren Wells, Luke Combs, Morgan Wallen, Magixx, Llona, Zoe Wees, Griff.

## Contact

- Email: [0@sot.dev](mailto:0@sot.dev)
- GitHub: [@borngraced](https://github.com/borngraced)
- Location: Lagos, Nigeria
