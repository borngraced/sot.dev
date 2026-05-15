---
layout: home
---

Hey, I'm Sami. I'm a Rust/Go systems engineer focused on backend infrastructure, WebAssembly, wallet protocols, and performance. I started in full-stack web development, then moved into systems work at Komodo, where I built DeFi backends in Rust and Go.

I'm looking for my next role in Rust, backend systems, infrastructure, WebAssembly, or developer tooling.

Some of my recent work: I ported ARRR/Zcash wallet support to WASM/browser targets in Komodo DeFi Framework, including async wallet sync, IndexedDB storage, and WASM-compatible transport. I built WalletConnect v2 support in Rust for EVM and Cosmos chains. I improved [REVM JumpTable lookup](https://github.com/bluealloy/revm/pull/2618) performance by 7.9x, from 9.4M to 74.6M ops/sec. I added a [Rust Clippy lint for bindings cast to the same type at every usage site](https://github.com/rust-lang/rust-clippy/pull/16139), suggesting the right type at definition.

I also built an LC-3 virtual machine in Rust to explore instruction decoding, memory, registers, and traps. I built [`paged-small-vec`](https://github.com/borngraced/paged-small-vec), an experimental Rust container mixing inline storage with fixed-size heap chunks.


## Work

### Rust Software Engineer @ Komodo Platform
*May 2022 - 2025*

Built Rust and Go infrastructure for Komodo's DeFi backend, with a focus on browser-compatible blockchain support, wallet correctness, swap reliability, and runtime tooling.

- Delivered ARRR/Zcash browser support with WASM, IndexedDB wallet storage, async sync, and gRPC-web transport. [#1957](https://github.com/GLEECBTC/komodo-defi-framework/pull/1957), [#1996](https://github.com/GLEECBTC/komodo-defi-framework/pull/1996)
- Fixed shielded-wallet sync and accounting bugs around unconfirmed change outputs and z-note tracking. [#2276](https://github.com/GLEECBTC/komodo-defi-framework/pull/2276), [#2331](https://github.com/GLEECBTC/komodo-defi-framework/pull/2331)
- Built WalletConnect v2 support for EVM and Cosmos/Tendermint activation, withdrawals, swaps, and session persistence. [WC #3](https://github.com/KomodoPlatform/WalletConnectRust/pull/3), [#2223](https://github.com/GLEECBTC/komodo-defi-framework/pull/2223)
- Improved SPV/UTXO reliability with IndexedDB header storage, chain-reorg recovery, confirmation fixes, and swap validation. [#1644](https://github.com/GLEECBTC/komodo-defi-framework/pull/1644), [#1728](https://github.com/GLEECBTC/komodo-defi-framework/pull/1728)
- Hardened browser/runtime paths for IndexedDB cursors, WebSocket validation, wallet RPCs, HD signing, and external-wallet startup. [#2028](https://github.com/GLEECBTC/komodo-defi-framework/pull/2028), [#2411](https://github.com/GLEECBTC/komodo-defi-framework/pull/2411)
- Maintained Rust tooling and architecture, including enum derive macros, safer lazy init, rustc compatibility, and 28-44% faster measured builds. [#1502](https://github.com/GLEECBTC/komodo-defi-framework/pull/1502), [#2449](https://github.com/GLEECBTC/komodo-defi-framework/pull/2449)

### Fullstack Engineer @ DDW
*April 2020 - May 2022*

- Built full-stack magazine sites with WordPress, Next.js, and TypeScript.
- Developed membership areas with authenticated access, gated content, account flows, and CMS-backed publishing.
- Improved performance, responsive UI, content modeling, and frontend/backend handoff for editorial teams.

## Contact

- Email: [0@sot.dev](mailto:0@sot.dev)
- GitHub: [@borngraced](https://github.com/borngraced)
- X: [@sotdev_](https://x.com/sotdev_)
- Location: Lagos, Nigeria
