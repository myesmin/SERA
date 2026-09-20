# How Sera is built

Sera's source is private. This page is about the *method* — how decisions get made, how they get checked, and what happens when they turn out to be wrong. Two full entries from the build journal are included at the end so you can see the real thing rather than a description of it.

---

## Every non-trivial decision is written down before it is forgotten

The project keeps a build journal: one file per decision, in a fixed shape.

1. **What was decided**, in a sentence.
2. **Why**, in the first person, naming the constraint that actually drove it.
3. **The alternatives**, at least two, each with where it is genuinely used and the case where it would have won.
4. **How it was tested** — the exact commands, the expectation written down *before* running, and the raw output pasted in.
5. **The result, and what was done next because of it.**
6. **Every failure**, with its root cause, how the replacement was chosen, and a retest. Failed attempts are never deleted.
7. **Steps to replicate it** from scratch.

There are 26 entries. The rule that makes it work is the boring one: an entry is never marked PASS without pasted command output.

## Expectations are written before the code runs

A test written after the code tends to describe what the code does. So the expectation goes in the journal first, in plain language, and then the code runs.

Sometimes it holds. Sometimes it does not, and that is the useful case. One recent change had six expectations; five held, and the sixth — "no file will get worse" — turned out to be false for exactly one file in a 43-document sample. That single exception led to a second feature, because the file in question was one no reader could have used anyway.

## Claims are measured, not asserted

Before changing how text is read off a page, the old behaviour and the new one were run across **1,053 pages** and scored on the same metric. The change moved it from 2.39 to 1.23 (lower is better), with a per-file table showing which files improved and which got worse.

That table is the point. The headline average would have hidden four regressions, and each one turned out to be a real bug:

- a rotated stamp down a page margin cut a paragraph in half
- a results table got read down its columns instead of across its rows
- two attempted fixes that *measured worse* and were reverted on the numbers rather than defended

Performance claims get the same treatment. "It adds about 3%" means someone ran it both ways on a 1,476-page file and wrote down 21.5s and 22.2s.

## Tests are checked by breaking the code

A passing test is not evidence until you have watched it fail. So after the tests are green, each tunable constant in the new code is broken in turn and the suite is re-run. The result is a table: mutant on the left, which tests died on the right. Every constant must kill at least one test, and ideally the test whose name claims to cover it.

This has repeatedly found bad tests rather than bad code:

- A test asserted `text.indexOf(a) < text.indexOf(b)`. When the output was wrong the first string was missing entirely, `indexOf` returned `-1`, and `-1 < 22` passed. The test could not fail.
- A fixture was built row by row, so a reader that ignored the rule under test reproduced the expected order by accident.
- One mutation run was itself wrong — a find-and-replace hit the wrong function — and "still green" was briefly misread as "not covered".

All three are written up in the journal, because the mistakes are more instructive than the fix.

## The product never claims something it cannot do

A standing rule: the interface does not pretend. No audio controls before there is an audio engine. No invented definitions. When a page cannot be reflowed faithfully, it says so and offers the original layout instead.

The strictest version of this came up recently. A notice existed to warn that a page's layout would not survive being reflowed — and it had never once appeared on a real document, because the test behind it could not detect the case it was written for. The page was being silently mangled while the code claimed to be honest about it. Silence is the worst failure mode, and it only surfaced because the finished feature was opened in a browser and looked at rather than being signed off from green test output.

## Research before building

For any new subsystem, how existing products and papers solve it gets studied first and written up with sources, and every claim is labelled *confirmed* (measured, or stated by the source) or *inferred*. Made-up claims about what some company does are worse than no claim at all, and an interviewer will find them.

## Scale is a question asked early and answered in writing

"What breaks at 100 users, at 1,000, at 10,000, and with thousands of documents?" gets asked for every change, and the answer is written down even when it is "nothing yet, and here is why". Storage per unit of data, per-request work, and what moves from the request path to a background worker are tracked as the design grows rather than discovered under load.

---

## What runs on every change

| | |
|---|---|
| Backend | 230 tests — unit, property-based, and integration against real Postgres, Redis and object storage |
| Frontend | 98 unit tests, including randomised property tests over generated pages |
| Browser | 175 end-to-end tests in Chromium, with the layout-sensitive ones repeated in Firefox and WebKit |
| Checks | type-check, lint and production build, all judged by exit code |
| Also | screenshots at desktop and phone widths, keyboard-only paths, and reduced-motion |

Continuous integration runs the backend suite against real service containers — not mocks — so "it works on my machine" is not a thing anyone has to say.

---

## Two entries from the journal

These are unedited, in the format described above.

- [Frontend hardening](docs/journal/018-frontend-stage-1-hardening.md) — fourteen deliberately awkward scenarios run against a finished-looking UI. Four of the new tests failed on the first run; all four were real bugs or bad tests, and each one is recorded with its cause.
- [Backend hardening](docs/journal/022-backend-hardening-round-1.md) — rate limiting, request-origin checks, connection pooling and write amplification. Twenty-nine tests written before the code, each confirmed to fail for the predicted reason first.

---

**Mohona Yesmin** · [github.com/myesmin](https://github.com/myesmin)

© 2026 Mohona Yesmin. All rights reserved.
