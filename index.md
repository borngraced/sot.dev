---
layout: home
---

Hey! I'm Sami; I write Rust, Go, and sometimes C for fun. I like building tools and understanding how things work at a low level.

I worked across fullstack web development before moving into systems work at Komodo, where I worked on DeFi backends in Rust and Go. These days I'm contributing to [rust-clippy](https://github.com/rust-lang/rust-clippy/pulls?q=author%3Aborngraced+) and building [Khoomi](/building-khoomi).

## Notable Contributions

**Zcash/ARRR WASM Implementation: Komodo DeFi Framework**: Ported the Zcash protocol (ARRR coin) to WASM for browser-based transactions. Refactored librustzcash for async, added IndexedDB wallet storage, and built a WASM-compatible gRPC transport. [View PR](https://github.com/KomodoPlatform/komodo-defi-framework/pull/1957), [librustzcash](https://github.com/KomodoPlatform/librustzcash/pull/8)

**WalletConnect v2 Integration: Komodo DeFi Framework**: Integrated WalletConnect v2 Protocol in Rust and implemented for EVM and Cosmos chains. Handles multi-session, persistent storage, and Sign & Pairing APIs. [View PR](https://github.com/KomodoPlatform/komodo-defi-framework/pull/2223), [WalletConnectRust](https://github.com/KomodoPlatform/WalletConnectRust/pull/3)

**REVM Performance Optimization**: 7.9x speedup in JumpTable lookups (9.4M → 74.6M ops/sec). [View PR](https://github.com/bluealloy/revm/pull/2618)

**Rust: needless_type_cast lint**: New lint that detects bindings cast to the same type at every usage site, suggesting the correct type at definition. [View PR](https://github.com/rust-lang/rust-clippy/pull/16139)

**LC-3 VM in Rust**: Implemented an LC-3 virtual machine in Rust while learning low-level systems concepts, instruction decoding, memory, registers, and traps. [View repo](https://github.com/borngraced/lc3-vm-rust)

**paged-small-vec in Rust**: Experimental Rust container that mixes inline storage with fixed-size heap chunks, exploring a middle ground between small-vector optimization and paged allocation. [View repo](https://github.com/borngraced/paged-small-vec)

**DoomHelix: Helix With an Agent Panel**: Forked Helix to add an ACP-powered agent panel inside the editor, with commands for chat, explain, fix, and refactor using editor context like selections, file paths, language, cursor position, and diagnostics. [Read writeup](/i-forked-helix-to-make-agentic-coding-feel-native.html), [View repo](https://github.com/borngraced/doom-helix)


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
- X: [@sotdev_](https://x.com/sotdev_)
- Location: Lagos, Nigeria
