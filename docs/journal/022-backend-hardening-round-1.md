# 022 — Backend hardening, round 1: progress writes, auth abuse limits, upload limits, pool

**Date:** 2026-09-20 · **Phase:** hardening (from docs/research/002) · **Status:** PASS for what was built; the biggest items (queue worker, presigned uploads) are NOT done

## 1. Decision
Fix the risks the research found that are small enough to fix and prove now, tests first: (a) reading-position saves, (b) login/register abuse limits and client-IP handling behind a load balancer, (c) cross-site write blocking, (d) page and word limits checked before extraction, (e) an explicit database pool. Leave the architectural changes (moving OCR and extraction into a queue worker, presigned uploads) for their own step.

## 2. Why (interview answer)
> "A research pass compared my backend with how real systems handle ingestion, position sync and auth. I checked each claimed risk against my own code first, and all of them were real. I fixed the ones I could prove with tests, and for each I wrote the test before the fix, watched it fail for the predicted reason, then broke my own fix on purpose to check the test would notice. For example, a reader saves its position constantly, and my upsert wrote a new row version even when nothing had changed. I proved that with the row's own version number, fixed it, then proved the fix with the same number."

## 3. Alternatives & who uses them
| Area | Chosen | Alternatives |
|------|--------|--------------|
| Progress conflicts | Last-write-wins by client timestamp, plus a "furthest" marker | Server arrival order (what I had); per-device positions merged on read; CRDTs (collaborative editors; wrong tool for one number). Kindle and Instapaper are described in research 002 as furthest / client-timestamp systems |
| No-op writes | Conditional `UPDATE ... WHERE (page, word_index) IS DISTINCT FROM ...` plus insert-if-missing | `INSERT ... ON CONFLICT DO UPDATE` (still locks the row and writes when nothing changes, per Datadog, 2026-03-23). A Redis write buffer is common at larger scale |
| Client IP | Read `X-Forwarded-For` from the right by a configured number of trusted proxies, else ignore it | Trust the header blindly (spoofable), or use the socket address (the balancer's, so all users share one IP) |
| Abuse limits | Redis counters per email+IP, per IP, per email; per-IP register limit | WAF or ALB rate rules (coarse, per IP only); a managed identity provider such as Cognito, Auth0 or Clerk (rotation, verification, MFA for free; deferred in journal 015) |
| CSRF | Origin check on state-changing methods, on top of SameSite=Lax | Per-session CSRF tokens (needed if cookies ever become SameSite=None) |
| Pool | Explicit size 5, overflow 5, timeout 10 s | pgBouncer or RDS Proxy (needed before roughly 1,000 users, still planned) |

## 4. How it was tested
Every test was written first and failed for the predicted reason on the old code (for progress: the version number changed on a no-op save, a stale write overwrote a newer one, no furthest field, the whole library was listed, concurrent first saves failed).
- New tests: `test_progress_hardening.py` (9), `test_auth_hardening.py` (14), `test_upload_limits.py` (4), `test_db_pool.py` (2). Whole backend suite: **206 passed, 0 failed** (was 177). `ruff check --no-respect-gitignore app tests alembic`: exit 0.
- **Mutation checks, 19 of 19 caught:** progress (drop the no-op guard, the stale guard, GREATEST, ON CONFLICT), auth (never trust the header, drop each throttle, drop the origin check, drop the fetch-site check, use the spoofable leftmost entry), limits (drop or shift each cap), pool (ignore the size).
- Migration 0008 verified both ways with a column listing at each step (down removes the three columns, up restores them). Old API code keeps working while it is applied (only nullable or defaulted columns).

## 5. Result → next step
Fixed: progress writes (no-op writes write nothing, LWW, furthest, one query instead of a library scan, storage tuned for a hot table), per-IP and per-email login throttles, per-IP register throttle, correct IP behind a balancer, cross-site write blocking, page cap (default 3,000) and word cap (default 3,000,000) before extraction, explicit pool. **Not done, still the biggest risks:** OCR and extraction still run in the API process; whole uploads are still buffered in memory (a hostile PDF can still exhaust one process; the caps only shrink the window); no presigned uploads; no content-hash dedup; no single-flight on cache misses; no per-request user cache; JWT still lives 7 days without refresh or revocation; `document_words` partitioning. **The dev API container was not rebuilt or restarted**, so the running stack still has the old code until you rebuild it.

## 6. Failure analysis
1. **Ruff was checking nothing.** The repo `.gitignore` is `*`, so ruff skipped every file and reported a clean pass. Use `--no-respect-gitignore`. (Found in journal 019, hit again here.)
2. **Formatting spilled into an unrelated file.** Running `ruff format` on the whole `app` directory reformatted `app/services/ocr.py` (formatting only, no behaviour change; found by listing recently modified files). I now format only the files I touch.
3. **Ambiguity I resolved on purpose:** a per-email throttle lets an attacker briefly lock one email out (the price of stopping guesses spread across addresses). Chosen limit is higher (20) than the email+IP rule (5).
4. **A footgun caught before shipping:** behind the load balancer every user would share one IP, so the per-IP throttles would lock everyone out together. Production now refuses to start unless `TRUSTED_PROXY_HOPS` is set explicitly (tested). Terraform/ECS must set it (1 behind one ALB).
5. **Test-suite side effect:** abuse counters live in Redis and the whole suite shares one address, so `conftest.py` sets very high limits; the hardening tests use tiny limits and a unique forwarded IP per test. The e2e suite registers many accounts from one address; the default register limit (100 per 15 minutes per IP) could trip repeated full runs, so raise `REGISTER_MAX_PER_WINDOW` for local e2e.
6. **Silent tooling output:** `alembic upgrade/downgrade` printed nothing, so I could not tell it had run; verified with a column listing between steps instead.

## 7. Replicate it yourself
1. Write the test that proves the risk (for a no-op write: read `xmin::text` from the row before and after an identical save).
2. Run it against the old code and confirm it fails for the predicted reason.
3. Fix it (`alembic revision` for schema, then code), rerun.
4. Break the fix on purpose (delete the guard), confirm a test fails, restore.
5. `ruff check --no-respect-gitignore app tests alembic`, full `pytest`, `alembic downgrade` then `upgrade` with a column listing between.
