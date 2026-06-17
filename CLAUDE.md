# Project conventions


- **Deployable site lives in `public/`.** Root-level site files are stale — `public/` is the source of truth.
- **Semantic versioning on zips.** When offering a download zip, label it with a version number (e.g. `v0.1.0`, `v0.1.1`). Increment the patch for fixes, minor for new features, major for breaking changes.


# Project Instructions

## File naming
- Files must not contain spaces.
- The main file must be named `index.html`.

## Project layout (applies to ALL sites in this project, current and future)
- **All servable site code lives in `public/`** — the deployable entrypoint is
  `public/index.html`, alongside  any assets the site loads. `public/` must be fully
  self-contained: use only **relative** references.
- **Project meta stays at the root, never inside `public/`:** `CLAUDE.md`, `HANDOFF.md`
  (and other plan files), `README.md`, and `screenshots/`. These must not ship with the site.
- When you create a new site or add files to one, place them under `public/` and keep this
  split. After editing a `.dc.html`, if the `support.js` runtime changed, refresh the copy
  in `public/` (the platform also keeps one at the root for in-platform editing).
- Preview/deliver sites via their `public/index.html`.

## Planning new features
- Before executing any new feature, write a plan to a plan file (e.g. `HANDOFF.md`) first:
  the feature restated, current architecture/key methods, data-model changes, UI work,
  and a step-by-step build order. Keep it updated as you go and mark sections done.
- This lets work resume cleanly after a restart — always check the plan file before starting.