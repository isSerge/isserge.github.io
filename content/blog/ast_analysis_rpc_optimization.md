+++
title = "An Unexpected Fit: AST Analysis as an RPC Optimization technique"
date = 2026-06-04
description = "How static analysis of Rhai filter scripts reduced RPC log fetching by 12–97% in an on-chain event monitoring engine."
[taxonomies]
tags = ["rust", "rhai", "blockchain", "open-source", "ethereum", "ast"]
+++

Every business eventually needs to track what's happening in its systems — log ingestion, event pipelines, alerting. If the business runs on a blockchain, that problem is more nuanced: you're listening to a global, append-only ledger potentially shared with millions of other participants, and the signal-to-noise ratio can be brutal.

One way to solve this is to reach for a third-party solution — Tenderly, Alchemy, and other similar services. But for infrastructure that's critical to your business, there are real reasons to build in-house: cost at scale, full control over the data pipeline, no dependency on external uptime, and the ability to tailor filtering logic exactly to your needs. It's also the kind of problem that turns out to be surprisingly interesting to build.

This post walks through how I built an on-chain event monitoring engine and how adding AST analysis to the filtering layer turned into an unexpected optimization that reduced RPC requests by 12–97% (depending on how selective the filter is).

## Basic Implementation

The foundation is straightforward: connect to an Ethereum node via RPC, subscribe to new blocks, and for each block fetch the logs you care about.

```rust
use alloy::{
    providers::{Provider, ProviderBuilder, WsConnect},
    rpc::types::{BlockNumberOrTag, Filter},
};

// Connect to RPC
let ws = WsConnect::new("wss://..."); // RPC provider URL
let provider = ProviderBuilder::new().connect_ws(ws).await?;

// Subscribe to logs (filter may include specific address, event signatures, etc.)
let filter = Filter::new().from_block(BlockNumberOrTag::Latest);
let mut stream = provider.subscribe_logs(&filter).await?.into_stream();

while let Some(log) = stream.next().await {
    // Decode log against ABI
    let decoded = decode_log(&log)?;

    // Filter by event name, contract address, parameters...
    if !is_relevant(&decoded) { continue; }

    // Process log: store in DB, send notification, trigger alert, call external API...
} 
```

This works, but it has two fundamental problems.

**Tight coupling**: filter logic lives in code. Adding a new monitor, tweaking a condition, 
or removing a filter means editing the application and redeploying.

**Overfetching**: every log from every block crosses the wire regardless of relevance. 
On a busy chain like Ethereum mainnet, a single block can contain tens of thousands of 
log entries — most of which you don't care about, and all of which cost RPC credits.

## Filtering Engine

Let's address the first problem — tight coupling between filter logic and code.

The core idea is to let users express filter logic as scripts. One approach is to build a custom DSL using a parser combinator like [winnow](https://docs.rs/winnow) — you get full control over syntax and evaluation, but you're also responsible for building and maintaining the language itself.

An alternative is to use an embedded scripting language such as [Rhai](https://rhai.rs/). It's sandboxed, fast, and has a familiar syntax. More importantly, it gives filter authors access to real programming constructs: variables, functions, conditionals, arithmetic — without having to implement any of that.

A simple filter script may look like this:
```javascript
log.name == "Transfer"
&& log.params.to == "0xAddress"
```

> Note: Rhai does not support Big Numbers by default — if you need to compare large values like `log.params.value > 1000`, see [this post](https://isserge.github.io/blog/rhai-bigint-and-evm-plugins/) for how to add BigInt and EVM-specific support.

Incoming logs are evaluated against the script at runtime — matches are sent downstream, the rest are discarded.

Filtering logic is now decoupled — it can be shared, swapped, or versioned independently, you may even load it during the runtime without restarting the application.

## Reducing RPC Overhead with AST Analysis

Now for the second problem — overfetching.

At this point the engine evaluates every incoming log against the script. But the logs still have to cross the wire first. On a busy chain, that's a lot of data to fetch and discard.

The insight is that the filter script already encodes exactly what we care about. Instead of evaluating it at runtime against every log, we can analyze the script's AST statically — before any logs are fetched — and extract that information upfront.

Concretely: walk the AST, find all comparisons against `log.name`, and collect the referenced event names. Ethereum's `eth_getLogs` RPC call supports filtering by `topic0` — the hash of the event signature. Instead of fetching everything and discarding irrelevant logs after, we can pass the exact signatures we care about directly in the request.

```rust
// Statically analyze the filter script AST to extract referenced event names
// e.g. `log.name == "Transfer"` → ["Transfer"]
let extracted_event_names = analyze_ast(&script)?;

// Build topic0 selectors by cross-referencing names with the ABI
let topics: Vec<B256> = extracted_event_names
    .iter()
    .filter_map(|name| abi.event_signature(name)) 
    .map(|sig| keccak256(sig.as_bytes()))
    .collect();

// Only logs matching these selectors cross the wire
let filter = Filter::new()
    .event_signature(topics)
    .from_block(BlockNumberOrTag::Latest);
```

This is the same principle behind compiler dead code elimination and database query planning: shift the filtering as early in the pipeline as possible, so work that was never needed never happens.

Even with a precise `topic0` filter, some blocks won't contain any relevant events at all. Ethereum block headers include a bloom filter — a compact probabilistic structure that encodes which topics and addresses appear in a block's logs. Instead of blindly firing `eth_getLogs` for every new block, it is possible to fetch the block header first and checks the bloom filter client-side against our AST-extracted topics. If the bloom filter guarantees our event isn't there, it is safe to skip the `eth_getLogs` RPC call entirely.

## Measuring the Impact

To quantify the impact, I ran two scenarios against 100 blocks on Ethereum Mainnet (blocks 23,545,500–23,545,600), covering tens of thousands of log entries:

- Broad filter: ERC-20 Transfer events from all contracts — covers the majority of on-chain activity in any given block
- Specific filter: WETH Deposit events from a single contract — a narrow, targeted monitor looking for one rare event type

### Broad filter: All ERC-20 Transfers

The monitor script is:
```javascript
log.name == "Transfer"
```

Without AST analysis, the RPC request includes all selectors from the ERC-20 ABI — fetching both `Transfer` and `Approval` logs. With AST analysis, the engine detects the `log.name == "Transfer"` comparison and narrows the request to the `Transfer` selector only.

The difference: 34,698 logs fetched without AST vs 30,827 with — **3,871 fewer log objects, a 12.6% reduction**.

### Specific filter: WETH Deposits

The monitor script is:
```javascript
log.name == "Deposit"
```

targeting the WETH contract (`0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2`), which has a 4-event ABI: `Transfer`, `Approval`, `Deposit`, and `Withdrawal`. Without AST analysis, all four selectors are included in the RPC request. With AST analysis, only the `Deposit` selector is requested.
The difference: 37,150 logs fetched without AST vs 1,078 with — **36,072 fewer log objects, a 97.1% reduction**.

The difference between the two scenarios becomes clear visually:

![AST optimization chart](/ast_chart.svg)

## Conclusion

The optimization came out of a specific pain point: eagerly fetching all block events to evaluate against every monitor script was an `O(N × M)` problem — `N` monitors, `M` events per block, and expensive RPC calls on every iteration. The AST already contained exactly what each script needed, making it possible to fetch only what was relevant before any evaluation happened.

The impact scales with the gap between ABI breadth and filter selectivity — 12.6% reduction for a 2-event ABI with a single-event filter, 97.1% for a 4-event ABI with the same. The broader the ABI, the more there is to leave on the table without static analysis. 

The same AST walk can also do schema validation, ensuring scripts only access fields that exist in the ABI and catching errors before any evaluation.

The AST analyzer ended up becoming its own crate — [rhai-analyzer](https://github.com/isSerge/rhai-analyzer). If you're working with Rhai and need to inspect scripts without executing them, it might be worth a look.
