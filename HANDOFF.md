# Barricelli's Universe — handoff

An interactive recreation of Nils Aall Barricelli's 1953–57 numerical-evolution
experiments, built as a single self-contained page.

- **Live:** barricelli.vercel.app
- **Source:** `index.html` — one file, no build step, no dependencies except a
  Google Fonts stylesheet. Deploys to any static host as-is.
- **Artifact preview:** https://claude.ai/artifact/5QQAUyEowFz4bpfefe4BmB

---

## 1. The rule (read this before changing any simulation code)

This is the part most worth getting right, because **I implemented it wrong the
first time and built a great deal on top of the mistake.** The authoritative
reference is Alexander R. Galloway's Processing reimplementation of Barricelli's
1957 paper (`cultureandcommunication.org/galloway/Barricelli`), which works from
the original norms and their regional assignment.

Every cell holds a signed integer (a *gene*) or zero (empty).

**Translation, not persistence.** Each generation is written fresh from the one
above. A gene at position `a` holding value `n` writes a copy one row down at
`a + n`, wrapped. **It does not stay where it was.** This is why lineages read as
diagonals. My original implementation had genes persisting and *deflecting* off
neighbours in a chain — an invention of mine, not Barricelli's, and the source of
several wrong conclusions I later had to retract.

**A fan, not a chain.** After placing a copy at `Na`, the rule reads `cur[Na]` —
the cell *directly above the landing site*, as it stood in the reading row. If
that cell is non-empty and the gene has copies left, it places again, stepping by
*that* value — **but starting again from the gene's original position `Xa`**, not
from where it just landed. So a gene with neighbours scatters a small fan of
offspring from one origin. "And again from there" is wrong; it was leftover
phrasing from my invented rule and survived in the prose for a while after the
engine was fixed.

**Collisions invoke a norm.** If the target already holds a *different* value, the
norm governing that column's quadrant computes a replacement. Norms don't pick a
winner — they build a new value from the local spacing. `u` is the distance
rightward from the cell directly above the collision to the nearest gene; `v` is
the distance leftward.

| Norm | Result |
|------|--------|
| A | `u+v`, kept positive if the flanking genes share a sign, negated if not. Empties if the cell above is occupied. |
| B | `u+v−1`, otherwise as A. |
| C | `v−u`. Otherwise as A. |
| D | Mirror test, the only norm that ignores the gaps *and* the occupancy check. With `p` the value directly above, compare the cells `p` steps to either side; if both hold the same value `m`, the cell becomes `2m−p`, else it empties. |
| 0 | Every collision empties the cell. |

**Regional norms.** The universe is quartered, each quarter running its own norm,
so organisms drifting across a boundary meet different rules. Presets: BBAA
(default), ADAD, AAAA, Zero.

Code: `applyNorm` (line ~743), `placeGene` (~764), `stepOnce` (~779),
`normForColumn` (~758).

---

## 2. Verified facts

Everything here was measured, not assumed. Several replaced claims I had
originally asserted on intuition and got wrong.

**Dependence is what keeps the universe alive.** Set *copies per gene* to 0 and a
universe of ~495 slides to ~65 by generation 10, ~17 by 50, and settles around 4.
A lone gene in an empty universe stays at population 1 forever — it copies itself
to exactly one new place each generation. *The notes previously claimed the
collapse happens "within a few generations." It doesn't; it's a slide over
roughly a hundred. That error was mine.*

**Every empty cell in the same gap measures the same `u+v`**, because one distance
grows exactly as fast as the other shrinks: between genes at `L` and `R`, any cell
between gives `u+v = R−L`. Verified across 1,439 gaps, zero mismatches. This is
why norms A and B paint a whole gap one flat value while C lays a gradient — the
origin of the broad colour fields in the record.

**Organisms are bimodal.** Matching components across generations by overlap:
80–91% carry their identity from one generation to the next, but **~73% of all
organisms live exactly one generation**, while the longest-lived persist 170–272
generations. This finding determined the design of the organisms view (§4).

**Norm presets differ in a stable, measurable way** (medians of 50 trials × 300
generations):

| Preset | Population | Distinct genes | Symbiont cells | Organisms | Largest organism |
|--------|-----------|----------------|----------------|-----------|------------------|
| BBAA | 374 | 10 | 116 | 13 | 43 |
| ADAD | 278 | **17** | 36 | 12 | 6 |
| AAAA | **420** | 8 | **257** | 16 | **76** |

There's a real trade-off here: ADAD sustains roughly twice the genetic diversity
but builds by far the weakest organisms; AAAA is the densest and most cooperative
but the least varied. **No single run can establish this** — symbiont counts range
from 4 to 427 on identical settings, which is the entire reason the Trials panel
exists.

**The injected seed** is `[4, 1, −1, ·, −4]`. The 4 and −4 read each other, as do
the 1 and −1: two mutual pairs, so all four genes count as symbionts. It was
previously `[5, −3, 1, −3, ·, −3, 1]`, which I had attributed to Barricelli. That
attribution was unsupported — it traces back to my own early research note, not to
any source — and worse, under the implemented rule that sequence has **zero**
mutual pairs, so "Inject organism" was injecting something the app's own ledger
classified as parasites. Its growth was also indistinguishable from random 7-cell
strips. Don't reinstate it.

---

## 3. Architecture

Single file. Inside one IIFE; no framework.

| Concern | Where | Notes |
|---|---|---|
| Simulation | ~735–811 | Pure-ish; reads settings off `state` |
| Roles & components | `analyzeRoles` ~818 | Returns `comps` (membership), counts |
| Organism tracking | `trackOrganisms` ~873 | Cross-generation identity |
| Record rendering | `paintRow` ~937, `appendGenerationRow` ~979 | Canvas scroll-blit |
| Teaching stage | `demoStep` ~1392, `renderTeach` ~1541 | SVG, separate tiny universe |
| Trials | ~1204–1290 | Headless, no history, no drawing |

**Two renderers on purpose.** The record is a canvas with a scroll-blit (many
thousands of cells, needs speed; cells are ~2px). The teaching stage is SVG (few
elements, needs crisp numerals and curves). Don't try to merge them.

**Zoom** is a multiplier over a "fit the array" baseline, with the canvas at
native resolution and CSS-sized to match. An earlier version set the canvas to
`width:100%`, which rescaled it to its container regardless of internal
resolution — so changing the cell size did literally nothing on screen.

**History** is capped at `HISTORY_CAP = 400` rows; each entry carries `values`,
`parent`, `roles`, `orgId`, `orgAge`, so redraws and the ancestry inspector work
from stored state rather than recomputation.

**The ruler is offset to start where the rows start** (`syncRulerSize`). The
quadrant strip sits above the record, and for a while the ruler began at the top
of the strip, so every label read about 15 generations high. The ruler canvas
also overhangs the rows by `RULER_PAD` (8px) at each end: a label for the first
or last row is centred about a pixel from the edge, and a canvas exactly as tall
as the rows clipped half of it. A negative bottom margin keeps the overhang from
changing the layout.

**Reset preruns exactly `VISIBLE_ROWS - 1` generations**, so the top of the record
is generation 0. This is deliberate: an earlier version pre-ran far more and you
only ever saw the settled endgame, missing the diverse transient entirely.

---

## 4. Design decisions worth preserving

**Organisms are coloured by identity, weighted by age.** My instinct was to
outline each cluster. The persistence measurement (§2) says that would have
failed: with ~73% of organisms lasting a single generation, uniform treatment
buries the signal under churn. So age *is* the encoding — each organism draws more
solidly the longer it survives. Faint speckle is churn; solid bands endure. If you
change this, re-run the persistence measurement first.

**The teaching stage narrates only what it draws.** Every column the text names is
visually marked in the same frame, using three distinct marks: amber = the origin
gene, faded green = the cell read *last* step, bright green + dotted connector =
the cell being read *now*. This took three attempts. The bug both times was state
assigned at the *end* of a step being rendered next to text describing *that*
step, so the diagram ran a beat ahead of the prose. `usedSrc` (this step's value
source) and `pendingSrc` (next step's) are split precisely to prevent this —
don't recombine them.

**The legend follows the view.** In the organisms view, hue means identity, not
value; the legend and swatches change accordingly, or they'd be quietly lying.

**Accessibility: all text meets WCAG AA (4.5:1) in both themes.** Measured on
the rendered page, not from token math — 132 HTML text elements plus every SVG
label and gene-cell numeral. In light mode this required darkening
`--ink-faint`, `--spark` and `--highlighter` (hue kept, each solved to just clear
its target). Amber *text* needs more contrast than amber *graphics*, so it has
its own token, `--hl-text`; use it for any amber text you add, and keep
`--highlighter` for strokes, markers and the focus ring (3:1). If you retune the
palette, re-run the audit rather than eyeballing it: the intro paragraph looked
fine and measured 2.6:1.

**Gene-cell numerals get their own fill (`teachFill`).** The record's gene colors
are mid-tones, and a mid-tone can't reach 4.5:1 against *any* text color — even
picking the better of dark or light per cell leaves 14 of 48 failing. So the
teaching stage nudges each fill's lightness until its numeral clears 4.5:1,
keeping the hue. The same value can therefore look a shade deeper there than in
the record. That's deliberate; don't "fix" it by making the two share
`colorForValue`.

**Hover tooltips exist only where hovering does** (`@media (hover:hover)`). On a
touchscreen a tap is also a hover, and it sticks — so a tooltip on an action
button popped up over the record every time the button was used. Touch users get
ⓘ buttons that open the same text in a dialog; that text is read from the
tooltips at runtime, so the two can't drift. Informational terms (ledger labels,
norm names) open on tap and close on a second tap.

**Keyboard and screen-reader support, invisible to everyone else.** The record
is a tab stop (`role="application"`); arrow keys select a gene and open the same
ancestry inspector a click does, always stepping between genes so they never land
on an empty cell, with Page Up/Down for ten generations and Escape to clear. Its
focus ring appears only for keyboard focus (`:focus-visible`). Status that
changes without moving focus — the inspector's contents, trial progress and
results, extinction — goes through one polite live region (`announce()`), and
the teaching narration is itself a live region. Escape dismisses any open
tooltip, and invisible bridges let the pointer cross the gap onto a tooltip
without it closing (WCAG 1.4.13). The sparkline's `aria-label` is rewritten with
its range each time it redraws. Only Tab can focus the record: a `mousedown`
handler stops mouse and touch from focusing it and blurs whatever had focus, which
is exactly what a click did before the record was focusable. (An earlier attempt
let clicks focus it and then ignored the arrows, which quietly stopped them from
scrolling the page — don't reintroduce that.) Mouse and touch users see no change.

**Both horizontal scrollers use a drawn scrollbar** (`attachScrollbar`), with
the native one hidden. Native bars on macOS and iOS are overlays that stay
invisible until you're already scrolling, and iOS ignores scrollbar styling
entirely — so CSS alone could not make these discoverable on phones, where the
teaching diagram *always* scrolls.

**Palette uses a coprime hue stride** (`×7` over a 24-entry table) so adjacent
*values* get distant *hues*. Sequential hues made the record look monotone.

---

## 5. Lessons from the mistakes

Recorded because they were expensive and are easy to repeat.

1. **Read the primary source before building.** I implemented an invented rule and
   built visualisation, analysis and prose on top of it. Rewriting the engine
   invalidated a structural claim I'd made confidently ("density and liveliness
   are mutually exclusive") — an artifact of my rule, not Barricelli's.
2. **Quantitative proxies can actively mislead.** A `displace` variant scored best
   on every metric I had — density, churn, never froze — and looked like TV
   static. My metrics had no term for spatial coherence. Only looking caught it.
3. **Verify against elements, not source text.** I once "confirmed" a connector
   was present by testing `svg.innerHTML.includes('tg-link')`. It always matched,
   because the class is *defined* in the inline `<style>`. Querying
   `svg.querySelectorAll('line.tg-link')` showed the real count was 0.
4. **When a harness disagrees with the app, suspect the harness.** My first
   organism-persistence script reported 2.7 organisms/generation against the
   ledger's 13. I nearly designed around it. The script was wrong.
5. **Write the invariant as an assertion.** The narration/visual sync was only
   settled by asserting *every column named in the text is marked in that frame*
   and running it over 60 consecutive steps. That caught a second bug I'd already
   claimed was fixed.

---

## 6. Deployment notes

`index.html` is a **complete standalone document** — doctype, `lang`, UTF-8,
viewport, `color-scheme`, body reset. This matters: the file was previously a
*fragment* that relied on the artifact host injecting all of that. Served raw, it
would have rendered in quirks mode and, most visibly, **at 980px zoomed out on
phones** for want of a viewport meta.

If you publish it back to a Claude Artifact, the host prepends its own skeleton,
so the document nests. Harmless in practice — the parser collapses it to a single
`html`/`head`/`body` and everything works — but the fonts `<link>` ends up in
body on that copy, so expect a possible brief flash of fallback font there. The
Vercel copy is unaffected.

Verified at 375px: no horizontal overflow. The only element wider than the
viewport is the teaching-stage SVG, which sits in its own `overflow-x:auto`
scroller by design.

---

## 7. Open items

- **Organisms view is explained only by its button tooltip**, by choice — the
  Field Notes were deliberately left alone.
- **Trials compares a setting against itself.** Comparing presets side by side
  currently means running it three times and reading the numbers off. A
  small-multiples view would make the §2 table visible in the app.
- **Run-to-run variance is large.** Reach for Trials before concluding anything
  from a single universe — including when evaluating a change you just made.
- **The teaching diagram's text is small on phones.** The SVG keeps a 620px
  minimum width and scrolls, so on a 375px screen everything in it is drawn at
  about 70%. Raising the minimum enlarges the text but means more scrolling to
  follow the arcs; 620 was left as the compromise.
- **`injectOrganism` writes every element of its pattern, zeros included**, so
  zeros clear live cells. Keep injected patterns free of trailing zeros.

---

*Built with Claude Code over several sessions. Rules follow Galloway's reading of
Barricelli's 1957 paper; see also Barricelli, N. A., "Numerical Testing of
Evolution Theories," Acta Biotheoretica 16 (1962/1963).*
