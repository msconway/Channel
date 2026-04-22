# Wick Geometry Channel – Outside-In Specification

This document consolidates the outside-in design for **Wick Geometry Channel v1 / WickChannel v1**. It replaces prior chat logs with a single, coherent specification that combines:

- The exact patch instructions (originally in “Chat 2”).
- The clean handoff spec, integrated design brief, and design principles (originally in “chat1”).

Use this as the canonical reference for any future implementation or AI coding session.

---

## 1. Design Principles

### 1.1 Purpose

- Discover tradable price channels from an **outside-in** perspective.
- Use wick geometry and session anchoring rather than generic regression as the primary structure.

### 1.2 Core ideas

- Same-side pivot pairs define channel slope.
- Opposite-side pivots validate the family; they do not redefine slope.
- Outer rails propose the channel; inner structure and midline refine it.
- Midline is inferred from internal density, not assumed.
- History is bounded (P1 or session start to now) and scanning is done at controlled times.

### 1.3 Inputs and scaffolding to preserve

- Session logic and manual anchor (P1) selection.
- Pivot-based touch detection with safe history bounds.
- `lastValidBest` fallback state so a previously good channel persists.
- One-pass scan per bar (typically at `barstate.islast`) with a single redraw.
- Existing drawing lifecycle: clear then draw rails, midline, and zones.
- Existing inputs, unless a new input is explicitly required (e.g., Shadow channels).

### 1.4 Outside-in construction

- Build arrays of same-side wick pivots between P1 and candidate end `xCur`.
- Compute slopes for all high–high and low–low pairs.
- Bin slopes within `wickRoundTicks * syminfo.mintick` tolerance.
- Winning slope is the bin with the most votes.
- Intercept is projected from the first pivot in the winning family back to P1.
- Use this slope/intercept as the structural midline for further evaluation.

### 1.5 Rail and width selection

- For each candidate, compute pivot offsets to the midline and bin them separately for highs and lows.
- On each side, identify:
  - Outermost qualifying bin: max offset with count ≥ 2.
  - Densest bin: highest count (inner rail).
- Channel width is defined by the outermost qualifying offsets (outside-in).
- Inner rails represent dense structure and are used for scoring and midline refinement.

### 1.6 Orphan structure

- Any offset family with count ≥ 3 that does not win as the outer rail becomes an orphan rail.
- Orphan offsets and counts are stored in persistent arrays:
  - `var float[] orphanOffsets`
  - `var int[] orphanCounts`
- Orphan rails draw as thin, dotted orange lines around the channel midline.
- Orphans must persist across bars until explicitly invalidated by the lifecycle.

### 1.7 Scoring

- Structural components:
  - `expandHits`: touches on expanded outer rails.
  - `railHits`: touches on primary rails.
  - `midTouches`: touches on the midline region.
  - `len`: length of the channel in bars.
- Inside containment:
  - `fullInsidePct`: percent of bars staying inside the channel during its life.
- Score:
  - `insideFactor = fullInsidePct / 100.0`
  - `score = (expandHits * 30 + railHits * 20 + midTouches * 10 + len * 0.05) * insideFactor`
- `minInsidePct` is a hard filter; candidates below it are rejected before scoring.
- No ATR filters; spikes are discouraged by poor containment.

### 1.8 Failure modes to avoid

- Channels failing to draw while orphan lines appear.
- Orphan rails being cleared unintentionally between bars.
- Winner logic that never promotes the first valid slope or width family.
- Rejecting structurally valid channels because only one outer rail side exists.

### 1.9 Guardrails

- Ensure slope, intercept, and width are all valid before accepting a candidate.
- Allow a newly created valid slope/offset family to win immediately.
- When only one side has a strong outer rail, mirror its width to the other side as a fallback.
- Preserve existing session, anchor, fallback, and redraw behaviors—only the geometry engine changes.

### 1.10 Implementation order (for future work)

1. Replace regression slope with same-side pivot-pair slope voting.
2. Change rail selection to outermost qualifying offsets with inner rails for density.
3. Add and persist orphan rail arrays; draw them concurrently with the main channel.
4. Apply the multiplicative score formula and keep `minInsidePct` as a floor.
5. Test specifically for: no channels, only orphans, disappearing orphans, and overly rare channels.

---

## 2. Patch Specification (Exact Changes)

This section is the cleaned version of the original patch recipe.

### 2.1 Patch 1 — Replace regression slope with extreme pivot-pair slope

Remove the `sumX/sumY/sumXY/sumXX` linear regression block inside the `for xCur` loop. Replace it with this logic:

- Collect all `pivotHigh` bars within `p1x` to `xCur` into a `highPivots` array (bar index + wick high price).
- Collect all `pivotLow` bars into a `lowPivots` array (bar index + wick low price).
- For each same-side pair (high-to-high, low-to-low), compute:

  `slope = (price2 - price1) / (x2 - x1)`

- Bin similar slopes within `wickRoundTicks * syminfo.mintick` tolerance — slopes that cluster together vote as one family.
- The winning slope is the bin with the most votes.
- `intercept` = price of the first pivot in the winning pair projected back to `p1x`.

### 2.2 Patch 2 — Outer rail from extreme offset, not most-common bin

In the existing `highBins`/`lowBins` offset histogram logic, change how `curBestWidth` is selected:

- Currently: winner = bin with highest `cnt` (most common offset).
- Change to: winner = the outermost bin with `cnt >= 2` on each side:
  - highest `highBins` value with 2+ votes,
  - highest `lowBins` value with 2+ votes.
- The inner-most bin with the most votes becomes the RH/RL inner rail (used for scoring but not for channel width).
- `curBestWidth` = the outer extreme, not the dense inner cluster.

Add orphan storage:

- Any bin with `cnt >= 3` that is **not** the winning outer rail gets stored in:
  - `var float[] orphanOffsets`
  - `var int[] orphanCounts`
- These arrays persist across bars.

### 2.3 Patch 3 — Draw orphan rails concurrently

In `f_draw_channel`, after drawing `topLine`/`botLine`/`midLine`, add:

- Loop over `orphanOffsets`. For each entry with `count >= 3`, draw a `line.new` at `midY ± offset` using:
  - `color.new(color.orange, 0)`
  - `width = 1`
  - `style = line.style_dotted`
- Push these lines into `drawnLines` so they get cleared and redrawn each confirmed bar.
- These are secondary to the winning channel — thinner, dotted, orange — but always visible as confirmed structure.

### 2.4 Score change

Replace:

```pinescript
score = curExpandHits * 30 + curBestRailHits * 20 + midTouches * 10 + fullInsidePct * 0.5 + curLen * 0.05
```

With:

```pinescript
insideFactor = fullInsidePct / 100.0
score = (curExpandHits * 30 + curBestRailHits * 20 + midTouches * 10 + curLen * 0.05) * insideFactor
```

Additional requirements:

- Keep `minInsidePct` as a hard floor — candidates below it are still rejected before scoring.
- No other changes: preserve all inputs, session logic, drawing functions, debug labels, `lastValidBest*` fallback, and `barstate.isconfirmed` gate exactly as they are.

---

## 3. Operational Scaffold to Preserve

This summarizes what the old code and the “History Fixed” master already do correctly and should not be rewritten away.

### 3.1 History and scanning

- Scan window runs from P1 (manual anchor or session start) to an end bar `xEnd`.
- Safety caps (e.g., ~5000 bars) prevent pathological deep scans.
- Heavy work runs at controlled times (typically `barstate.islast`), with one redraw per confirmed bar.

### 3.2 Pivot detection

- Uses a configurable pivot length (`pivotLen`).
- Implements `f_is_pivot_high_off` / `f_is_pivot_low_off` with `safeHist` bounds to avoid out-of-range history.
- Touch detection is pivot-based on wick highs/lows.

### 3.3 Candidate evaluation

For each candidate end `xCur` from P1 to `scanEnd`, the engine:

- Builds a trend definition (currently regression slope/intercept; to be replaced by Patch 1).
- Computes midline values for each bar between P1 and `xCur`.
- Computes pivot offsets to that midline and bins them into high/low histograms.
- Counts:
  - `curBestRailHits`
  - `curExpandHits`
  - `midTouches`
  - inside bars and `fullInsidePct`
  - length `curLen`
- Calculates a score and selects the best candidate.

### 3.4 Fallback and state

- Maintains `lastValidBest` so a previous good channel persists when no better candidate is found.
- Displays debug labels summarizing:
  - best score,
  - hits,
  - inside percentage,
  - current length,
  - and other diagnostics.

### 3.5 Drawing lifecycle

- Stores all lines in `drawnLines`.
- On each confirmed bar:
  - Deletes all old lines.
  - Calls `f_draw_channel` with the chosen candidate.
- Draws:
  - midline,
  - upper and lower rails,
  - optional parallels from inner/outer offset families,
  - optional expanded zone via `linefill.new`.

Outside-in replaces the *geometry engine* (slope, rails, orphans, scoring) but not this scaffold.

---

## 4. Failure Modes and Guardrails

These arose from previous debugging and should shape any new implementation.

### 4.1 Orphan persistence

- **Bug**: orphan arrays were cleared on redraw or winner replacement, causing orphan lines to disappear.
- **Rule**: orphan rail state must persist across bars until explicitly invalidated by your lifecycle. Redraw should not wipe them unintentionally.

### 4.2 Winner selection

- **Bug**: strict `>` comparisons and poor initialization for slope/width winners meant the first valid family might never be selected; slope or width could stay `na`.
- **Rule**:
  - A newly created, valid slope/offset family must be eligible to win immediately.
  - Use `>=` or explicit initialization logic to avoid “no winner” states.

### 4.3 One-sided channels

- **Bug**: after switching to outermost rails, sometimes only the upper or lower side had enough votes; the code then rejected the candidate because it expected both sides.
- **Rule**:
  - When only one side has a qualifying outer rail, mirror its width to the other side as a fallback.
  - Preserve outside-in intent but avoid over-rejection of clearly structured channels.

### 4.4 “Only orphans draw” mode

- **Bug**: main channels sometimes never appeared; orphan rails accumulated and drew, indicating partial logic success but failed promotion of the main winner.
- **Rule**:
  - Ensure slope, intercept, and width all resolve together for the main winner before downstream filters.
  - If they don’t, skip the candidate early rather than half-instantiating structure.

---

## 5. Acceptance Criteria and Tests

A correct outside-in implementation should satisfy:

- Channel slope comes from same-side pivot-pair voting, not regression.
- Opposite-side pivots act as validation points only.
- Width is defined by the outermost qualifying offsets with `cnt ≥ 2`.
- Dense inner offsets remain available as inner rails / support structure.
- Non-winning strong offset families persist as orphan rails and draw as dotted orange structure.
- Scoring uses the multiplicative formula with `insideFactor`, and `minInsidePct` remains a hard floor.
- Existing session, anchor, fallback, debug, and redraw behaviors remain intact.
- Channel frequency is reasonable:
  - no “only orphans draw” behavior,
  - valid one-sided channels are not rejected purely for asymmetry.

Recommended tests:

- No main channels drawing (indicates slope/width winner issues).
- Only orphan rails drawing.
- Orphans disappearing unexpectedly between bars.
- Channels appearing extremely rarely even on clearly trending charts.
- One-sided valid channels being rejected.

---

## 6. Handoff Summary (for new AI sessions)

If you need to brief a new AI quickly, this paragraph is enough context:

> “I have a Pine Script indicator called Wick Geometry Channel v1 / WickChannel v1. I need you to apply the outside-in patch to my master file and save it. The design rules are: Same-side pivot pairs define candidate trendlines first — angle comes from the defining rail, not from regression or a midline. Opposite-side pivots are independent validation points, not assumed to be their own trendline. Symmetry is the fallback if no strong opposite parallel exists. Midline is tertiary — discovered from internal pivot density, not a structural source. Outer rail = outermost offset bin with 2+ votes. Inner rails and midline = densest offset bin. Any 3 pivots that line up = orphan line, drawn dotted orange, persists across bars. Score = (expandHits×30 + railHits×20 + midTouches×10 + len×0.05) × insideFactor. No ATR filtering — spikes score lower naturally. Shadow channels setting 0–3 added as input, activation phase 2. Preserve all session logic, manual anchor, debug labels, lastValidBest fallback exactly.”

---

## 7. Final instruction to a new coding model

Use the latest working master file as the base. Apply only the explicit structural changes described above. Do **not** simplify away the existing anchor/session/fallback framework. The goal is not a generic channel indicator; it is this specific Wick Geometry Channel architecture converted to an outside-in model while preserving its proven operational scaffolding.

### 1.3 Patch theme priority

When deciding what to fix or implement next, treat these themes in priority order:

1. Outside-in geometry  
   - Outer rails propose the channel first.  
   - Inner rails and midline organize and refine what the outer rails already define.

2. Orphan rails  
   - Clean up half-dead channels where one rail has lost its partner.  
   - Preserve valid parallel structure (orphan offset families) as dotted orange lines.

3. Best / New semantics  
   - “New” = discovered or updated in the current scan.  
   - “Best” = current selected winner after scoring and validation.  
   - New must not overwrite Best unless it clearly wins under the rules.

4. Shadows support  
   - Use a consistent definition of candle shadows (upper/lower wicks) across bullish and bearish bars.  
   - Decide whether rails use full wick extremes or shadow-based filters, and keep that behavior explicit.

5. Validation & invalidation  
   - After any rail move or winner change, re-validate the channel.  
   - Reject channels that no longer satisfy width, pairing, touch, or continuity constraints.

### 1.6 Orphan structure

There are two distinct “orphan” concepts in this design:

1. Orphan parallel families (geometry concept)  
   - Any offset family with count ≥ 3 that does not win as the main outer rail.  
   - Stored in persistent arrays:
     - `var float[] orphanOffsets`
     - `var int[] orphanCounts`  
   - Drawn as thin, dotted orange lines around the channel midline.  
   - Must persist across bars until explicitly invalidated by the lifecycle.

2. Orphan channel rails (cleanup concept)  
   - A half-dead channel where a rail’s paired partner is missing, invalid, or no longer consistent.  
   - These rails must not keep acting as a full channel.  
   - On each major update (channel rebuild, promotion, invalidation, or merge), detect unpaired rails and either:
     - delete them, or
     - demote them so they no longer participate in Best/New selection or scoring.
    
### 4.5 Best / New semantics

- New = any candidate channel discovered or updated on the current scan.  
- Best = the current selected winner after scoring and validation (often aligned with `lastValidBest`).  
- Guardrails:
  - A New candidate must not overwrite Best unless its score and validity clearly exceed Best under the defined rules.  
  - When a New candidate fails validation after promotion, revert cleanly to the previous Best or to a neutral “no channel” state.  
  - State transitions must be deterministic: the same bar sequence should always produce the same Best/New history.
 
