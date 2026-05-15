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

- Delivered browser support for ARRR/Zcash, including WASM compilation, async wallet sync, IndexedDB-backed block and wallet storage, transaction history, balance streaming, and gRPC-web transport. [#1957](https://github.com/GLEECBTC/komodo-defi-framework/pull/1957), [#1996](https://github.com/GLEECBTC/komodo-defi-framework/pull/1996)
- Fixed shielded-wallet correctness issues around resumable sync, activation status, unconfirmed change outputs, and unconfirmed z-note tracking. [#2276](https://github.com/GLEECBTC/komodo-defi-framework/pull/2276), [#2331](https://github.com/GLEECBTC/komodo-defi-framework/pull/2331)
- Built WalletConnect v2 support for EVM and Cosmos/Tendermint flows, including Rust Sign and Pairing APIs, WASM/native relay transport, multi-session persistence, coin activation, withdrawals, and swap signing. [WC #3](https://github.com/KomodoPlatform/WalletConnectRust/pull/3), [#2223](https://github.com/GLEECBTC/komodo-defi-framework/pull/2223)
- Improved SPV/UTXO and swap reliability: block-header storage limits, IndexedDB SPV storage for WASM, chain-reorg recovery, confirmation retries/timeouts, MTP RPC support, and swap parameter validation. [#1644](https://github.com/GLEECBTC/komodo-defi-framework/pull/1644), [#1728](https://github.com/GLEECBTC/komodo-defi-framework/pull/1728)
- Hardened browser/runtime infrastructure with IndexedDB cursor fixes, worker-safe IndexedDB access, WebSocket URL validation, wallet password RPCs, HD message signing, and external-wallet startup support. [#2028](https://github.com/GLEECBTC/komodo-defi-framework/pull/2028), [#2411](https://github.com/GLEECBTC/komodo-defi-framework/pull/2411)
- Maintained core Rust architecture and tooling, including a custom enum-conversion derive macro, safer lazy initialization, Rust toolchain compatibility work, and workspace dependency organization with measured 28-44% faster builds in PR timing runs. [#1502](https://github.com/GLEECBTC/komodo-defi-framework/pull/1502), [#2449](https://github.com/GLEECBTC/komodo-defi-framework/pull/2449)

### Fullstack Engineer @ DDW
*April 2020 - May 2022*

- Built full-stack magazine websites with WordPress, Next.js, and TypeScript, covering frontend implementation, backend integration, custom editorial workflows, and production delivery.
- Developed membership areas for reader audiences across the US and Europe, including authenticated access, gated content, account flows, and CMS-backed editorial publishing.
- Improved performance, responsive UI, content modeling, WordPress customization, and frontend/backend handoff for editorial teams.

## Contact

- Email: [0@sot.dev](mailto:0@sot.dev)
- GitHub: [@borngraced](https://github.com/borngraced)
- X: [@sotdev_](https://x.com/sotdev_)
- Location: Lagos, Nigeria
