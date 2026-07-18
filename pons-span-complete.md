# Pons Span ($SPAN) Test Token
### The first span of pons.family

> *"Every bridge starts with a single span."*

**Contract Address (CA):** `0x47c54fb66467944f9fbe9c6e57ec0b8bbf29869a` 

> ⚠️ **This is a test token.**

---

## 1. What It Is

**Pons Span** is the test token of [pons.family](https://pons.family) — the very first `$SPAN` ever minted by the `PonsLaunchFactory`, at the address everyone now recognizes at a glance by its `...bbbb` suffix.

It wasn't created to be traded. It was created to be **measured**: every parameter of its launch — graduation threshold, anti-snipe window, wallet limits — was chosen to sit exactly at the edge of the protocol's rules, not comfortably in the middle. It's the token the team uses to ask *"what if someone bought one wei more than allowed, in the very same block as the launch?"* — and to already have the answer ready.

Hence the name: a *span* is the stretch between two piers. It's the part of a bridge that carries the load without touching the ground — exactly like the *one-sided* liquidity position the factory creates on every launch: the entire supply parked on one side of the price, suspended in a single range, before the first buyer even shows up.

---

## 2. The Story

Back when `pons.family` was still a `.sol` file without a single test, someone on the team asked: *"let's launch the first token — not for the market, for us."*

That's how **Pons Span** came to be. No influencer squad, no 3am Telegram call, no countdown. Just a `launchToken()` call from a whitelisted wallet, with a salt patiently searched until the address stopped being random and started ending in `bbbb` — the mark every token born from pons.family carries forever, like a factory serial number.

For ten blocks, `$SPAN` lived under the strictest rules the contract knows how to enforce: no wallet could accumulate more than 2% of supply by buying from the pool, no single transaction could move more than 2.2% of it at once, and anyone attempting a snipe in the same block as the launch — anyone who wasn't the seed-buy recipient — had their transaction rejected outright.

After the tenth block, the restrictions fell away on their own — not switched off by an admin, but expired by design, exactly as the `constructor` intended from the start. From that moment on, `$SPAN` became what every token launched after it would be: free, tradable, indistinguishable from the rest — except for one thing. Having been first.

---

## 3. Fixture Specification

Every parameter below is chosen for a specific technical reason — not arbitrarily. This is the sheet used to reconstruct $SPAN identically across every test run and every protocol version.

### Token identity

| Field | Value | Notes |
|---|---|---|
| `name` | Pons Span | — |
| `symbol` | SPAN | Short, doesn't collide with common DeFi tickers |
| `logo` | `ipfs://pons-span-logo` | Stable placeholder |
| `description` | *"Canonical genesis fixture for the pons.family test suite"* | Openly stated: not a token meant for the market |
| `feeWallet` | unset by default | Deployer collects LP fees by default; a variant enables fee redirect |

Social fields (`telegram`, `discord`, `farcaster`) are left **intentionally blank**, proving the protocol doesn't require them to be filled in.

### Launch parameters

| Parameter | Value | What it tests |
|---|---|---|
| `initialTick` | `-60000` | Negative and an exact multiple of `tickSpacing` (60): no rounding in range math; tends to trigger the `isToken0 = false` branch |
| `tickSpacing` | `60` | Consistent with `poolFee = 3000` |
| `poolFee` | `3000` | Standard reference V3 fee tier |
| `maxWalletBps` | `200` (2%) | Base for the wallet cap during the restricted window |
| `maxTxBps` | `220` | **The only value the contract accepts** for this `maxWalletBps` — derived from `200 × 110 / 100`. Off by one basis point and validation fails |
| `restrictionBlocks` | `10` | Long enough to have an interior block beyond the launch block, short enough to test cheaply |
| `supply` | `1,000,000` | Well above the minimum floor; bps math never rounds to zero |
| `graduationThreshold` | `10 ether` | Deliberately low: crossable within a single test scenario |

### Derived negative-path variants

Each changes exactly **one field** from the sheet above:

| Variant | Change | Expected result |
|---|---|---|
| Invalid bps | `maxTxBps` → 221 | Rejected at configuration time |
| Zero supply | `supply` → 0 | Rejected: below minimum floor |
| Missing pair token | `pairToken` → zero address | Rejected: no quote asset |
| Disabled config | `enabled` → false | Launch blocked despite valid parameters |
| No graduation | `graduationThreshold` → 0 | Always reports "not graduated" |
| Deadline router | DEX-side variant | Covers the classic V3 router path |

---

## 4. Contract Technical Reference

$SPAN is produced and governed by two contracts. This section documents the functions that actually shape its lifecycle — not the full ABI, only what matters for understanding how $SPAN behaves.

### 4.1 `PonsLaunchFactory` — the entrypoint

| Function | Role for $SPAN |
|---|---|
| `launchToken(TokenParams, launchConfigId, dexId, salt)` | The single atomic call that deployed $SPAN: mints the token via CREATE2, creates/initializes the Uniswap V3 pool, seeds a one-sided liquidity position across the full supply, hands the position NFT to the locker, and optionally executes a seed buy with any native value sent above `launchFee`. |
| `predictTokenAddress(...)` / `predictVanityTokenAddress(...)` | Used before the real launch to compute $SPAN's deterministic CREATE2 address and to search for the salt that produces the required `bbbb` suffix. |
| `hasVanitySuffix(address)` | Confirms $SPAN's address ends in `bbbb`, the mark shared by every pons.family launch. |
| `graduationStatus(token)` | Reads $SPAN's *locked* Uniswap V3 position (not the pool's raw balance) and reports `pairedPrincipal`, the configured `threshold`, and whether `graduated` is true. Direct token donations to the pool do not affect this. |
| `getLaunchedToken(token)` | Returns the permanent on-chain record for $SPAN: deployer, paired token, position manager, position ID, dex/config IDs, restriction end block, supply, and initial buy amount. |
| `addDexConfig` / `addLaunchConfig` / `updateLaunchConfig` | Owner-only functions that registered $SPAN's DEX wiring and launch configuration before it could be launched. |
| `setLaunchEnabled` / `setWhitelistedLauncher` | Governed whether $SPAN's launch required whitelisting or was open to any caller. |

### 4.2 `PonsLauncherToken` — the ERC-20 itself

| Function | Role for $SPAN |
|---|---|
| `constructor(...)` | Mints $SPAN's entire fixed supply directly to the factory in the same transaction as deployment, records `launchBlock` and `restrictionEndBlock`, and stores metadata (logo, description, socials) immutably where applicable. |
| `_update(from, to, value)` *(internal, overrides ERC-20 transfer logic)* | The anti-snipe engine. While `block.number <= restrictionEndBlock`, any transfer originating from $SPAN's Uniswap V3 pool is checked: blocked outright in the launch block unless the recipient is the designated seed-buy address, and otherwise capped by `maxWalletLimit()` and a cumulative `maxTxLimit()`. Wallet-to-wallet transfers are never restricted. |
| `liquidityPool()` | Resolves $SPAN's canonical Uniswap V3 pool address from the DEX factory, pair token, and fee tier. |
| `maxWalletLimit()` / `maxTxLimit()` | Compute the live wallet and transaction caps as a percentage of $SPAN's total supply, per the `maxWalletBps` / `maxTxBps` fixture values. |
| `setInitialBuyRecipient(address)` | Callable only by `launchFactory`; temporarily exempts the seed-buy recipient from the launch-block block, then is cleared immediately after. |
| `socials()` / `getTokenInfo()` | Read-only views exposing $SPAN's creator, logo, description, and social links for frontend display. |
| `_isPairPool(candidate)` | Recognizes not just $SPAN's canonical pool but any other fee-tier pool for the same pair as "restricted-scope," by verifying the candidate against the DEX factory. |

---

## 5. What $SPAN Actually Proves

- **Reference fixture** — every new `LaunchConfig` or `DexConfig` proposal is checked against "what would $SPAN's config have done here."
- **Graduation canary** — its low threshold proves `graduationStatus` reads only locked liquidity, never pool donations.
- **Living proof of the anti-snipe logic** — the only token that has completed its full restriction window under observation, block by block.
- **A bridge between versions** — relaunching $SPAN with identical parameters on a new factory version is the first migration sanity check.

---

## 6. The Future of $SPAN

**To be unambiguous: $SPAN is a test token today, with no active utility for holders.** It is not marketed, not liquid by design, and not intended to be bought or sold as a speculative asset. Every planned role below is protocol-facing, not holder-facing:

- **Launchpad v2** — before any new `PonsLaunchFactory` version ships, a second `$SPAN` (same parameters, new `bbbb` address) will be relaunched on it, to guarantee behavioral continuity across protocol generations.
- **Living documentation** — $SPAN's parameters become the canonical worked example in pons.family's public docs.
- **Locker certification** — any new `IPonsLaunchLocker` implementation must prove it can correctly custody $SPAN's position NFT before being promoted to production.

It doesn't need its price to go up. Its only job is to stay exactly as it is — so that everything launched *after* it, on the bridge that carries its name, can bear the load.

---

*Pons Span ($SPAN) — the first span holds the bridge. Test token. No holder utility today. Protocol reference standard, permanently.*
