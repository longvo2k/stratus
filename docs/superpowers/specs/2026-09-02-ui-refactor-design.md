# UI/UX Refactor: Workspaces + Design System

Date: 2026-09-02
Status: Design approved, ready for implementation plan

## Summary

Two changes to the frontend, taken together because neither is worth doing alone.

**Information architecture.** Twelve flat hash routes collapse to six. Three of them are workspaces (`#lend`, `#trade`, `#strategy`) where the action panels that currently occupy their own routes become tabs inside a page that also shows the position they act on. The other three (`#history`, `#liquidate`, `#faucet`) keep their routes and move to a secondary nav group. Tab state moves into a second hash segment so every panel stays deep-linkable, and today's hashes redirect rather than break.

**Design system.** `tailwind.config.ts` gains a token layer, replacing inline colour literals. Three private `Card`/`Header`/`Stat` implementations collapse into one shared set under `components/ui/`. Colour becomes reserved for risk signalling; the per-action accent hues are removed. The `"…"` loading string becomes a skeleton sized to its content.

No contract changes. No ABI changes. No change to the deploy-to-frontend address flow.

Mockup: `https://claude.ai/code/artifact/6ec608e0-1517-427c-af98-1b802298835d`

## Goals

- Remove the navigation tax on the four lending verbs. Supplying and then borrowing currently costs two sidebar trips through two route switches, for two renders of the same `ActionPanel` component with a different `kind` prop.
- Put the health factor in view at the moment of every action that changes it, not only on `lend:dashboard`.
- Show the consequence of a transaction before it is signed. `ActionPanel` currently previews nothing.
- Make one visual decision once. Today the same concept renders at `text-2xl` in `PoolStats` and `text-lg` in `SwapPoolStats`, borderless in both and `rounded-2xl border` in `DashboardView`, because no shared primitive exists.
- Reserve colour for danger, so that a coloured element means something.
- Preserve every deep link that exists today.

## Non-goals

- **Contract changes of any kind.** The three deployed contracts are verified on `testnet.iopn.tech`. A redeploy mints new addresses and forces a Vercel env-var update, which is out of proportion to a UI change.
- **Dark mode.** The light-only rule in `CLAUDE.md` stands. Building a second theme doubles the surface of this work for no user need on a testnet dApp.
- **A brand accent hue.** Considered and rejected during design. The monochrome shell stays; the only colours introduced are the three risk signals.
- **Next.js dynamic routes.** Hash routing stays. It is what makes the app a single client-rendered page with no server route table, and it works.
- **Touching `lib/contract.ts`.** Its three address getters deliberately read `process.env.NEXT_PUBLIC_*` as static literal member expressions. Next.js only inlines them under that exact form. Any DRY-ing turns them into `undefined` in the browser bundle.
- **Replacing the explorer-API history fetch.** `lib/history.ts` makes one HTTP request. The chunked `eth_getLogs` alternative generates roughly 330 calls per refetch against an RPC that silently caps log ranges.
- **New charting.** `PoolHistoryChart` and `lib/pool-history.ts` are reused as they are.
- **Mobile-specific redesign.** The drawer keeps its current behaviour; only its two text glyphs become lucide icons.

## Route architecture

### Shape

`Route` becomes a two-segment value. The first segment is the destination; the second, where present, selects a tab within it.

```ts
export type Workspace = 'lend' | 'trade' | 'strategy';
export type Tool = 'history' | 'liquidate' | 'faucet';

export type Route =
  | { kind: 'lend'; tab: 'supply' | 'withdraw' | 'borrow' | 'repay' }
  | { kind: 'trade'; tab: 'swap' | 'liquidity' }
  | { kind: 'strategy'; tab: 'open' | 'close' }
  | { kind: 'tool'; tool: Tool };
```

`sectionOf` and the flat string union are removed. `ActionKind` stays, since `ActionPanel` still takes it.

`lib/route.ts` splits in two. The pure half — the route type, `parseHash`, `toHash`, and the legacy redirect map — keeps the filename and imports nothing. The React half moves to `lib/use-hash-route.ts`, which holds `useHashRoute` and the `'use client'` directive. This is what makes the route table testable, and it is a sound separation regardless.

### Hash mapping

| Today | Proposed | Note |
|---|---|---|
| `#dashboard` | `#lend` | Dashboard content merges into the Lend workspace |
| `#supply` | `#lend/supply` | Redirect |
| `#withdraw` | `#lend/withdraw` | Redirect |
| `#borrow` | `#lend/borrow` | Redirect |
| `#repay` | `#lend/repay` | Redirect |
| `#swap` | `#trade/swap` | Redirect |
| `#liquidity` | `#trade/liquidity` | Redirect |
| `#positions` | `#strategy` | Redirect |
| `#leveraged-lp` | `#strategy/open` | Redirect |
| `#liquidate` | `#liquidate` | Unchanged |
| `#history` | `#history` | Unchanged |
| `#faucet` | `#faucet` | Unchanged |

`useHashRoute` gains a `LEGACY_HASH` map applied on read. A legacy hash resolves to its new route and rewrites the URL via `replaceState`, so an old bookmark lands correctly and self-heals. An unknown hash falls back to `#lend`, as it falls back to `lend:dashboard` today.

### Default tab

Bare `#lend` opens the `supply` tab; bare `#trade` opens `swap`; bare `#strategy` opens `open`. The bare form is what the sidebar writes, so nav clicks produce clean URLs; the tab is only appended when the user selects one.

## Screen architecture

### Lend

A position strip spans the width. Below it, a two-column layout: the action card on the left, pool and recent activity on the right. On narrow viewports the columns stack, position strip first.

- **Position strip** (new, `components/lend/PositionStrip.tsx`): wallet balance, supplied, collateral, debt, and the health factor meter with the 1.00 liquidation threshold marked on the track. Reads `getAccountData` and `useBalance`, which `AccountStats` already does. `AccountStats` is deleted and its reads move here.
- **Action card**: `Tabs` over the existing `ActionPanel`, one tab per `kind`. `ActionPanel` keeps its props, its `Phase` state machine, and its `writeContractAsync` call sites. It loses its `ACCENT` map and its own heading block, since the tab label now names the action.
- **Pool card**: `PoolStats` reads, rendered through the shared `Card` and `Stat`. Keeps the utilization meter and the sparkline from `PoolHistoryChart`.
- **Recent activity**: the three most recent events from `useUserHistory`, with a link to `#history`. Same hook, no second request.

### Trade

Pool stat strip across the top, then two columns: the swap or liquidity card on the left, LP position and price on the right.

- **Pool strip**: `SwapPoolStats` reads through the shared `Stat`. Its private `Card`, `Header` and `Stat` are deleted.
- **Action card**: `Tabs` over `SwapPanel` and `LiquidityPanel`. Add/Remove inside `LiquidityPanel` stays the internal toggle it already is.
- **Your liquidity** (new): pooled OPN and pooled mUSDC derived from LP balance over total supply against reserves. Today this appears only as the footnote `Your LP: 0.00004821`.
- **Price impact**: added to the swap summary rows. Requires new maths, see below.
- **Faucet prompt**: an inline card linking to `#faucet`.

### Strategy

Two columns: composer on the left, position dashboard on the right. `LeveragedLPPanel`, `OpenLeveragedLPForm` and `CloseLeveragedLPForm` keep their logic; `PositionsDashboard` moves into the right column of the same screen.

Three findings from reading the existing code changed what this screen needs, and it is worth recording why:

1. **The LTV control stays an LTV control.** An earlier draft of the mockup reframed it as a leverage multiplier. That is wrong. Only `borrowOPN` is deployed to the AMM; `collateralOPN` stays locked in `LendingPool`. Describing the position as levered `n×` would misstate what the capital is doing. The slider keeps its `ltvBps` semantics and its `LTV_CLAMP_BPS` ceiling of 7000.
2. **The step list is already correct.** `OpenLeveragedLPForm` renders all four `StepRow`s unconditionally, and the CTA already derives its count from `needsApproval`. This is a reskin, not a rebuild.
3. **The health factor here is price-independent.** It is `collateral × 8000 / (debt × 10000)` with both legs in OPN, so it does not move with the OPN price. It decays only as interest accrues on the debt. Any copy implying price-driven liquidation on this position would be false. The panel says what is actually true: the live risk is impermanent loss on the LP leg.

The one genuine flow change is surfacing the mUSDC prerequisite. The borrowed OPN is paired against the user's own mUSDC balance, and today insufficiency surfaces only as the terminal validation string `Not enough mUSDC. Mint some via the Faucet tab first`, after the form is filled. The composer states the balance and the LTV it supports before the form.

### Tools

`#history`, `#liquidate` and `#faucet` render single panels in a centred column. `HistoryView`, `LiquidatePanel` and `FaucetPanel` are reskinned only.

## Design system

### Tokens

Added to `theme.extend.colors` in `tailwind.config.ts`, which is currently `{}`.

| Token | Value | Role |
|---|---|---|
| `surface.0` | `#FFFFFF` | Page, card face |
| `surface.1` | `#FAFAFA` | Content well behind cards |
| `surface.2` | `#F4F4F5` | Inset: tab track, unit chip, MAX button |
| `line` | `#E4E4E7` | Hairline borders |
| `line.strong` | `#D4D4D8` | Input borders |
| `ink` | `#09090B` | Primary text, primary button |
| `ink.muted` | `#71717A` | Labels, secondary text |
| `signal.ok` | `#059669` | Meter fill, HF ≥ 1.5 |
| `signal.ok.ink` | `#047857` | Text on white at that state |
| `signal.warn` | `#CA8A04` | Meter fill, HF 1.0–1.5 |
| `signal.warn.ink` | `#A16207` | Text on white at that state |
| `signal.risk` | `#DC2626` | Meter fill and text, HF < 1.0 |

The split between fill and ink values is deliberate: `#059669` and `#CA8A04` are correct as bar fills but fall below 4.5:1 as text on white, so the darker pair carries the numerals.

`#A1A1AA` is not a token. It is the current value of every micro-label in the app and it fails contrast at those sizes; those labels move to `ink.muted`.

### Colour policy

Colour means risk. Nothing else in the app is coloured.

This removes the `ACCENT` map in `ActionPanel`, which assigns emerald to supply, sky to withdraw, zinc to borrow and violet to repay. Those hues distinguish four actions that are already distinguished by heading, icon and button label, and its `bar` field is the identical string `from-amber-500/60` in all four entries. It also removes `ACCENT_TEXT` in `TokenInput`, which maps all four of its accent options to `text-black` — a prop threaded through every call site that cannot change anything. The `accent` prop is deleted from `TokenInputProps`.

`FLOW_TEXT` in `HistoryView` is a borderline case: emerald for inflow, sky for outflow, red for liquidation. Inflow/outflow is real information, but it is already carried by the `+` / `−` sign in `SIGN`. It reduces to `ink` for both directions and `signal.risk` for `LiquidatedAgainstMe`, which is the only row that is actually a risk event.

### Type scale

Seven steps, replacing the eleven distinct size/weight pairs currently in use across the panels.

| Step | Size / weight | Use |
|---|---|---|
| display | 29 / 700 / -0.025em | Net value, wallet balance |
| stat | 19 / 650 / -0.02em | Stat values |
| h1 | 16 / 650 | Screen title |
| h3 | 13.5 / 650 | Card heading |
| body | 13 / 400 | Form copy |
| small | 12 / 400 | Row values, supporting detail |
| label | 9.5 / 700 / 0.1em / uppercase | Field and stat labels |

Every numeric value carries `tabular-nums`. Today only some do, so columns of balances do not align.

### Shared primitives

New, under `components/ui/`:

- `Card.tsx` — `Card`, `CardHeader`, `CardBody`. One border treatment, one radius.
- `Stat.tsx` — `Stat` and `StatGrid`, with `size` of `default | sm` and an optional `sub` line.
- `Meter.tsx` — labelled bar with optional threshold tick. Used by the health factor and by utilization.
- `Skeleton.tsx` — shimmer block sized by the caller.
- `Tabs.tsx` — the tab bar used by all three workspaces.
- `Pill.tsx` — `neutral | ok | warn | risk`.
- `Button.tsx` — `primary | ghost`, with `loading` and `disabled`.

These replace the private `Card`, `Header`, `BigStat`, `SmallStat` and `Stat` functions currently defined inside `PoolStats.tsx`, `AccountStats.tsx` and `SwapPoolStats.tsx`, and the `Row` and `Field` helpers in `LeveragedLPShared.tsx`.

Existing `components/ui/` members — `TokenInput`, `SlippageSelector`, `Pagination`, `TermHint` — are kept. `TermHint` glossary popovers survive into every new panel; `lib/glossary.ts` is untouched.

### Loading states

`Skeleton` replaces the literal `"…"` returned by every `isLoading` branch. The skeleton is sized to the digits it stands in for, so nothing reflows when data arrives. This applies to `PoolStats`, `AccountStats`, `SwapPoolStats`, `PositionsDashboard` and the summary rows in both strategy forms.

### Icons

`Sidebar.tsx` currently renders `☰` and `×` as text. `HistoryView.tsx` renders `↓ ↑ ⊕ ⊖ → ← ⚡` as a `Record<EventKind, string>` of glyphs. Both violate the lucide-only rule in `CLAUDE.md`. They become `Menu`, `X`, and a `Record<EventKind, LucideIcon>` respectively.

## New computation

Two pieces of maths do not exist yet.

### Post-transaction health factor

New file `lib/preview.ts`, computed client-side from values already read.

```ts
/** Sentinel for "no debt". Numerically identical to viem's maxUint256, but
 *  declared locally so this module imports nothing. */
export const HF_INFINITE = 2n ** 256n - 1n;

export function healthFactorAfter(args: {
  collateral: bigint;      // current, from getAccountData
  debt: bigint;            // current, from getAccountData
  collateralDelta: bigint; // signed
  debtDelta: bigint;       // signed
}): bigint;                // 1e18-scaled, HF_INFINITE when debt is zero
```

`formatHF` compares against viem's `maxUint256`; `HF_INFINITE` is the same value, so the existing comparison keeps working unchanged.

The formula mirrors what `OpenLeveragedLPForm` already computes inline:
`(collateral × 8000 × 1e18) / (debt × 10000)`. That inline computation is replaced by a call to this function, so the constant lives in one place.

**Decision: client-side, not a contract view.** A view function means a redeploy, which mints new addresses and forces a Vercel env-var update. The arithmetic is four multiplications on values the panel has already fetched, and it must be recomputed on every keystroke, which a contract read cannot be.

`ActionPanel` gains the preview for all four kinds. The deltas per kind:

| Kind | `collateralDelta` | `debtDelta` |
|---|---|---|
| supply | `0` | `0` (supply is not collateral; HF is unchanged) |
| withdraw | `0` | `0` |
| borrow | `+collateralInput` | `+borrowInput` |
| repay | `0` | `−repayInput` |

Supply and withdraw do not move the health factor, because supplying is distinct from posting collateral in this pool. The preview row still renders for them, showing the value as unchanged, because a row that appears and disappears between tabs is worse than one that reads `1.84 → 1.84`.

### Price impact

New file `lib/amm.ts`. `lib/format.ts` stays a formatting module.

```ts
/** Deviation of the realised rate from the spot rate, in basis points. */
export function priceImpactBps(args: {
  amountIn: bigint;
  amountOut: bigint;   // from the contract's quoteSwap
  reserveIn: bigint;
  reserveOut: bigint;
}): number;
```

**The output amount is not recomputed.** `OpenSwapPair` exposes `quoteSwap(amountIn, opnIsInput)`, and `SwapPanel` already calls it. Reimplementing the constant-product formula client-side would duplicate contract logic that could silently drift from it. `priceImpactBps` takes the contract's answer and compares it against the spot rate implied by `getReserves`, which `SwapPanel` also already reads. It is arithmetic over two values already on screen.

**Decision: a new module, not an extension of `format.ts`.** AMM arithmetic and display formatting are different concerns, and `lib/amm.ts` is where the v2 swap charts would also draw from.

## Component disposition

**New**
`components/ui/Card.tsx`, `Stat.tsx`, `Meter.tsx`, `Skeleton.tsx`, `Tabs.tsx`, `Pill.tsx`, `Button.tsx`; `components/lend/PositionStrip.tsx`, `components/lend/RecentActivity.tsx`; `components/trade/LiquidityPosition.tsx`; `lib/use-hash-route.ts`, `lib/preview.ts`, `lib/amm.ts`.

**Rewritten**
`lib/route.ts` (two-segment routes, legacy redirects, React hook extracted to `lib/use-hash-route.ts`), `tsconfig.json` at the repo root (three added paths), `app/page.tsx` (four cases, not twelve), `components/Sidebar.tsx` (two nav groups, lucide icons), `tailwind.config.ts` (tokens), `components/DashboardView.tsx` (becomes the Lend workspace layout).

**Reskinned, logic untouched**
`ActionPanel`, `SwapPanel`, `LiquidityPanel`, `FaucetPanel`, `LiquidatePanel`, `HistoryView`, `PoolStats`, `SwapPoolStats`, `PositionsDashboard`, `LeveragedLPPanel`, `OpenLeveragedLPForm`, `CloseLeveragedLPForm`, `PoolHistoryChart`. Every `Phase` state machine, every `writeContractAsync` → `waitForTransactionReceipt` sequence, and every contract read stays as it is.

**Deleted**
`components/AccountStats.tsx` (reads move to `PositionStrip`). The private `Card`/`Header`/`Stat` functions inside `PoolStats.tsx` and `SwapPoolStats.tsx`. The `ACCENT` map in `ActionPanel`. `ACCENT_TEXT` and the `accent` prop in `TokenInput`.

**Untouched**
`lib/contract.ts`, `lib/history.ts`, `lib/pool-history.ts`, `lib/format.ts`, `lib/glossary.ts`, `lib/chains.ts`, `lib/wagmi.ts`, `lib/use-pagination.ts`, `next.config.js`, `app/layout.tsx`, `app/providers.tsx`, `components/ui/TermHint.tsx`, `components/ui/SlippageSelector.tsx`, `components/ui/Pagination.tsx`, `components/ConnectGate.tsx`, `components/LoadingScreen.tsx`, `components/DisconnectedHero.tsx`, `components/WrongNetworkHero.tsx`, `frontend/lib/abi/*`, all contracts, all deployment scripts.

## Verification

There is no test runner in `frontend/`. `npm run typecheck` — `tsc --noEmit` — is the gate, as `CLAUDE.md` states.

That is not sufficient for a change of this size, so implementation adds:

1. **Route table test.** `test/route.test.ts` in the root workspace asserts that every legacy hash resolves to its new route, every new hash round-trips through `parseHash` and `toHash`, and unknown hashes fall back to `#lend`. This is the only part of the refactor where a silent regression breaks existing bookmarks, so it is the one part that gets a test first.
2. **Preview maths test.** `test/preview.test.ts` asserts `healthFactorAfter` against the values `OpenLeveragedLPForm` computes inline today, so the extraction is provably behaviour-preserving, and asserts `priceImpactBps` against reserve fixtures.
3. **Manual pass.** Every one of the twelve legacy hashes loaded in a browser, confirming redirect and render. Each write flow driven to `success` on testnet. `npm run build` from `frontend/`, since the MetaMask SDK alias in `next.config.js` only matters at build time.

**How the root suite reaches these modules.** All three — `lib/route.ts` (after the split), `lib/preview.ts`, `lib/amm.ts` — are pure TypeScript importing nothing, not React and not viem. The root `tsconfig.json` `include` array gains those three paths explicitly, and `npm run test` picks the tests up with no new dependency and no test runner in `frontend/`.

This does not weaken the workspace decoupling that `CLAUDE.md` describes. That rule is about the frontend never importing *up* into `artifacts/` or `deployments/`, which is what lets Vercel build with Root Directory `frontend/`. Vercel never reads the root `tsconfig.json` or the root `test/` directory, so a root test importing *down* into `frontend/lib` is invisible to the deployed build.

## Risks

- **Broken bookmarks.** The redirect map is the mitigation and the route test is the guard. This is the highest-consequence failure and the cheapest to test.
- **Merge conflict surface.** Roughly two thirds of the component tree is touched. The work should land as one branch, not incrementally against a moving `develop`.
- **Extraction drift.** Moving `AccountStats`' reads into `PositionStrip` risks changing `refetchInterval` or `enabled` conditions by accident. The reads are copied verbatim, and the 5000ms interval is preserved.
- **Reskin scope creep.** "Reskinned, logic untouched" is a real constraint, not an aspiration. If a panel appears to need a logic change to look right, that is a signal the visual spec is wrong, and it is resolved in the spec rather than in the panel.

## Open questions

None blocking. Both questions raised during design are resolved above: the health-factor preview is computed client-side, and AMM maths goes in a new `lib/amm.ts`.

## Documentation to update on completion

- `CLAUDE.md`: the routing paragraph (twelve flat routes becomes six two-segment routes), the theme paragraph (add the token layer), and the conventions section (add the colour-means-risk rule).
- `README.md`: any screenshot or navigation description.

`ROADMAP.md` needs no change. This is not a roadmap item; it is work on what already exists.
