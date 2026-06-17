# HANDOFF — Amba AXI / AMBA Interface Explorer

Living plan + architecture record. Per `CLAUDE.md`, every new feature is restated here
first, then its sections are checked off as they're implemented. Always read this before
starting work.

---

## Current architecture

Static, self-contained site served from `public/`. No build step — each page is a
Design Component (`.dc.html`) that loads the `support.js` runtime via a relative
`<script src="./support.js">` and renders inline-styled markup. Pages link to each
other with plain relative `<a href>`s.

### Files (all in `public/`)
| File | Role |
|------|------|
| `index.html` | Landing page — AMBA family overview, cards linking to each explorer |
| `AXI-Explorer.dc.html` | AXI4 signal-level interactive walkthrough |
| `ACE-Explorer.dc.html` | AXI Coherency Extensions — snoop channels, MOESI, coherent read |
| `LPI-Explorer.dc.html` | LPI interactive walkthrough |
| `APB-Explorer.dc.html` | APB interactive walkthrough |
| `AXI-Explorer-Directions.dc.html` | AXI visual-direction study (3 takes) |
| `support.js` | DC runtime (do not hand-edit; refresh from platform copy when it changes) |

### Navigation
- `index.html` → each explorer via `href="<Explorer>.dc.html"` (cards: AXI, ACE, LPI, APB).
- Every explorer + the directions study → back to `href="index.html"`.

### Conventions (see CLAUDE.md)
- No spaces in filenames; hyphens instead.
- `public/` is the deployable source of truth; root-level site files are stale.
- Project meta (`CLAUDE.md`, `HANDOFF.md`, `README.md`, `screenshots/`) stays at root,
  never ships in `public/`.
- Inline styles only; all references relative.
- Download zips are labelled with a semantic version (current: **v1.1.0**).

---

## Backlog / planned features

### MOESI FSM diagram + ACE5 atomics/stash  _(planned)_

**Goal.** Two additions to `public/ACE-Explorer.dc.html`:
1. **MOESI as a finite-state machine.** Add an interactive state-transition diagram to the
   Cache States section. MOESI *is* canonically an FSM — each line transitions on local CPU
   ops (read/write/evict) and remote snoops. Diagram: 5 nodes (M/O/E/S/I) in a permission
   ring, edges labelled by trigger, solid = local op / dashed = snoop. Tie selection to the
   existing `moesi` state — selecting a tile (or clicking a node) highlights that state's
   edges and lists its outgoing transitions in the detail panel.
2. **ACE5 · Atomics & Stash section** (new nav item under a new "ACE5" group). ACE5 pulls in
   the AXI5 set; cover the two biggest: hardware **atomics** (AWATOP — AtomicStore/Load/
   Swap/Compare, with a fetch-and-add timing diagram stepper) and **cache stashing**
   (WriteUniqueStash / StashOnceShared / StashOnceUnique, AWSTASH* target hints). Link the
   ACE5 version card → this section.

**Data-model.** None — extends the one explorer file. New state: `atomicStep`. New section
key `ace5`. Reuse buildWave / step / inspector / nav engine. FSM rendered via a `buildFSM`
createElement helper (same pattern as buildWave).

**Build order.**
1. [ ] Logic: `moesiEdges()`, `buildFSM()`, `atomicSteps()`; add `atomicStep` state + `ace5`
       to allowed sections; nav ACE5 group; FSM + transitions in renderVals; atomic wave +
       atomicOps + stashCards; return wiring.
2. [ ] Template: FSM panel + transitions list in States section; new ACE5 sc-if section;
       ACE5-version-card link.
3. [ ] Verify all sections render; bump version (minor) + update changelog/file notes.



### ACE — AXI Coherency Extensions  _(DONE · v1.1.0)_

Shipped `public/ACE-Explorer.dc.html` (cyan #195 accent) + landing-page card. Reuses the
AXI explorer engine (buildWave / miniDrive / dirBadge / step / nav). Sections: Overview,
Cache States (MOESI 2×2 matrix + Invalid), Snoop Channels (AC/CR/CD stepper), Coherent
Read (ReadShared snoop-hit-dirty, cache-to-cache, with CPU0/CPU1 waveform dividers),
Domains & Types, Versions (ACE / ACE5 / ACE5-Lite), Cheat Sheet. Colour rules honoured:
extended AR/AW signals inherit blue/red; new snoop channels AC=cyan(195) CR=magenta(330)
CD=indigo(275). `startSection` prop exposed as a tweak.

<details><summary>original plan</summary>

**Goal.** Add ACE as a fourth protocol in the AMBA family: a landing-page card on
`index.html` plus a new `public/ACE-Explorer.dc.html` interactive walkthrough, matching
the visual language and interaction model of the existing AXI/LPI/APB explorers.

**What ACE is (content the explorer must cover).** ACE extends AXI4 with hardware cache
coherency. It keeps the five AXI channels (AW, W, B, AR, R) and adds:
- **Three snoop channels** for the interconnect to query a cached master:
  - `AC` — snoop address (interconnect → master): `ACADDR`, `ACSNOOP`, `ACPROT`
  - `CR` — snoop response (master → interconnect): `CRRESP`
  - `CD` — snoop data (master → interconnect): `CDDATA`, `CDLAST`
- **Coherency qualifiers on AR/AW**: `ARSNOOP`/`AWSNOOP` (transaction type),
  `ARDOMAIN`/`AWDOMAIN` (shareability domain), `ARBAR`/`AWBAR` (barrier).
- **Five cache-line states (MOESI)**: Modified, Owned, Exclusive, Shared, Invalid —
  worth a small state-diagram panel.
- **Domains**: Non-shareable, Inner, Outer, System.

**Data-model changes.** None to existing files. New page is self-contained like the
others (inline styles, `./support.js`, relative back-link to `index.html`).

**UI work.**
- [ ] `index.html`: add a 4th protocol card for ACE. Pick an accent hue distinct from
      AXI blue (235) / LPI green (155) / APB amber (70) — propose **violet ~oklch(0.72 0.17 305)**
      (currently only used as AXI's R-channel chip, so confirm it doesn't read as "AXI R").
      Card shows the 3 snoop channels (AC/CR/CD) as the channel chips + a "builds on AXI"
      note. Decide status badge: `Ready` only once the explorer is built — until then use a
      `Soon` badge variant (new — none exists yet).
- [ ] Update hero rail counter ("3 interfaces · all ready" → "4 interfaces …") and the
      footer line ("AXI, LPI and APB …").
- [ ] `public/ACE-Explorer.dc.html`: new explorer. Reuse AXI explorer's header/back-link,
      waveform and channel-stepper patterns; add snoop-channel walkthrough + MOESI state
      panel + a snoop-transaction example (e.g. ReadShared causing a snoop into another master).

**Build order.**
1. [ ] Confirm scope with user: full interactive explorer now, or land the card as `Soon`
       first and build the explorer in a follow-up?
2. [ ] Add ACE card + `Soon` badge to `index.html`; update hero/footer counts.
3. [ ] Build `public/ACE-Explorer.dc.html` (header + back-link → channels → snoop flow → MOESI).
4. [ ] Flip ACE card badge to `Ready`; wire `href="ACE-Explorer.dc.html"`.
5. [ ] Update file table + nav map in this doc; bump version (minor — new feature).

**Open questions for user.**
- Build the explorer now, or ship the card as "Soon" first?
- Accent colour: violet, or another hue?
- AXI version focus — ACE is an AXI4 extension; keep examples at AXI4 level?
</details>

---

## Changelog
- **v1.1.0** — Added ACE (AXI Coherency Extensions) explorer + landing-page card;
  hero/footer interface counts updated to 4.
- **v1.0.0** — Site reorganised into `public/`; stale root copies removed; missing
  back-link added to the AXI directions study; semantic versioning + plan-file
  conventions established.
