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
| `AXI-IssueL-Explorer.dc.html` | AXI Issue L (2025) — credit-based transport, credit-flow stepper, multi-RP interleave |
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
- Download zips are labelled with a semantic version (current: **v1.2.1**).

---

## Backlog / planned features

### AXI Issue L · Credit-Based Transport  _(DONE · v1.2.0)_

Shipped `public/AXI-IssueL-Explorer.dc.html` (teal #165 accent) + landing-page card +
a credit-flow teaser in the AXI explorer's Versions section linking to the deep-dive.
Standalone page (not a new section of AXI-Explorer, per user). Reuses the AXI engine
pattern (buildWave / step / nav / sticky topology header). Sections: Overview (K→L, no new
signals, AXI_Transport + wakeup caveat), Transport (VALID/READY vs credit comparison table),
Credit flow (grant→spend→stall→return stepper with live credit ledger + waveform),
Resource Planes (full multi-RP interleave waveform: RP0/RP1/RP2 beats on one W channel +
per-RP credit pools + AW/W-same-RP, B/R-single-RP, credit-limit rules), Versions (J/K/L),
Cheat sheet. Landing hero count 4→5; RP hues blue/orange/violet, credit=green, shared=amber.

<details><summary>original plan</summary>

**Context.** AXI Issue L (IHI 0022L, 2025) adds no new signals to the AXI signal list — the
K→L delta is a new *transport mechanism* and its flow-control model, gated by a new AXI5
property `AXI_Transport`. Default stays VALID/READY; when set, channels use credit-based
transport. So this is documented as **behaviour + property**, not as new channel chips.

**What must be covered (content):**
- **Two transport modes.** VALID/READY (reactive, per-transaction handshake) vs
  credit-based (proactive, pre-issued credits). Side-by-side comparison: throughput,
  latency, flow control, determinism, complexity, use-case fit.
- **The `AXI_Transport` property** — AXI5 only; default = VALID/READY; not in AXI3/4.
  Note the constraint: with credit-based transport enabled, `wakeup_signal` is **not**
  supported.
- **Credit model / flow-control rules.** Min 1 credit per Resource Plane (RP); max 15 per
  RP plus 15 shared credits. Receiver allocates; sender spends.
- **Resource Planes (RP).** Enable write interleaving (not allowed in AXI3). Constraints:
  AW and W of a write must share the same RP; B and R are each restricted to a single RP.
- **Shared credits.** Across multiple RPs for dynamic buffer utilisation under bursty /
  uneven traffic; AXI-L supports shared-credit compatibility.

**Recommended placement — new section group in `public/AXI-Explorer.dc.html`** (not a new
top-level protocol page — Issue L is a revision of AXI, not a sibling protocol like ACE/LPI/
APB, and the explorer already owns the AXI channels + Versions section this builds on):
- New nav group **"Issue L"** with 2 items:
  1. `transport` — **Credit vs Handshake**: the mode comparison table + `AXI_Transport`
     property callout + wakeup caveat. Reuse the existing card/table styling.
  2. `credits` — **Credits & Resource Planes**: an interactive credit-flow stepper
     (receiver grants N credits → sender issues transactions until credits exhausted →
     credits returned) built with the existing `buildWave`/`step` engine, plus an RP diagram
     showing AW+W sharing an RP and the shared-credit pool. Reuse `dirBadge`/inspector.
- Extend the **Versions** section with an Issue L entry (K→L summary) so the timeline is
  complete even for users who don't open the new group.
- Accent: introduce one new hue for credit-based transport chips (propose amber-teal
  ~oklch(0.78 0.13 195) is taken by Exclusive; pick **oklch(0.80 0.14 165)** teal-green, or
  confirm). Keep VALID/READY examples in the existing handshake blue.

**Data-model.** None — extends the one explorer file. New section keys `transport`,
`credits`; new state `creditStep` (+ maybe `rpSel`). Reuse buildWave / step / inspector /
nav engine. Credit stepper as a `creditSteps()` array; RP diagram via a `buildRP()`
createElement helper (same pattern as buildWave).

**Build order.**
1. [ ] Confirm scope with user: extend AXI-Explorer (recommended) vs standalone
       `AXI-IssueL-Explorer.dc.html` page + landing card; confirm accent hue.
2. [ ] Logic: `creditSteps()`, `buildRP()`, transport-comparison rows; add `transport` +
       `credits` to allowed sections; add `creditStep` state; nav "Issue L" group; wire
       into renderVals + return.
3. [ ] Template: comparison table + property callout (transport section); credit stepper +
       RP diagram + shared-credit panel (credits section); Issue L row in Versions.
4. [ ] Verify all sections render; bump version (minor — new feature) + changelog.

**Open questions for user.**
- Extend the AXI explorer, or a dedicated Issue-L page in the family grid?
- How deep on the credit stepper — a simple grant/spend/return animation, or a full
  multi-RP interleaving waveform?
- New accent hue OK, or keep it monochrome within the existing AXI blue?

</details>

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
- **v1.2.1** — Removed the standalone Issue L landing-page card (hero count back to 4);
  Issue L deep-dive now reached from within the AXI spec — a callout after the Handshake
  section plus the existing Versions-section link.
- **v1.2.0** — Added AXI Issue L explorer (credit-based transport, credit-flow stepper,
  full multi-RP interleave) + landing-page card + Versions-section teaser; hero interface
  count updated to 5.
- **v1.1.0** — Added ACE (AXI Coherency Extensions) explorer + landing-page card;
  hero/footer interface counts updated to 4.
- **v1.0.0** — Site reorganised into `public/`; stale root copies removed; missing
  back-link added to the AXI directions study; semantic versioning + plan-file
  conventions established.
