# 018 — Hardening stage 1: scenario tests, and what they found

**Date:** 2026-09-20 · **Phase:** UI quality · **Status:** PASS after 6 real defects found and fixed

## 1. Decision
After stage 1 "worked" (journal 017), go back and attack it: write a scenario suite for Home (`frontend/e2e/home.spec.ts`, 14 tests) *before running it*, add static guards (`src/tokens.test.ts`), review screenshots of every page I had not looked at, and fix what surfaces. Also set standing rules so this happens every time (CLAUDE.md §10, `.claude/skills/sera-quality-gate`).

## 2. Why (interview answer)
> "The first pass passed all its checks, and it still had bugs. Passing tests only prove the things you thought to test. So I wrote down expectations first, then tried to break my own work with different situations: an empty account, sixty documents, a 90-character title, the API returning 500, Night mode, a phone width, keyboard only, and a second account. Six real defects fell out, two of them silent (an invisible focus ring, and a test that passed for the wrong reason). Then I added guards so those classes of bug can't return unnoticed."

## 3. Alternatives & who uses them
| Choice | Alternative | Notes |
|--------|-------------|-------|
| Scenario e2e with mocked API responses (`page.route`) for failure and scale cases | Seed a real database with 60 PDFs | Real data is slower and flakier; mocks make failure cases (500, abort) deterministic. Real-PDF corpus tests still belong in the load/ingestion suite (see docs/research/002) |
| Static token-guard unit test | Stylelint / a CSS-in-JS type system | Commonly used; a 40-line test was enough here and is easy to explain |
| Screenshot review by a person | Automated visual regression (Playwright `toHaveScreenshot`, Percy, Chromatic) | Visual-regression tools are commonly used for exactly this; worth adding once the design settles (stage 2) |

## 4. How it was tested
Expectations are the comment block at the top of `home.spec.ts` (written first).
- First run: `npx playwright test e2e/home.spec.ts --project=chromium` → **10 passed, 4 failed** (see 6).
- Final: `tsc`, `lint`, `build` exit 0; unit **43 passed**; Chromium whole suite **127 passed, 0 failed, 5 skipped**; Home + auth on Firefox, WebKit and hi-DPI Chromium **59 passed, 1 skipped**.
- Mutation checks (broke the code, saw the guard fail, restored): re-adding `var(--ai)` → `index.css: --ai` reported; re-adding `class="frame"` → `layout/AuthLayout.tsx` reported.
- Also fixed the 6 earlier "failures": `e2e/fixtures/scanned-turned.pdf` was simply not generated (`conda run -n sera python scripts/make_e2e_fixtures.py`).

## 5. Result → next step
Stage 1 is now covered by scenarios. Next: read the two research notes (`docs/research/001`, `002`, being written by Fable subagents), then stage 2 (Reader). Not yet verified: real audio (none exists), Home with genuinely huge PDFs, Safari on a real iPhone, screen-reader pass, visual-regression baselines.

## 6. Failure analysis (each is a real setback, kept on purpose)
1. **Undefined CSS variables (silent).** I rewrote `index.css` but kept the old `@layer base` block, and my rename script only touched `.tsx/.ts`. `:focus-visible { outline: … var(--ai) }` therefore became *no outline*, and `body { color: var(--sumi); background: var(--washi) }` fell back to browser defaults. Found by the keyboard test (`outlineStyle: none`), then a script listing every `var(--x)` used vs defined. Fix: rename in `index.css`; guard test. Alternative considered: keep old names as aliases: rejected, it would hide exactly this kind of drift.
2. **A test that passed for the wrong reason.** My Night contrast test measured the headline against `body`, which was transparent because of (1), i.e. against black. Fix: walk up to the first ancestor that actually paints a background, and throw if none does.
3. **Legacy `frame` class left behind.** My regex skipped `className="frame …"` (lookbehind on `"`), so Login, Setup, Upload, Discover and Collection cards rendered square, unstyled boxes. Found by looking at the login screenshot, not by any test; fixed with a replace, plus a guard that fails on removed class names.
4. **A failed request looked like an empty shelf.** Home turned any error into `docs = []`, so an outage told a user with fifty books to "begin with a page". Test written first (expected to fail), it failed, then Home got a real error state with **Try again**. The lint rule `set-state-in-effect` rejected my first fix (resetting state in the effect); reset moved into the click handler (same lesson as journal 016).
5. **A very long single word overflowed and was hidden.** A 90-character title made the headline 537px wide on a 390px phone; the shell's `overflow-x: clip` hid the overflow, so a "no sideways scroll" check alone passed. The test also asserts the headline's own box, which exposed it. Fix: `min-w-0` on the grid child and `overflow-wrap: anywhere`.
6. **My own test mistakes:** the route glob `documents?*` doesn't match `/documents` with no query (regex used instead); the keyboard test pressed Tab before the hero rendered (now waits for it); `Playwright` isn't an exported type (use `PlaywrightWorkerArgs["playwright"]`); a zsh loop variable was not word-split (exit 127), a trap I had already met in 017.
7. **Visual only:** in Night the darkest spines nearly vanish into the shelf band; added a faint ring. Spine labels now use `displayTitle` so filename titles read as words.

## 7. Replicate it yourself
1. Write the expectations as a comment block before any test code.
2. Cover: empty, one, many (60), long/odd titles, non-ok API responses, Night, 390px, keyboard, second account.
3. Use `page.route(regex, …)` to mock lists and failures; use real uploads where realism matters.
4. Add static guards for silent failures: undefined CSS variables, removed class names, decoration characters.
5. Mutation-check each guard, then run the whole suite and look at screenshots.
