# MarkoutFee

**A fee that learns. The pool measures whether its own past trades turned out to be informed, and charges the next one accordingly.**

A production Uniswap v4 hook. It prices every swap by overriding the pool's LP fee, so the value it captures is paid to in-range liquidity and never to the hook. No owner, no pause switch, no upgrade path.

- **Site:** https://markout-fee.pages.dev
- **Catalogue:** https://hookforge.pages.dev
- **Contract:** [`src/hooks/MarkoutFeeHook.sol`](src/hooks/MarkoutFeeHook.sol)
- **Licence:** Apache-2.0

## How it works

Every dynamic-fee hook published so far prices a swap from something observable at the moment it arrives: the size, the recent volatility, the gas it bid, the distance from an oracle. All of those are proxies for the question that actually matters, which is whether the person on the other side knew something. None of them measure it, because at the moment a swap arrives that fact has not happened yet.

It has happened a few seconds later. If a swap pushed the price down and the price kept falling, the seller was right and the liquidity that filled them lost; that is an informed trade, and it is the cost the LP literature calls adverse selection. If the price came back, the swap was noise and the fee it paid was pure revenue.

This is the markout that every market maker computes on their own flow, and a pool has everything needed to compute it: the tick it was left at, and the tick it is at now. So this hook grades each swap after the fact. On the next swap it compares the current tick to the tick the previous swap left behind, decides whether that swap was informed, and folds the verdict into an exponentially weighted average.

The fee it quotes is: fee = baseFee + maxSurcharge * toxicity A pool whose flow is mostly retail converges toward `baseFee` and becomes cheap. A pool being picked off converges toward the cap and becomes expensive, without anybody deciding that, and without an oracle to manipulate or a governance process to capture. The score is public, so anything else on-chain can read a pool's measured toxicity rather than guessing at it.

Two details keep the measurement honest. A verdict is only recorded once `minHorizon` seconds have passed, because a comparison inside the same block is measuring the swap's own price impact rather than what happened next. And the verdict is graded by magnitude rather than treated as a coin flip: a move that continues by a tenth of a tick is weak evidence, a move that continues by a hundred ticks is strong, so the observation is scaled and clamped instead of being rounded to a yes or a no.

{FlowClassifierHook} measures a related quantity and deliberately does nothing with it; this hook is the other half of that pair, the one that acts on what it measures. The hook takes no fee for itself, custodies nothing, and has no privileged role. `baseFee`, the cap, the smoothing and the horizon are fixed before the pool exists and can never be changed.

## Prior art

Dynamic fees keyed on realized volatility, swap size, price movement or an oracle gap are all well covered, and markout is the standard way a market maker grades its own flow off-chain. Computing markout on-chain, from the pool's own tick history, and feeding it back as the pool's fee, is the contribution here. It is the difference between reacting to a proxy for adverse selection and measuring the thing itself.

## Where it does not help

The verdict on a swap arrives with the next swap, so a pool that trades once a day prices today's flow on yesterday's evidence, and a pool with no second swap never grades the first. It is a lagging signal by construction: a regime change is paid for at the old rate until the average catches up. Note also that a sustained one-directional run grades as informed, because every swap in it is followed by one pushing the same way. That is the intended reading rather than a flaw, but it does mean a pool tracking a strong trend will quote its trend-following flow expensively even when that flow is not picking anyone off.

## Using it

Uniswap v4 removed `hookData` from `initialize`, so per-pool parameters arrive out of band. Fix them for a pool key whose pool does not exist yet, then initialize. Nobody can change them afterwards, including you.

```solidity
hook.configure(
    key,
    MarkoutFeeHook.Config({
        baseFee: /* uint24 */ 0,
        maxSurcharge: /* uint24 */ 0,
        alphaWad: /* uint64 */ 0,
        minHorizon: /* uint32 */ 0,
        saturationTicks: /* uint24 */ 0
    })
);

poolManager.initialize(key, startingSqrtPriceX96);
```

The pool's `fee` field must be `LPFeeLibrary.DYNAMIC_FEE_FLAG`. The hook rejects a pool initialized without it.

### Parameters

| Parameter | Type | Units |
| --- | --- | --- |
| `baseFee` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `maxSurcharge` | `uint24` | hundredths of a bip (`3000` = 0.30%) |
| `alphaWad` | `uint64` | fixed point, `1e18` = 1.0 |
| `minHorizon` | `uint32` | seconds |
| `saturationTicks` | `uint24` | ticks |

## What it reverts with

| Error | Meaning |
| --- | --- |
| `FeeTooLarge(uint24)` | A fee was configured above the protocol maximum of 100%. |
| `InvalidSaturation()` | `saturationTicks` of zero would make every move maximally informed. |
| `InvalidSmoothing()` | `alphaWad` must be in (0, 1e18]: zero never learns, above one overshoots. |
| `NotDynamicFee()` | The hook was attempted to be initialized with a non-dynamic fee. |
| `PoolAlreadyInitialized()` | The pool already exists, so its configuration is final. |
| `PoolNotConfigured()` | The pool was initialized without a configuration for this hook. |
| `SurchargeTooLarge()` | `baseFee + maxSurcharge` must leave room under the 100% protocol maximum. |

## The callbacks it claims

Uniswap v4 reads a hook's permissions from the low fourteen bits of its own address, which is why deploying one means mining a CREATE2 salt. This hook claims 3 of the fourteen:

- `afterInitialize`
- `beforeSwap`
- `afterSwap`

Mask: `0x10c0`, so every deployment of this hook has an address ending in those bits.

## It says what it is, on-chain

Every hook in this family implements `IHookMetadata`: four view functions that let an indexer, a wallet, a router or an agent identify a hook from its address alone, with no registry in the loop.

```bash
cast call $HOOK "hookName()(string)"    # MarkoutFee
cast call $HOOK "hookVersion()(string)" # 1.0.0
cast call $HOOK "specURI()(string)"     # the machine-readable manifest
cast call $HOOK "hookTags()(string[])"  # mev, dynamic-fee, markout, adverse-selection, oracle-free
```

The manifest this repository ships as [`hook.json`](hook.json) is what `specURI()` points at.

## Build and test

```bash
git clone --recurse-submodules https://github.com/nirholas/markout-fee
cd markout-fee
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
