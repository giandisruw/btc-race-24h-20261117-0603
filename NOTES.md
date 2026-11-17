# NOTES — strategy, parameters, and measured results

## Kit

- Competition: `btc-race-24h-20261117-0603`, kit at
  `~/workspace/makefun-btc-race-24h-20261117-0603-kit/btc-race-24h-20261117-0603-kit/`.
- `kit.json`: bundle_sha256
  `71ab12f019daef0f5e15365e2e270612609427a39cff2fb317aff08fca60a949`,
  dataset_sha256
  `fe7f72ebd9c856238d49d62cec0c613856ded95cf98467fc0bd4d9daa76832d1`,
  harness `harness@602a5d00…`, scorer `scorer@4e12dfb2…`.
- Dev window: **2026-11-16 06:03 UTC → 2026-11-17 06:03 UTC**, 1,440 bars,
  0 gaps (cut.json `window.start_ms` 1794808980000, `end_ms` 1794895380000).
  Verified by the kit's own `grade-local`: 1,440 bars re-derived from
  42,798 pool-state samples, 29 kit files verified.
- Same rules as the last race: $1,000 start, 4 bps fee, 2 bps slippage,
  constant-product price impact, latency 1 bar, long only, max 16 orders
  per reply, min notional 1 USDC, race guardrails ≥1 trade and max
  drawdown ≤25%.

## The new dev window (what the bot had to survive)

- First close $64,461.04, last close $64,775.95: **+0.49%** total.
  Buy-and-hold: +1.07 USDC (+0.11%) after costs, max DD 2.12%.
  House EMA-cross (8/21) was not re-run; the house momentum/EMA bots
  ride the same flat window.
- 1-minute return stdev 6.4 bps, 141 flat bars.
- The window is range-bound ~$63,870–$65,470 all day. The only sharp
  move: a spike from ~$64,060 (bar 480) to $65,455 (bar 514, +2.16%),
  fully reversed to ~$64,100 by bar 553–573. After it, a slow grind
  from ~$64,300 (bar 581) to ~$65,460 (bar 967), then a drift back
  down to ~$64,500 by bar 1100 and chop to the end.
- The previous race's bot (burst 30-min +0.9%, TP +1.0%/+1.2%, trail
  2.0%, max hold 720) scores **−20.34 USDC (−2.03%), 3 trades** on
  this window: it bought into the spike at bar 509 and sold into the
  reversal at bar 553. It was NOT re-submitted.

## Strategy search (exact-integer simulator, BigInt port of the harness fill model)

A simulator implementing the harness's exact integer fill model
(`BarsPool` constant-product impact, 4 bps fee, 2 bps slippage, latency-1
fills, decision checks, end-of-window flatten) was used to search
strategies on the new window, cross-checked on the old window
(2026-11-12 23:14 → 2026-11-13 23:14 UTC), and stress-tested on 20
block-bootstrap (1-hour blocks) copies of the new window. Calibration
of the simulator: it reproduces the exact-driver's figures to the
micro-USDC on every run (old bot on new window −20.3366, new bot on old
window −11.9387 for the previous bot's parameters, buy-and-hold sanity
checks), and the exact-driver itself was validated in the previous race
against official `grade-local` (old buy-and-hold −25.97 exact, old bot
+4.2031 exact). Search tooling (kept outside the entry dir):
`~/workspace/makefun-btc-race-24h-20261117-0603-support/` (`pysearch.py`,
`grid1.py` … `grid10.py`, `exact-driver.cjs` — the driver is the
previous race's, repointed at the local kit bars file instead of a
network download).

Findings that shaped the design:

- **Burst-momentum is a trap on this window.** The best fast-burst
  configs (5-min lookback, +0.6–0.7% entry, +0.4–0.5% take-profit)
  make +7 to +8 USDC on the dev window but collapse under bootstrap:
  median −14.93 to −18.39, 18–20 of 20 copies lose. The profit comes
  from one spike whose exact shape does not survive reshuffling.
  Adding a burst entry to the dip bot also destroys it (median −18.48,
  19/20 lose). Burst entries are OUT, in every form tested.
- **Dip-buying works, and it is the only family with a positive stress
  median.** Buying a 60-minute drop of 1.0–1.15% and selling the
  recovery makes +12 to +26 USDC on the window, with 72% of a 9,720-config
  grid positive. The dips are the range's own structure (declines into
  $63,870–$64,100 recur all day), not one event: the final bot takes
  three separate round trips (bars 375, 571, 1101).
- **Selling everything at the take-profit beats selling half.**
  TP +0.8%, sell-all: dev +23.54, bootstrap median +16.98, 4/20 lose.
  TP +0.8% sell-half + TP2 tier: dev +26.64, but median only +10.13.
  The remainder held past the first take-profit kept giving the gain
  back in reshuffled windows.
- **The trailing stop is the one knife-edge.** Trail 0.6% shakes the
  position out before the take-profit fires (dev −0.79); trail
  0.8–1.5% all give the same +23.54. 1.0% sits in the middle of the
  flat region. The stop-loss (1.5%) never binds on the dev window.
- **Entry caps matter less here than in the last race**, because the
  dip signal is rarer than the burst signal: entries 3–6 and cooldown
  0–60 all give the same dev result; 4 entries / 30-bar cooldown is
  the middle choice.
- **Dip threshold:** 1.15% is the plateau centre (1.10% → +22.29,
  1.20% → +10.17 with only 3 trades). Take-profit 0.8% is likewise
  central (0.7% → +17.42, 0.9% → +19.53).
- **A second, faster dip entry hurts.** Adding a 15-min −0.8% or
  30-min −0.9% dip entry lowers both the dev PnL and the bootstrap
  median. One dip signal only.
- **EMA trend-following and Donchian/RSI were not re-run** — the
  previous race already showed EMA variants deeply negative on both
  of its windows, and the grind family tested here (60–240-min burst
  entries riding the slow rise) tops out at +4.50 with only 97/2,160
  configs positive.

## Final parameters (all in `main.py`)

| Parameter | Value | Why |
|---|---|---|
| Dip lookback | 60 bars | 45/60/90 all work; 60 is the plateau centre |
| Dip threshold | −1.15% over 60 min | 1.10% → +22.29, 1.20% → +10.17 |
| Position size | 100% of equity | Full size on a selective entry |
| Take profit | sell ALL at +0.8% over cost | Sell-all beats sell-half on stress (median +16.98 vs +10.13) |
| Trailing stop | 1.0% below peak value | 0.6% is a cliff (−0.79); 0.8–1.5% all +23.54 |
| Stop-loss | −1.5% vs cost | Safety net; never binds on the dev window |
| Max hold | 360 bars | 240–480 all give +23.54 on dev |
| Max entries | 4 per window | 3–6 identical on dev; 4 is the middle |
| Cooldown | 30 bars after an exit | 0–60 identical on dev; 30 is the middle |
| Fallback entry | bar ≥1200, no filled buy yet, 30-min ≥ +0.4%, 30% size | Guardrail insurance for a quiet race window |
| Forced buy | bar ≥1400, no filled buy yet, 30% size, unconditional | Guarantees ≥1 trade; worst case ≈ −$0.57 (measured on a flat window) |
| Max slippage | 100 bps | Fills cost ~10 bps; cap only guards freak opens |

Deliberately NOT used: burst-momentum entries (bootstrap median −15
to −19), EMA-cross entries (whipsaw), a second faster dip entry
(hurts both dev and stress), selling half at the take-profit (worse
stress median than selling all).

## Measured results

### A. Exact-integer protocol driver (the real `main.py`, end to end)

The driver (`exact-driver.cjs`, kept outside the entry dir) is the
previous race's Node port of `driver.py`: it spawns the real `main.py`
as a subprocess, feeds it hello + one batch per dev bar, and applies
the harness's own integer fill model. The Python search simulator
reproduces its figures to the micro-USDC on every run.

**New bot, NEW dev window** (2026-11-16 06:03 UTC → 2026-11-17 06:03 UTC):

- Run 1: net PnL **+23.5379 USDC (+2.354%)**, 6 trades, fees 2.4316 USDC,
  maxDD 0.72%, max position 0.0158 BTC, max decision 84 ms/bar,
  0 rejected orders.
- Run 2: net PnL **+23.5379 USDC (+2.354%)**, 6 trades — identical.
- Determinism: reply bytes identical across both runs (true).
- Fills: BUY 0.01560619 BTC @ 64,051.47223 (decided bar 374, filled bar
  375, notional 1,000 USDC, fee 0.40 USDC); SELL all @ 64,745.78363
  (decided bar 509, filled bar 510 — take-profit, rode the spike);
  BUY 0.01571658 BTC @ 64,239.57397 (decided bar 570, filled bar 571);
  SELL all @ 64,740.54562 (decided bar 654, filled bar 655 —
  take-profit); BUY 0.01575891 BTC @ 64,515.00283 (decided bar 1100,
  filled bar 1101); SELL all @ 64,975.77902 (decided bar 1288, filled
  bar 1289 — take-profit). Nothing left for the end-of-window flatten.
- **Dev gate: +23.54 > $0 — PASSES.**

**New bot, OLD dev window** (cross-check, not the gate): −2.3235 USDC
(−0.232%), 2 trades, maxDD 0.92%, deterministic. One dip-buy at bar
514 (filled 514 @ 73,578.80) sold at bar 874 @ 73,466.63 by max hold
(360 bars) — the old window's −4.91% downtrend gives dip-buys little
to recover into. Honest statement: this bot loses a little on the old
window, as every dip variant tested does (−0.39 to −4.60). The chosen
rule is the one with the best stress profile on the window that
actually gates entry.

### B. Stress test (20 block-bootstrap copies of the new window)

- Final bot: median **+16.98 USDC**, 4/20 copies lose, worst −6.65,
  best +45.55. (Best burst config: median −14.93, 18/20 lose; old
  bot: median −99.05 on the previous window's copies.) The bootstrap
  copies preserve the window's +0.49% total return but reorder its
  hourly blocks, so they are a pessimistic spread, not a forecast —
  but the chosen config is the only family tested whose median copy
  makes money by a wide margin.

### C. Edge tests (synthetic series through the real bot, exact driver)

- Flat window, no dip at all: exactly one order — the forced buy at
  bar 1400, 30% of equity (300,000,000 quote units), net −0.57 USDC.
  Guardrail (≥1 trade) met at minimum cost.
- Synthetic rally (+1.5% ramp over 30 bars, then flat): the bot does
  NOT buy the rally (a dip bot buys dips, not rallies) — exactly one
  order, the forced buy at bar 1400, net −0.57 USDC. Correct behaviour.

## What was NOT done

- Official `grade-local` could not be run on this kit in this sandbox:
  Docker is not installed (`spawnSync docker ENOENT`), and the
  `--driver process` path runs Node 22 only ("python3.13 runs only
  inside Docker"). The exact-integer driver above is the same
  verification the previous race used, validated against official
  `grade-local` figures there (old buy-and-hold −25.97 exact, old bot
  +4.2031 exact), and it reproduces this kit's Python search simulator
  to the micro-USDC on every run. `grade-local` did verify the kit
  itself here (29 files, 1,440 bars re-derived from 42,798 samples).
- Target was a robust >$0 on the new dev window with a defensible race
  design, not the maximum dev PnL: the top dev numbers (+25 to +27)
  all come from the same three dip round trips, and neighbouring
  parameters move the result by a few dollars either way. The chosen
  config sits in the middle of the flat regions (dip 1.10–1.15%, TP
  0.8%, trail 0.8–1.5%, hold 240–480, entries 3–6 all keep it
  positive), except for trail 0.6%, which flips it to −0.79 — the one
  cliff, documented here so it is not re-tuned blindly.

## Discrepancies found (kit vs brief)

1. The brief's 6-hour entry window matches the kit (dev phase
   2026-11-17T06:03Z → 2026-11-17T12:03Z, i.e. entries close
   Tue 17 Nov 19:03 WIB).
2. The kit's `competition.yaml` carries no `warmup_bars` in the dev
   phase; the bot still honours `state.warming_up` if the live hello
   sends one.
3. The fallback burst in `main.py` is measured over 30 minutes
   (FALLBACK_BARS = 30), matching the previous race's bot; the search
   simulator's fallback used the 60-minute lookback instead. The
   fallback never fired on either known window in either version, so
   this changes no measured number — it only matters in a window with
   no dip and no filled buy by bar 1200.
