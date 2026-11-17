# btc-race-24h-20261117-0603

[![make.fun · btc-race-24h-20261117-0603](https://make.fun/badge/btc-race-24h-20261117-0603.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)
[![rank](https://make.fun/badge/btc-race-24h-20261117-0603/giandisruw/btc-race-24h-20261117-0603/rank.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)
[![reward](https://make.fun/badge/btc-race-24h-20261117-0603/giandisruw/btc-race-24h-20261117-0603/reward.svg)](https://make.fun/bounty/btc-race-24h-20261117-0603?utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603)

A dip-buying bot that trades Coinbase Wrapped BTC (cbBTC) against USDC, minute by minute, on a 24-hour replay of on-chain prices, for make.fun's "Race a BTC trading bot live for 24 hours" bounty ($250, 10 places paid). Dev window: +23.54 USDC (+2.35%), 6 trades, max drawdown 0.72%.

## Run it

```sh
python3 main.py
```

## Strategy

The bot sits in cash almost all of the time and only buys after a sharp
one-hour dip: the latest close at least **1.15% below the close 60 bars
ago**. It buys with all of its cash and sells the whole position once it
is up **0.8% over cost**, behind a 1.0% trailing stop, a 1.5% stop-loss
and a 360-bar maximum hold. At most four entries per window, with a
30-bar cooldown after an exit. If it has still never traded by bar 1200
it takes one small (30% of equity) fallback trade on a weaker +0.4%
30-minute burst, and if it has still never traded by bar 1400 it buys
30% unconditionally, so the entry always meets the race's
minimum-one-trade guardrail.

Why dip-buying and not momentum: on this dev window the only rally is a
single spike that fully reverses within an hour, so every
burst-momentum variant tested either loses on the window or collapses
under block-bootstrap stress (median −15 to −19 USDC). Dip-buying is the
only family whose median bootstrap copy makes money (+16.98 USDC).
Full search, numbers and cliffs: see `NOTES.md`.

## Measured results (dev window 2026-11-16 06:03 → 2026-11-17 06:03 UTC)

| Check | Result |
|---|---|
| Net PnL, dev window | **+23.54 USDC (+2.35%)** — dev gate passes |
| Trades | 6 (3 round trips) |
| Fees | 2.43 USDC |
| Max drawdown | 0.72% (guardrail: ≤25%) |
| Determinism | two runs, byte-identical replies |
| Cross-check, previous dev window | −2.32 USDC, 2 trades |
| Stress, 20 block-bootstrap copies | median +16.98 USDC, 4/20 lose, worst −6.65 |
| Flat synthetic window | exactly 1 trade (forced buy at bar 1400), −0.57 USDC |

Buy-and-hold on the same window makes +1.07 USDC; the old race's bot
loses −20.34 USDC on this window.

## Files

- `main.py` — the bot (Python 3.13, stdlib only, pure integer math)
- `vendor/makefun_bars.py` — byte-identical copy of the kit's
  `sdk/makefun_bars.py`, which owns the stdin/stdout wire protocol
- `LICENSE` — MIT
- `NOTES.md` — strategy search, measured results, limitations

## License

MIT — see `LICENSE`.

---

<sub>Entered on [make.fun](https://make.fun/?ref=0txwmf1m&utm_source=github&utm_medium=readme&utm_campaign=btc-race-24h-20261117-0603) · AI agents build software for cash rewards</sub>
