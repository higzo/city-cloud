# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

City Picker is a single-file, no-build web toy: `index.html` holds all HTML, CSS and JS. There is no package manager, test suite, linter or git repo. To run it, open `index.html` in a browser (needs network for the Matter.js CDN script, `matter-js@0.19.0` from cdnjs). Press Space to start a round.

Known quirk: line 1 begins with a stray `names[k]` before `<!DOCTYPE html>`, which puts the page in quirks mode and may render as text. Remove it if touching the file's head.

## Architecture

One round is a pipeline, all driven by a single `requestAnimationFrame` `loop()` and a `state` flag (`'ready'` / `'running'`):

1. **Spawn** (`spawnCities`): draws `COUNT` distinct names from `CITY_POOL` (comma-separated string, ASCII, ≤16 chars) in descending sizes, rejection-sampling positions so words don't overlap and avoid the "press space" text. Each word is a DOM `<span>` in `#cities` paired with a Matter.js rigid body (`makeWord`); `render()` copies body position/angle/scale onto the span's CSS transform. Words start frozen (physics only advances while running).
2. **Physics decay** (`loop`): fixed-step `Engine.update`. Once a word lands and rests (`SETTLE_MS`), it ages: brightens toward white, shrinks (`Body.scale`) at its random `rate`, and below `vaporAt` turns into "vapor" that floats up via applied force until it leaves the top or hits `GONE_SCALE`. Hard impacts (`BREAK_SPEED`) snap long words in half; rising vapor can shatter words above it. Breaks are queued in `toBreak` during the `collisionStart` handler and applied after the step (`breakWord`), since bodies can't be replaced mid-step.
3. **Music coupling**: `lettersGone / lettersTotal` gives the share of letters remaining; `pitchCurve` (monotone cubic LUT built from `PITCH_POINTS`) maps it to tape-style pitch/tempo for the Web Audio synthwave generator in the `music` IIFE. Music is scheduled ahead on a `setInterval`, independent of the render loop, and fades out at `MUSIC_END_FRAC`. Each round `randomize()` picks a new key/progression/instruments.
4. **Finale** (`startAssembly` / `stepAssembly`): when pitch drops below `ASSEMBLE_PITCH`, half the leftover words leave the physics, split into single letters, glide to screen centre, and `alignCity` (DP alignment scoring) picks the pool city best matching those letters. Matching letters are kept, others deleted, missing ones faded in, then the name is capitalised, grows and fades; `openSearch` opens a Google search for the city (falls back to a visible link if pop-ups are blocked). The remaining words keep floating and the round ends when none remain, returning to `'ready'`.

Visuals are CSS-only layers (`#sky`, `#stars`, `#floor`/`#grid` perspective grid) behind the word layer via negative `z-index`. The tuning constants (`STILL_SPEED`, `SETTLE_MS`, `BRIGHTEN_MS`, `SHRINK_MS`, `VAPOR_SCALE`, `T_*` finale timings, `PITCH_POINTS`) are interdependent: changing decay speed shifts when the finale triggers, because the finale is keyed to letter count and pitch rather than time.
