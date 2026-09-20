# Deck contract

`assets/board-pack.html` renders a deck from one JSON object. You write the
numbers; the renderer draws every slide. **There is no slide markup to author**
— if you find yourself writing HTML for a slide, you have left the contract.

Replace the contents of the `<script id="deck" type="application/json">` block.
That block is the only edit point, and the finished file is the source of truth:
a later revision edits the JSON in place and re-prints.

```jsonc
{
  "meta": {
    "entity":   "Northwind Robotics, Inc.",   // legal name from the ledger entity
    "title":    "Board Pack",
    "period":   "Q3 FY2026 · quarter ended 31 August 2026",
    "prepared": "2026-09-19",
    "theme":    "light"                       // "light" (default) | "dark"
  },
  "slides": [ /* ordered; slide i is page i */ ]
}
```

Every slide takes `kind` plus, where the kind has a header: `eyebrow` (2–4 word
beat name — the renderer prefixes the number), `headline`, `subhead`, `source`
(the footer line: name the tools the numbers came from).

## Slide kinds

| `kind` | Use it for | Fields |
|---|---|---|
| `cover` | Page 1. Carries the attestation. | `mark`, `headline`, `subhead`, `attestation[{label,value}]` |
| `section` | A divider between parts of the pack. | `number`, `headline`, `lead` |
| `bullets` | Executive summary, risks, the asks. | `bullets[]`, `numbered`, `stats[{label,value,format}]` |
| `kpi` | The scoreboard. | `cards[{label,value,format,delta,deltaLabel,tone,note,highlight}]` |
| `statement` | A financial statement with comparatives. | `columns[]`, `rows[]`, `format`, `unit`, `footnote` |
| `table` | Any other table (controls, provenance). Same renderer as `statement`. | as `statement` |
| `chart` | A trend or a comparison. | `chartType`, `categories[]`, `series[]`, `format`, `unit`, `dp`, `highlight`, `labelSeries`, `takeaway` |
| `split` | An argument beside its evidence. | `bullets[]`, `right` (a `statement` body, or `{kind:"kpi",cards:[…]}`), `footnote` |
| `callout` | One number that is the whole point. | `label`, `value`, `tone`, `context` |

### Rows

A row is `{label, values[], indent, emphasis, highlight, tone, signed, format, unit}`.
`indent` is `1` or `2`. `emphasis` is `"subtotal"` or `"total"` (a total gets a
rule above it). `signed: true` colours positive green and negative red — use it
for a variance column, never for the statement body. `["Revenue", 14200000, 10400000]`
is shorthand for a plain row.

### Series

`{name, values[], projected}`. `values` aligns 1:1 with `categories`; use `null`
for a period with no value — that is how the actuals-to-forecast seam is drawn.
`projected: true` renders the line dashed. **Two series maximum.** The palette
has exactly two validated categorical hues because the rest of the ramp is
reserved for variance; a third series means small multiples, not a third colour.

### Formats

`format` is `usd` (`$14,200`), `usdc` (compact, `$14.2M`), `pct`, `x`, `mo`, or
`num`. `unit` scales the raw value before formatting: `ones` (default),
`thousands`, `millions` — so a statement in `$ thousands` carries real dollars in
the JSON and says `"unit": "thousands"`. `dp` overrides decimal places. Say the
unit in the `subhead`; the renderer does not.

**Put raw numbers in base currency units.** Several ledger reads return *minor
units* — `openBalanceCents` is cents. Convert once, on the way in, and never mix
the two in one slide.

### Tone

`tone` is `positive` | `negative` | `warning` | `neutral`. It colours a delta, a
highlighted row, or a callout. **Set it deliberately**: a highlighted row with no
tone renders in the brand blue, and a writedown, a missed plan and a covenant
breach all read as neutral emphasis unless you say otherwise. Status colours are
reserved for this and are never used as a series colour.

## Capacity — measured, not estimated

The renderer does not wrap, shrink or error on an overrun. These are the limits
at which the shipped geometry actually breaks, measured against the template:

| Rule | Fits | Breaks at |
|---|---|---|
| `statement` / `table` rows | **9** | 10 |
| …with a `footnote` | **8** | 9 |
| `bullets` | **8** | 9 |
| …with a `stats` rail | **6** | 7 |
| `kpi` cards | **6** | 7 — the 7th is *dropped*, not drawn |
| `chart` categories | **11** with ~9-character labels; 16+ with 3-character ones | label-length dependent |
| `split` bullets (beside a 6-row table) | **5** | 6 |
| `split` table rows (beside 4 bullets) | **9** | 10 |
| `chart` series | **2** | a 3rd is dropped |

**When something does not fit, move it — never delete it.** A row that will not
fit rolls into an "All other (n)" line with the detail moved to an appendix
table; a figure that will not fit goes in the `subhead` or the `footnote`. The
board losing a number to a layout constraint is the one outcome this contract
exists to prevent.

## A deck that already fits

`assets/example-deck.json` is a complete deck — all nine slide kinds, each sitting
**at** the capacity limits in the table above, with fictional numbers. It exists so the
template can be checked on its own, before any real figures are in it:

```bash
python3 - <<'EOF'
import re, pathlib
src = pathlib.Path('assets/board-pack.html').read_text()
deck = pathlib.Path('assets/example-deck.json').read_text()
out = re.sub(r'(<script id="deck"[^>]*>).*?(</script>)',
             lambda m: m.group(1) + "\n" + deck + "\n" + m.group(2), src, flags=re.S)
pathlib.Path('/tmp/example-board-pack.html').write_text(out)
EOF
```

Then run the gate below against `/tmp/example-board-pack.html`. It passes today: nine
slides, no overflow, nothing dropped, printing to nine pages at 1440×810.

Because it sits *at* the limits rather than comfortably inside them, it is also what
catches a geometry regression — a change that costs a row of vertical space flags here
before it reaches a real board pack. Adding a bullet to its bullets slide is enough to
make the gate fail, which is the intended behaviour, not a bug.

## The gate

The template measures itself after layout and marks every slide that overran,
that dropped a card, or whose axis labels collide. Run it before you print:

```bash
CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
"$CHROME" --headless=new --disable-gpu --virtual-time-budget=6000 \
  --dump-dom "file://$PWD/<entity>-<period>-board-pack.html" \
  | grep -o 'data-kind="[a-z]*" data-overflow="true"'
```

Empty output is the pass. Any line names a slide that is drawing over its own
footer — fix the content, not the CSS, and re-run. A flagged deck also renders a
red `OVERFLOW` flag on the page itself, so it cannot be printed by accident.

## Print

```bash
"$CHROME" --headless=new --disable-gpu --no-pdf-header-footer \
  --virtual-time-budget=8000 \
  --print-to-pdf="$PWD/<entity>-<period>-board-pack.pdf" \
  "file://$PWD/<entity>-<period>-board-pack.html"
```

One 1920×1080 page per slide. **Never overwrite a PDF that has been presented** —
a revision prints to `…-v2.pdf`, so the record matches what the board saw.

## Fonts

The template uses the system sans stack so it renders identically on a machine
that has never seen this repo. If you are producing a deck for RoboSystems' own
use and have the brand faces installed, add them to the `--font` token; nothing
else changes.
