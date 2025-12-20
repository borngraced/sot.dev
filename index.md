---
layout: home
---

I'm a self-taught systems engineer focused on blockchain infrastructure and low-level performance work. I write mostly Rust, Golang, and C, but I'm comfortable across the stack.

Most recently at Komodo Platform, where I built DeFi backend systems and cross-chain trading infrastructure.

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

## Notable Contributions

### Zcash/ARRR WASM Implementation - Komodo DeFi Framework
Ported the Zcash protocol (ARRR coin) to WASM for browser-based transactions. Refactored librustzcash for async, added IndexedDB wallet storage, and built a WASM-compatible gRPC transport. [View PR](https://github.com/KomodoPlatform/komodo-defi-framework/pull/1957) | [librustzcash](https://github.com/KomodoPlatform/librustzcash/pull/8)

### WalletConnect v2 Integration - Komodo DeFi Framework
Integrated WalletConnect v2 for EVM and Cosmos chains. Handles multi-session, persistent storage, and Sign & Pairing APIs. [View PR](https://github.com/KomodoPlatform/komodo-defi-framework/pull/2223) | [WalletConnectRust](https://github.com/KomodoPlatform/WalletConnectRust/pull/3)

### REVM - Performance Optimization
7.9x speedup in JumpTable lookups (9.4M → 74.6M ops/sec). [View PR](https://github.com/bluealloy/revm/pull/2618)

### Rust - needless_type_cast lint
New lint that detects bindings cast to the same type at every usage site, suggesting the correct type at definition. [View PR](https://github.com/rust-lang/rust-clippy/pull/16139)

### Open Source

- **rustlibzcash** - Zcash protocol in Rust
- **WalletConnectRust** - WalletConnect protocol in Rust
- **Rust** - cargo fmt, rust-clippy
- **rustaceanvim** - Neovim Rust plugin
- **eza** - modern ls replacement
- **MongoDB** - BSON Rust implementation
- **Penumbra** - Cosmos privacy blockchain
- **Khoomi** - Golang e-commerce API

## Contact

- Email: [0@sot.dev](mailto:0@sot.dev)
- GitHub: [@borngraced](https://github.com/borngraced)
- Location: Lagos, Nigeria
