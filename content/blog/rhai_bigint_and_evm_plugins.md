+++
title = "Embedding Lossless BigInt Arithmetic into Rhai"
date = 2026-05-13
description = "How I built two open-source Rhai plugins to solve IEEE 754 precision loss in a blockchain stream processor."
[taxonomies]
tags = ["rust", "rhai", "blockchain", "open-source"]
+++

I was building a blockchain stream processor in Rust — a tool that watches on-chain events and lets users define their own alerting rules. The goal was for a user to write something like this:

```js
tx.value > 1500000000000000000 && tx.gas_price < 50000000000 && tx.to == "0xd8dA6BF..."
```

and have the engine evaluate it against a live stream of transactions. But how do you actually run user-supplied expressions safely in Rust? One option was to implement AST parsing using the `winnow` crate (which I'd already done before). But this time, I wanted something more flexible — so I decided to try an embedded scripting language.

## Why Rhai

After checking out a few options, I landed on [Rhai](https://rhai.rs/). Here's why it stood out:

- Easy syntax
- Native to Rust — integrating with Rust types is a breeze
- Sandboxed by default
- Highly customizable — you can restrict resources or even disable things like loops and functions
- Tiny binary footprint

Rhai looked perfect for what I wanted. But there was a problem with numbers.

## The IEEE 754 problem

Blockchains require large numbers.

On Ethereum, 1 ETH is represented as 10¹⁸ Wei. This is deliberate: the EVM does not support floating-point numbers, so all token amounts are stored as integers with an implicit decimal offset. A transfer of 1.5 ETH is actually the integer `1500000000000000000`.

Rhai's default integer type is `i64`, which has a maximum value of about 9.2 × 10¹⁸. A balance of 10,000 ETH (`10_000_000_000_000_000_000_000`) already exceeds `i64::MAX`.

One option is to cast these values to `f64`. The problem is that IEEE 754 double-precision floats have only 53 bits of mantissa, so integers beyond roughly 9 × 10¹⁵ cannot be represented exactly:

```js
// Rhai FLOAT is f64 under the hood
let a = 9999999999999999.0;
let b = 9999999999999998.0;
a == b  // true — silent precision loss makes them indistinguishable
```

The other option is to keep all values as strings in Rhai and use custom comparison helpers from the host:

```js
tx.value == "1500000000000000000"
```

This approach is awkward. Every comparison becomes a string comparison, so you lose the ability to use normal arithmetic or operators. Even basic tasks like checking if a value is greater than a threshold require extra helpers. The code becomes harder to read and maintain, and it is easy to make mistakes.

This is a [known limitation of Rhai](https://github.com/rhaiscript/rhai/issues/587). I decided to build a more robust solution.

## Fixing the math: `rhai-bigint`

The first step was to solve the precision problem.

Rhai's plugin system allows you to register custom Rust types and overload their operators. I wrote a package that injects `num_bigint::BigInt` directly into the engine and wires up all the standard operators, so you can work with big integers natively in scripts.

```rust
use rhai::{Engine, packages::Package};
use rhai_bigint::BigIntPackage;

let mut engine = Engine::new();
BigIntPackage::new().register_into_engine(&mut engine);
```

The package supports:
- Arithmetic: `+`, `-`, `*`, `/`, `%`, `**` (power), unary `-`
- Comparisons: `==`, `!=`, `<`, `<=`, `>`, `>=`
- Bitwise: `&`, `|`, `^`, `<<`, `>>`

You can construct a `BigInt` from integers, floats (truncated toward zero), or strings:

```js
let balance   = bigint("10000000000000000000000");
let threshold = bigint("1500000000000000000");

balance > threshold  // true, no precision loss
```

All operations use `num_bigint` under the hood — there is no precision loss, no matter how large the numbers get. For the full API see the [GitHub repository](https://github.com/isSerge/rhai-bigint).

## Fixing the ergonomics: `rhai-evm`

Solving precision was only half the problem. Even with `rhai-bigint` in place, a user writing an ETH threshold would still have to write `bigint("1500000000000000000")` — a 19-digit string that is easy to miscount and impossible to read at a glance. The decimal offset is an implementation detail of the EVM; there is no reason to expose it in a scripting layer.

`rhai-evm` solves this by letting users think in human units. It registers denomination helpers that accept integers, floats, or strings and scale by the appropriate power of ten, returning a `BigInt`:

```js
let price  = ether(1.5);       // 1500000000000000000
let gas    = gwei(30);         // 30000000000
let fee    = usdc(500);        // 500000000
```

When none of the named helpers fit — say, a token with 8 decimals — there is a generic `decimals(value, n)` function that scales by 10ⁿ:

```js
let custom = decimals(1.5, 8); // 150000000
```

Beyond denominations, the package also exposes `keccak256` for hashing and address utilities for the two most common EVM validation needs:

```js
let topic = keccak256("Transfer(address,address,uint256)");
is_address("0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045")  // true
to_checksum("0xd8da6bf26964af9d7eed9e03e53415d37aa96045") // "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045"
```

### Bridging `alloy-primitives` on the host side

Injecting data from Rust into a Rhai script requires converting your native types into values the engine understands. The naive approach is to serialize everything to strings and parse them back inside the script — but that reintroduces the same precision problem we just solved. What you want is a direct, lossless conversion from your Rust type into a `BigInt` that Rhai can work with natively.

[`alloy-primitives`](https://github.com/alloy-rs/core/tree/main/crates/primitives) has become the de facto standard for EVM types in the Rust ecosystem, so decoded transaction values most likely arrive in your host code as `U256` or `I256`. `rhai-evm` exposes `u256_to_bigint_dynamic` and `i256_to_bigint_dynamic` for injecting these directly into the script's scope without going through string serialisation:

```rust
use alloy_primitives::U256;
use rhai_evm::u256_to_bigint_dynamic;

let raw: U256 = tx.value;
scope.push("value", u256_to_bigint_dynamic(raw));
```

The conversion is byte-exact: it reads the big-endian representation of the `U256` and reconstructs a `BigInt` with the same sign and magnitude.

## Writing rules

With both crates registered into the engine, this is what user-facing rules look like in practice.
The opening expression from the beginning of this post, with raw numbers replaced:

```js
tx.value > ether(1.5) && tx.gas_price < gwei(50) && tx.to == "0xd8dA6BF..."
```

And because these are real operators on real numeric types, rules can combine transaction and event log data in the same expression:

```js
let watched  = "0xd8dA6BF...";
let min      = ether(2.0);
let max_gas  = gwei(50);

// match a large WETH deposit from a watched address
log.name == "Deposit" && log.params.dst == watched
&& tx.value > min && tx.gas_price < max_gas
```

The user writes what they mean. The precision is handled out of sight.

## Open-source release

`rhai-bigint` and `rhai-evm` started as internal pieces of [Argus](https://github.com/isSerge/argus-rs), but the problems they solve are not unique to it. `rhai-bigint` applies anywhere large integers appear — finance, games, cryptography. `rhai-evm` is more focused, but if you are embedding Rhai in an EVM project, it handles the boilerplate you would otherwise write yourself.

Both are MIT licensed and available on crates.io and GitHub:

- [rhai-bigint on crates.io](https://crates.io/crates/rhai-bigint)
- [rhai-evm on crates.io](https://crates.io/crates/rhai-evm)
- [rhai-bigint on GitHub](https://github.com/isSerge/rhai-bigint)
- [rhai-evm on GitHub](https://github.com/isSerge/rhai-evm)
