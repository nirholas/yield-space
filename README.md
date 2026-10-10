# YieldSpace

<!-- three.ws:badges -->
[![GitHub stars](https://img.shields.io/github/stars/nirholas/yield-space?style=flat&logo=github)](https://github.com/nirholas/yield-space/stargazers) [![License](https://img.shields.io/github/license/nirholas/yield-space?style=flat)](https://github.com/nirholas/yield-space/blob/HEAD/LICENSE) [![Last commit](https://img.shields.io/github/last-commit/nirholas/yield-space?style=flat)](https://github.com/nirholas/yield-space/commits) [![PRs welcome](https://img.shields.io/badge/PRs-welcome-brightgreen?style=flat)](https://github.com/nirholas/yield-space/pulls) [![AI agent friendly](https://img.shields.io/badge/AI%20agents-AGENTS.md%20%2B%20llms.txt-6d5dfc?style=flat)](https://github.com/nirholas/yield-space/blob/HEAD/AGENTS.md)
<!-- /three.ws:badges -->


**An interest-rate market as a Uniswap v4 pool: the curve quotes a yield rather than a price, and converges to par at maturity on its own.**

A production Uniswap v4 hook. It holds no funds and takes no fee for itself. No owner, no pause switch, no upgrade path.

- **Site:** https://yield-space.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/YieldSpaceHook.sol`](src/hooks/YieldSpaceHook.sol)
- **Licence:** Apache-2.0

## How it works

A zero-coupon claim is worth less than the thing it pays out, by exactly the interest between now and maturity, and that discount shrinks to nothing as maturity arrives. Quote it on a constant-product curve and the curve does not know that. It prices the claim as a permanently distinct asset, so the pool's own convergence to par is something arbitrageurs have to impose on it, trade by trade, and the liquidity provider pays for every one of those trades.

YieldSpace, from the Yield protocol's 2019 paper, is the curve that knows. It holds x^(1 - t) + y^(1 - t) = k where `t` is the time to maturity as a fraction of a year. Far from maturity `t` is large, the exponent is small, and the curve is steep: a big discount, and the pool quotes a rate.

As maturity approaches `t` goes to zero, the exponent goes to one, and the invariant becomes `x + y = k`, a constant-sum pool that trades one for one. The convergence is the curve, not an arbitrage opportunity, so nobody has to be paid to enforce it. The consequence for the provider is the whole point.

On a constant-product pool, holding a maturing claim means being run over as it converges. Here the pool moves with it, so what the provider earns is fees on rate trading rather than a slow bleed to whoever noticed the calendar. `currency0` is the underlying and `currency1` is the claim on it.

The pool's implied rate is published by {impliedRate}, in ray per year, which is the number this market is actually about.

## Prior art

The curve is the Yield protocol's YieldSpace (Niemerg, Robinson, Livnev, 2019), and Notional and Pendle ship relatives of it. All of them are their own AMMs. Bringing it to v4 as a custom curve, so a fixed-rate market is an ordinary pool that every v4 router, indexer and position manager already understands, is the contribution here; the maths is deliberately theirs.

## Where it does not help

The time term is clamped at one year, so a claim maturing further out than that is priced as if it matured in a year and the pool will quote it wrong until it comes inside the window. The pool also has no idea whether the claim is honoured: it converges to par because the clock says so, and if the issuer defaults the curve keeps confidently quoting par on a worthless asset. That risk belongs to whoever chose the pair.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
// This hook needs no configuration.

poolManager.initialize(key, startingSqrtPriceX96);
```


### Parameters

This hook takes no per-pool configuration.

## What it reverts with

| Error | Meaning |
| --- | --- |
| `AlreadyInitialized()` | Hook was already initialized. |
| `AmountTooSmall()` | A deposit was too small to mint any shares, or a withdrawal too small to return anything. |
| `ERC20InsufficientAllowance(address,uint256,uint256)` | Indicates a failure with the `spender`’s `allowance`. Used in transfers. |
| `ERC20InsufficientBalance(address,uint256,uint256)` | Indicates an error related to the current `balance` of a `sender`. Used in transfers. |
| `ERC20InvalidApprover(address)` | Indicates a failure with the `approver` of a token to be approved. Used in approvals. |
| `ERC20InvalidReceiver(address)` | Indicates a failure with the token `receiver`. Used in transfers. |
| `ERC20InvalidSender(address)` | Indicates a failure with the token `sender`. Used in transfers. |
| `ERC20InvalidSpender(address)` | Indicates a failure with the `spender` to be approved. Used in approvals. |
| `ExpiredPastDeadline()` | A liquidity modification order was attempted to be executed after the deadline. |
| `InsufficientInitialLiquidity()` | The first deposit must exceed the permanently locked minimum. |
| `InsufficientReserves()` | The curve cannot fill this swap without emptying the side being bought. |
| `InvalidFee()` | The fee must be below 100%. |
| `InvalidNativePayer(address)` | The native currency was settled on behalf of a `payer` other than the contract paying it. |
| `InvalidNativeValue()` | Native currency was not sent with the correct amount. |
| `LiquidityOnlyViaHook()` | Liquidity was attempted to be added or removed via the `PoolManager` instead of the hook. |
| `MaturityInThePast()` | The maturity must be in the future when the pool is created. |
| `NoLiquidity()` | A quote was requested against an empty pool. |
| `PoolNotInitialized()` | Pool was not initialized. |
| `SafeERC20FailedOperation(address)` | An operation with an ERC-20 token failed. |
| `TooMuchSlippage()` | Principal delta of liquidity modification resulted in too much slippage. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 5 of the fourteen:

- `beforeInitialize`
- `beforeAddLiquidity`
- `beforeRemoveLiquidity`
- `beforeSwap`
- `beforeSwapReturnsDelta`

Mask: `0x2a88`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # YieldSpace
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # curve, custom-curve, fixed-rate, maturity, rates
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/yield-space
cd yield-space
forge build
forge test
```

Foundry 1.7 or newer, Solidity 0.8.26, EVM version `cancun` (Uniswap v4 requires transient storage).

## Deploy

```bash
# Dry run: mines the salt and prints the address without sending anything.
forge script script/Deploy.s.sol --rpc-url $RPC_URL

# For real.
forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify
```

Needs `PRIVATE_KEY` in the environment and a funded deployer on the target chain. See [`docs/deploying.md`](docs/deploying.md).

## Status

**Unaudited.** Built to an audited shape, on OpenZeppelin's audited hook bases, and tested against a real `PoolManager`. No third party has reviewed it. Read "where it does not help" above before putting money behind it.

Not affiliated with Uniswap Labs.

## Star History

[![Star History Chart](https://api.star-history.com/svg?repos=nirholas/yield-space&type=Date)](https://www.star-history.com/#nirholas/yield-space&Date)

<!-- three.ws:growth -->
## Support the project

If yield-space saves you time, **[star it on GitHub](https://github.com/nirholas/yield-space)**. Stars are how other developers and AI agents find the repositories worth trusting, and they cost you one click.

Know someone who would use it? [Post on X](https://twitter.com/intent/tweet?text=yield-space%3A%20An%20interest-rate%20market%20as%20a%20Uniswap%20v4%20pool%3A%20the%20curve%20quotes%20a%20yield%20rather%20than%20a%20price%2C%20and&url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fyield-space) · [Share on Bluesky](https://bsky.app/intent/compose?text=yield-space%3A%20An%20interest-rate%20market%20as%20a%20Uniswap%20v4%20pool%3A%20the%20curve%20quotes%20a%20yield%20rather%20than%20a%20price%2C%20and%20https%3A%2F%2Fgithub.com%2Fnirholas%2Fyield-space) · [Share on LinkedIn](https://www.linkedin.com/sharing/share-offsite/?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fyield-space) · [Submit to Hacker News](https://news.ycombinator.com/submitlink?u=https%3A%2F%2Fgithub.com%2Fnirholas%2Fyield-space&t=yield-space%3A%20An%20interest-rate%20market%20as%20a%20Uniswap%20v4%20pool%3A%20the%20curve%20quotes%20a%20yield%20rather%20than%20a%20price%2C%20and) · [Share on Reddit](https://www.reddit.com/submit?url=https%3A%2F%2Fgithub.com%2Fnirholas%2Fyield-space&title=yield-space%3A%20An%20interest-rate%20market%20as%20a%20Uniswap%20v4%20pool%3A%20the%20curve%20quotes%20a%20yield%20rather%20than%20a%20price%2C%20and)

## Built for AI agents too

Coding agents and LLM tooling can read this repo directly: [AGENTS.md](./AGENTS.md), [llms.txt](./llms.txt), [llms-full.txt](./llms-full.txt). Point an agent at `https://github.com/nirholas/yield-space` and it has the context it needs.

## More from the same author

- [All repositories by nirholas](https://github.com/nirholas/nirholas#readme): the full catalog, grouped by topic
- [three.ws](https://three.ws): the platform for 3D AI agents with Solana wallets, a skill marketplace and x402 payments
- Questions or ideas: [open an issue](https://github.com/nirholas/yield-space/issues) or [start a discussion](https://github.com/nirholas/yield-space/discussions)

## Contributors

[![Contributors](https://contrib.rocks/image?repo=nirholas/yield-space)](https://github.com/nirholas/yield-space/graphs/contributors)

<!-- /three.ws:growth -->
