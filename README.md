[README (1).md](https://github.com/user-attachments/files/33056621/README.1.md)
# Internal to External Liquidity Delivery

A TradingView indicator for delivery from internal range liquidity to external
range liquidity. Pine Script v6, built on ICT concepts.

```
dealing range  →  internal structural execution  →  external liquidity delivery
```

The CISD, the retracement band and the imbalance zones are the internal
execution structure. The opposing dealing-range extreme is the external draw on
liquidity.

Underneath that sits one testable claim, and it is the only part here you will
not find in a dozen other scripts: **whichever extreme of the range forms first
tells you which extreme price is drawn toward afterwards.** Everything else
either executes that idea structurally or measures whether it holds.

---

## The model

1. A dealing range forms over a user-defined window. Its high and low are the
   external liquidity; everything the model does between them is internal.
2. The extreme that printed **first** sets the direction of the expected
   delivery.
   - low first → the high becomes the draw
   - high first → the low becomes the draw
3. A fib retracement is drawn across the range. The extreme that formed first
   carries `1` and the later extreme carries `0`, so the ratios read from the
   objective back into the origin.
4. A CISD is located from swing structure inside the range.
5. A retest of that CISD level is the trigger.
6. Touching the far extreme completes delivery and ends the session.

The far extreme is a draw on liquidity, not a breakout target. A touch fills it.
Nothing has to break.

---

## Structural detection

Swing pivots are 2 left / 2 right. No ATR bands, no point thresholds, no
leg-length windows anywhere in the detection.

For a short setup:

- **run** — the up-closing candles between a swing low and the following swing high
- **anchor** — the last candle of that run, the one nearest the swing high
- **level** — that candle's open
- **confirmation** — a later bearish close beneath the level

Mirrored for longs.

Three constraints decide which block becomes the session's CISD:

| Rule | Effect |
|---|---|
| **EQ** | A short's level may never form below range EQ, a long's never above it. A level in discount for a short is the same stop distance for a fraction of the run. |
| **Range** | Both the level and the swing it was built from must sit inside the range. A block from an older, higher leg is not part of the delivery that made the range. |
| **Extreme** | Among all blocks a bar confirms, the most extreme level wins — the swing the range was delivered from, not whichever is nearest. |

**One trade attempt per range.** Before an entry, a structural break may retire a
CISD and a replacement may be hunted. Once an entry fires the model is
committed: no second CISD is sought and no second entry taken.

---

## Signals are never erased

A trigger that printed on a confirmed bar stays on the chart. If it later
invalidates — by the two-engulf rule or by structural failure — the arrow is
muted and an `×` is added at the exit bar. A CISD line that dies is terminated
and faded at that bar rather than deleted, and the same happens once delivery
completes.

Deleting an invalidated signal leaves history showing something no trader ever
saw, and makes the outcome statistics impossible to check against the chart.

---

## Retaining a range past its close

A dealing range does not stop mattering because the clock started a new one. If
yesterday's range is still the reference price is working inside, the scheduled
open would otherwise discard it mid-thesis — and a tap of its extreme an hour
later would land on a range the script had already thrown away.

Two controls, inline with their session rows:

- **`-1`** — suppress the scheduled current range and keep the previous one live,
  levels and state intact. A tap of its extreme still registers as delivery.
- **`+1`** — with `-1` on, carry that range's tracking window through one
  additional cycle so it can still deliver after the scheduled open has passed.

Both are latest-cycle controls, not historical mode switches. Enabling `-1`
retains one range and leaves every past session exactly as it was, which is what
makes it safe to toggle while a trade is live. Delivery or a structural
invalidation before the scheduled open cancels the retention and the new range
forms normally; after the open it stops the extension without rebuilding a
partial range mid-cycle.

Two things make it behave predictably:

**Recorded opens, not calendar dates.** Every range open is stored as it
happens, and `-1` reads back from that record. A weekend, a holiday or a half
day cannot be pointed at — if no range formed, nothing was recorded. The window
closes on the second observed open rather than a predicted date, and setting
`-1` before the day's range has opened works the same as setting it after.

**Midnight-crossing ranges target the cycle in progress.** A window that crosses
midnight belongs to the day it *started*. On the default `1800-0800`, `-1` set
at 03:00 retains the range that opened last evening — not tonight's, which has
not begun.

---

## Requirements

- **Standard candles only.** Every level here is a candle open. On Heikin Ashi,
  Renko or any other synthetic series those opens are computed rather than
  traded, so every CISD would sit at a price nobody transacted at.
- **Timeframe:** 1m to 15m. Above that the CISD is the open of a very coarse
  candle and the trigger a close of the same size, which is a different
  proposition rather than a coarser view of the same one.
- **Bar count:** the range must contain at least 6 bars. A 2/2 pivot needs five
  bars and confirms on the fifth, so below six the range cannot host a swing of
  its own.
- **Divisibility:** the range length must divide evenly by the chart timeframe,
  or its boundary falls mid-bar.
- **Alignment:** the tracking window must start exactly when the range ends, and
  the two must not overlap. Checked against the raw windows, so a
  misconfiguration is still caught while a range is being retained.
- **Length:** the range may not span a full 24 hours — that leaves nothing for
  the tracking window.

All are enforced with `runtime.error`, not warned about. A late tracking start
would leave a blind window in which a target or stop could occur unseen.

**Windows may cross midnight.** The shipped default is `1800-0800` with tracking
`0800-1700`. Session arithmetic wraps, and the daily housekeeping runs when the
tracking window closes rather than at midnight, so it can never fall inside a
range that straddles the date boundary.

When both extremes land on one chart bar, formation order is resolved from the
1-minute series before the session is given up as ambiguous.

---

## Running it on a perpetual or CFD

A futures chart simply has no bars when CME is shut. A perpetual or CFD prints
straight through, so without a clock the script would build ranges across the
weekend and feed them into the formation and setup statistics — which are the
entire research output.

`Hold perpetual futures to the CME clock` is the first input in the list, on by
default. It ignores bars from Friday 17:00 to Sunday 18:00 New York and in the
daily 17:00–18:00 halt, exactly as the futures contract does. **A futures chart
is never affected either way.** Turn it off on a genuine 24/7 market with no CME
session to mirror, such as a crypto perpetual.

The windows are **held** through closed periods, not gated by them. That
distinction matters: ANDing each window with a market-open flag makes the daily
halt read as *outside the window*, so any range spanning it closes at 17:00 and
reopens at 18:00 as a new range. Holding reproduces what a futures chart does,
where those bars do not exist at all.

The test is the symbol type, so it covers any non-futures feed, not only
perpetuals. That is deliberate: an explicit ticker list that misses a symbol
fails **silently** — weekend sessions feed the statistics with nothing on screen
to say so. Clocking every non-futures feed fails loudly, and a genuine 24/7
market is one unticked box away.

**Known cost.** On a clocked feed, a swing within two bars of the halt or the
weekend can be missed, because its confirmation bar is skipped.

### Why there is no spread buffer

An earlier version carried one, reasoning that a bid-based chart shows a short's
stop and target better than they would fill. It was removed: these venues
trigger stops and targets on their own last traded price, which is what the
perpetual's own chart plots, so touch-based scoring already matches the account.

The requirement that follows is simple — **chart the perpetual itself, not the
futures contract it mirrors.**

---

## The dashboard

### Execution view

```
INTERNAL TO EXTERNAL LIQUIDITY DELIVERY

Formation      HIGH FIRST
Delivery       DR LOW        29424.00
Historical     BREAK LOW     80.0% · n=10 · AGREES

Setup          TRIGGERED
CISD           29576.00      PREMIUM · IN BAND
Confluence     BEAR FVG      ALIGNED
```

Four questions and nothing else: what is the thesis, where is the objective,
does history agree, is the setup valid. The panel ships switched off — turn it
on when you want the narrative rather than just the levels.

- **Formation** is coloured by what it implies, not the extreme it names. High
  first delivers down, so it wears the bearish accent.
- **Historical** always prints its denominator. A rate without its sample size is
  the easiest number on a panel to over-trust, and formation order is this
  model's only hypothesis. `AGREES` means the modal historical outcome points
  the same way — not that it confirms anything.
- **Setup** runs `WAITING → FORMING → HUNTING → ARMED → TRIGGERED`, then
  `DELIVERED / MISSED / STOPPED / EXITED / AMBIGUOUS`.
- **CISD** gives the level, its side of EQ, and whether it sits in the
  retracement band. When no level has printed it says why: `NO PENDING BLOCKS`,
  `BEST OUTSIDE RANGE`, `BEST BLOCKED BY EQ`, or `AWAITING CLOSE THROUGH`.
- **Confluence** is directional. It reports the polarity of the zone actually
  inside the band and whether it delivers with the thesis.

### Detailed statistics (optional)

Two research blocks, split by formation.

**Range outcome** — which boundary broke first. A tendency of the range.

**Setup outcome** — whether the tradeable sequence paid:

```
sessions → CISD formed → triggered → target / invalidated / no delivery
```

Different populations, both denominators labelled. A range tendency and a setup
win rate are not the same statistic, and conflating them is the easiest way to
make a model look better than it is. Post-entry exits are reported by mechanism,
`engulf · struct · ambiguous`, because the two-engulf rule was a deliberate
choice and judging it means knowing how often it closed a trade that structure
had not yet invalidated.

Also included: a **rule vs baseline** comparison. The naive baseline predicts
whichever boundary price was nearest when the range locked. If formation order
were only a proxy for that, the two would score the same. A lock exactly at EQ
is `TIED` and excluded from the baseline rather than assigned a side.

---

## Statistical treatment

- Percentages are withheld below **20 observations**; the raw count prints
  instead. The Historical row is the deliberate exception and shows both.
- **Ambiguous** sessions — target and stop touched on the same bar — are excluded
  from the outcome rates *and their denominator*. OHLC cannot order two touches
  inside one bar, and scoring them either way introduces a bias.
- **Fills and breaks are separate concepts.** Reaching a price is a touch and
  always inclusive. `Break measure` governs only the range-outcome statistic:
  did price trade *beyond* the boundary.
- Changing any input recalculates the script, so counters reset by themselves
  when the range length changes.

There is deliberately **no R:R or expectancy display**. The trigger is a
confirmed close *through* the CISD level, so any figure anchored on entering
*at* that level assumes execution the trader could not have had.

---

## Settings

The groups follow the model rather than the drawing types:

```
Market  →  Dealing Range  →  Execution  →  Imbalance Confluence
```

**Market** — the CME clock toggle. First in the list: on a perpetual it is the
first thing to check, and a weekend session quietly feeding the statistics is
the costliest thing here to get wrong.

**Dealing range** — range window with `-1`, tracking window with `+1`, break
measure, mark target bar, sessions drawn.

**Dealing range levels** — DR high, DR low, show values, EQ, show EQ value.
Where the range is.

**Execution** — CISD and retest arrows. How it is traded. Separate from the
levels because they are a different class of object: one describes structure,
the other describes a decision.

**Range box** — shaded box over the forming range.

**Fib levels** — eight configurable ratios plus the retracement band, which is
scored as confluence and gates nothing.

**Imbalance zones** — ordered as: do I want them, what qualifies, how do they
behave, how do they look. Detected on the chart timeframe, so a 5m chart gives
5m gaps.

**Dashboard** — position, size, colours, volume row toggle, detailed statistics
toggle. The panel itself is off by default.

### What is deliberately not configurable

Three things in the Imbalance zones group are fixed, for the same reason each
time: making them optional would offer a choice nobody should take.

- **The C.E. line always draws.** A zone's midpoint is the level price actually
  reacts to. It is part of what the zone *is*, not decoration on top of it.
- **Its colour is fixed** at `#424351`.
- **The four zone labels are fixed** — `+FVG`, `-FVG`, `+IFVG`, `-IFVG`. The
  dashboard's Confluence row reads those same four names, so a renamed label
  would only let the chart and the panel describe the same zone differently.

### Historical drawings

Range boxes, terminated CISD levels, retest arrows and target marks outlive
their session on purpose — they are the audit trail for the setup statistics.
`Sessions drawn` bounds how many are kept, defaulting to **1**: today only.

Without that bound they accumulate until Pine silently evicts the oldest at its
500-object ceiling, which means how much history is visible depends on how many
triggers happened to fire. Raise it to 10 or 20 to check the research panel
against the chart, then put it back. The statistics accumulate over all loaded
history either way.

### Volume

The volume row is **off by default**. It is a close-location proxy, not order
flow, it is not an ICT prerequisite and it gates nothing, so a `CONFLICTING`
reading under a valid setup asks you to weigh something with no bearing on the
decision. The calculation still runs, and the alignment flag is available in the
research view for testing whether it contributes anything.

---

## Known limitations

- **The range is time-defined, not structure-defined.** A dealing range in the
  strict sense runs swing high to swing low, whatever the clock says. This one
  takes the high and low inside a window you specify. They coincide often, which
  is why the terms get used loosely, but they are not the same object. If you
  want a structurally-derived range — a 1h/4h swing pair, PDH/PDL, a prior
  session — that is a range-source feature this does not yet have. Everything
  downstream is already source-agnostic: it reads two extremes and which came
  first, and nothing else.
- **`-1` retains exactly one cycle.** If a range needs to hold for several days,
  the clock is the wrong tool for it — a range that survives that long is being
  held by structure, which is the range-source point above.
- **Sample size.** A 1m chart holds roughly two weeks of futures history. Split
  by formation and then by trigger, the setup statistics land in single digits.
  A 5m chart holds roughly ten times that and is where the numbers start to mean
  something.
- **Long ranges stretch the entry mechanism.** The longer the range, the further
  the CISD anchors from the retracement band, because the swing that delivered
  the range extreme happened hours before the band is tested. Expect more
  sessions that call the direction correctly and never trigger. That is the
  model behaving as specified, and it shows up as a low `Triggered` rate under a
  healthy `Historical` rate — exactly the divergence the two research blocks
  exist to expose.
- **Executable entry price is not formalised.** The trigger confirms on a close
  through the level; whether the tradeable entry is that close, the next open,
  or a resting limit at the level has not been decided. Nothing displayed
  depends on it, but realised-outcome statistics would.
- **Pivot thinning on coarse bars.** Near the six-bar minimum a pivot confirms
  two bars late and the last two bars cannot host one, so the model leans hard on
  pre-session structure and the CISD it finds will not match a finer chart's.
- **Overnight pivots.** Structure is tracked continuously, so pre-session pivots
  enter the queue. Protective-swing pruning clears them on a trending open; a
  quiet overnight is where to look if a level appears somewhere odd.
- **Two-engulf exits and undelivered trades** are recorded as outcomes but not
  marked to market.

---

## Relationship to Session-Based Liquidity Reversals

Deliberately a separate script, not a mode. The reversal engine arms on a sweep
of a pre-session pool; this one arms on the formation order of the range itself.
Same structural machinery, opposite trigger, and merging them would have put a
mode switch through a script that is already trusted.

The forked regions are marked in the header with their source line ranges. The
`-1` / `+1` lifecycle and the CME clock both came across from that script; the
spread buffer did too, and was then dropped from both.

Not everything crosses, and the exceptions are worth knowing before porting a
change either way:

- SBLR gates its CISD anchors on being execution-window candles. Here the gate
  is **price, not time** — the CISD must come from the delivery that made the
  range, so its level and protective swing sit inside the range, and the anchor
  is often at or before the range open.
- SBLR's MSS → FVG entry is a different execution model. This script enters on a
  close-based retest of the CISD; its zones are confluence only.
- SBLR's R gates and its range filter apply to features this script either never
  had or removed by design.

---

## Verification

The detection chain has been replayed bar by bar against exported 1-minute MNQ
data for two sessions of opposite shape — one rallying into the open, one
selling off into it — at 1m, 2m, 3m and 5m. Several bugs in this script were
found that way and could not have been found by reading the chart.

The perpetual features are the regression guarantee: the clock is bypassed on a
futures chart, so **these two sessions on MNQ must reproduce exactly.** If
either moves, the hold logic is wrong.

| Session | Expected |
|---|---|
| 12 Aug 2026, MNQ | CISD 29,967.25 confirms 09:45; retest fires 09:49 at 29,961.25 (09:30–10:00 range) |
| 07 Jul 2026, MNQ | Level 29,593.75; the 09:30 outside bar builds its block |

If something looks wrong, export the session's OHLC and replay it. The
dashboard's CISD row is built to narrow that down: it names which of the
possible reasons produced no level.

---

## Naming

Formerly **Opening Range Liquidity Delivery**, and before that the **ORB
Breakout Model**. Both described a derivation rather than a behaviour, and both
implied a morning session the script never required. The current name claims
nothing about how the range is obtained, so it stays honest if the range source
ever becomes selectable.

---

## Files

| File | Purpose |
|---|---|
| `Internal_to_External_Liquidity_Delivery_v2_8.pine` | The indicator |
| `README.md` | This document |

The script header carries a full changelog, newest first. Versions through v27
used a single number; from v2.7 the scheme is major.minor, where 2 is the
Internal to External Liquidity Delivery generation. Earlier entries keep their
original numbers — a history that renumbers itself stops being a record.
