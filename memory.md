# Project Memory — priya_global_jobs

Priya's job board for **English-speaking countries outside Finland**, with a local-LLM-only scraper.
Created 2026-09-27 as a sibling of [../priya_jobs](../priya_jobs/memory.md) (her Finland + remote
board). The site list comes from a read-only probe of ~45 boards that day (details in
priya_jobs/memory.md, "Trial: English-speaking countries").

## Firestore locked down; Python scripts use a service account (2026-09-28)

`firestore.rules` used to leave `shared_state` / `user_feedback` readable and updatable by
anyone (`if true`) so the unauthenticated Python REST calls could write. Now every collection is
allow-listed accounts only, and scripts authenticate via **`firestore_auth.py`** (`session()` returns a
`google.auth` `AuthorizedSession` with the project's service account; SA requests bypass rules via IAM).
- Key file: `~/.secrets/priya-global-jobs-sa.json` - OUTSIDE the repo (Firebase Console -> Project settings ->
  Service accounts -> Generate new private key). `.gitignore` blocks `*firebase-adminsdk*.json` / `*-sa.json`.
- **Any other PC** running scripts or skills that touch Firestore (tailor-resume, fill-form,
  find-apply-link, mark-job-deleted) needs its own key at that path plus `pip install google-auth`
  in the venv - otherwise `firestore_auth.session()` raises FileNotFoundError.
- New Firestore calls must use `firestore_auth.session().get/patch(...)`, never bare `requests` - a bare
  call now gets 403. Rules deploy: `firebase deploy --only firestore:rules` from `firebase_app/`.
- Key access (2026-09-28): priyabkc99@gmail.com has roles/firebase.viewer + roles/iam.serviceAccountKeyAdmin on this project, so Priya can generate her own PC key in the console (no DB/rules/hosting rights). Owner: vineethkaimal1989@gmail.com.


## Major Features
1. **Global job scraping:** LinkedIn in 10 English-speaking countries plus 9 national job boards (UK, IE, CA, AU, ZA, MT), 12 role keywords.
2. **Local-LLM screening only:** every job is reviewed by the shared LM Studio server against `job_requirements.md` — no cloud LLMs.
3. **Dashboard:** Firebase Hosting at https://priya-global-jobs.web.app, cross-linked with the Finland board, data served from GitHub Pages.

## Scope decisions (user, 2026-09-27)
- Countries: UK, Ireland, Canada, Australia, New Zealand, Singapore, India, South Africa, Malta,
  Hong Kong. Any work model; she's open to relocating. Never the US.
- A posting that explicitly says no visa sponsorship / must already have the right to work there
  is a "no" (she only holds a Finnish permit). Silent postings are judged normally.
- The LLM reviews every scraped job (no title pre-filter), even though local review is slow.
- Lowest priority on the local LLM: OpenClaw > manju > vineeth > priya (Finland) > priya-global.

## Infrastructure
- Folder `C:\Users\vinee\priya_global_jobs`, GitHub `vinchess1989/priya-global-jobs-dashboard`
  (public, Pages from `main`), Firebase project `priya-global-jobs` (Firestore `eur3`, Hosting).
- Firestore placeholder docs `shared_state/job_status` and `shared_state/re_review_request` were
  created on setup (same create-vs-update rules gotcha as priya_jobs).
- Two machines: the scraper/Scheduled Task runs on Vineeth's PC (`C:\Users\vinee\...`) and pushes
  to `vinchess1989/...` (the canonical repo). Priya's PC (`C:\Users\priya\priya_global_jobs`, where
  the resume/apply skills run) has `origin` = her fork `priyabkc99/priya-global-jobs-dashboard`
  and `upstream` = vinchess1989. So skill commits (e.g. `input.csv`) land on the fork only; the
  dashboard is unaffected because resume links go straight to Firestore. Don't "fix" the
  vinchess1989 slug in CLAUDE.md/scripts to priyabkc99 — vinchess1989 is correct.
- venv uses LM Studio's bundled CPython 3.11 like the siblings, but has Playwright 1.63 (newer
  than the siblings) — it needed its own `playwright install chromium`.
- Scheduled Task `PriyaGlobalJobsLocalLLMOrchestrator` → `orchestrator.py` → `scraper.py`.
  Restarting: stop the task AND kill the child `scraper.py` (single-instance lock).
- `LOCAL_LLM_ENDPOINT`/`LOCAL_LLM_MODEL` come from the Windows user env vars (shared).

## Scraper specifics (vs priya_jobs)
- `SITE_JOB_URL_PATTERNS` + `parse_known_board`: on the listed boards only links matching the
  board's job-detail regex are kept, with query/`;jsessionid` stripped. Without it, town filters
  (`/jobs/<kw>/in-<town>`), menus and categories got through, and PNet (`-inline.html`) and
  CareerJunction (`-job-NNN.aspx`) jobs were dropped. Job Bank wraps the whole card in one link,
  so the title comes from `.noctitle` (`_anchor_title`).
- LinkedIn: 1 page per country/keyword and `LINKEDIN_DELAY_SECONDS` (8s) before each request —
  the probe got HTTP 429 after ~60 quick requests. Malta needs `geoId=100961908`:
  `location=Malta` resolves to Malta, Ohio.
- Cross-board dedupe: `_dedupe_key` compares LinkedIn jobs by numeric ID (the same job appears
  as fi./uk./mt.linkedin.com) and skips anything already in `..\priya_jobs\jobs.json`.
- Review queue: never-evaluated first, newest `added_at` first within that (the Finland board
  uses file order).
- Job Bank's keyword search is loose (a "Release Manager" search returns admin jobs), and
  JobsInMalta ignores keywords entirely — both cost local-LLM time on irrelevant jobs.

## Setup status (2026-09-27)
- Verified: all 9 boards + LinkedIn Malta return clean job links (live parser test); a test
  scrape added 15 LinkedIn UK DevOps jobs (still `pending`); both dashboards are deployed and
  cross-linked; GitHub Pages serves `jobs.json`.
- **Local-LLM review verified live (2026-09-28):** the task started 13:49 with the other
  scrapers, and 159 reviews ran in ~95 min. Every `eval_model` was `local/...` (gemma-4, then
  qwen3-14b once LM Studio switched), with no cloud calls and no regex-fallback verdicts. 11 gemma
  runs hit the 4096-token budget → `error` → retried. The sponsorship rule produced 14 correct
  "no"s (UK/IE/AU/SG/ZA: citizenship, no sponsorship, existing right to work).
- **Clearance rule tightened (2026-09-28):** "DevOps Engineer- SC Cleared" (posting: "eligible for
  SC Clearance") got "yes" from qwen3-14b, because the rule only named clearance roles that
  *require citizenship*. It now says any role requiring a security clearance or eligibility for
  one is "no" (UK SC/DV/NPPV, AU Baseline/NV1/TSPV, CA Reliability/Secret, NZ vetting). Re-tested
  on that job → "no". The edit re-queued all 150 jobs for re-review (needs_re_review).
- Throughput with qwen3-14b (non-reasoning) is far better than the planning estimate: ~35–40 s
  per job, i.e. ~1,500+/day if LM Studio is free; gemma-4 is several times slower.
- Google sign-in is enabled in the Firebase Console (done by the user 2026-09-28).
- The first Firestore database was accidentally created in `nam5`; it was deleted and recreated in
  `eur3`. A deleted `(default)` ID can be reused only after ~5 min.

## Error-retry cap: fixed a commit storm (2026-09-29)
Five Totaljobs detail pages failed every time with `ERR_HTTP2_PROTOCOL_ERROR` (most Totaljobs
pages load fine). `error` jobs counted as pending, so each loop retried them (~1s each), found
nothing else to do, saved a history snapshot, and committed+pushed: 258 commits in one hour.
Fix: `_needs_review()` / `_record_review_outcome()`. An `error` job is retried at most
`ERROR_MAX_ATTEMPTS` (3) times, no sooner than `ERROR_RETRY_SECONDS` (6h) apart; `error_attempts`
/ `last_error_at` are stored on the job and cleared by any real verdict. Jobs that exhaust their
retries stay `error` on the dashboard. The same retry-every-loop pattern exists in the sibling
scrapers (priya_jobs, manju_jobs, vineeth_jobs) — not ported yet.
**Gotcha (cost a scare):** `open(path, "w", newline=<invalid>)` truncates the file BEFORE raising
ValueError. It emptied scraper.py once (restored with `git checkout`). Patch files via a temp file
+ `os.replace`, never by rewriting in place from an inline PowerShell here-string.

## Dashboard: "Jobs by country" card + jobs-added timeline (2026-09-28)
- Card under the Today card, with one tile per country: total jobs, "+N today" (by `added_at`, local
  date), and yes count (Firestore `shared_state/job_status` override wins, like the Today card).
  Country comes from the **scrape `source` prefix** (`COUNTRY_SOURCES` in index.html), not the
  free-text location. **Add a mapping there whenever a board is added to the scraper**, or its
  jobs land in "Other".
- "Timeline" button / tile click opens `#timeline-modal`: an SVG stacked bar per Day/Week/Month
  (yes matches bottom, other jobs top) with a country selector, hover tooltip and summary line.
  It counts `jobs.json` + `deleted.json` by `added_at`, so its totals can exceed the tiles by the
  jobs removed since. Colors #059669 / #6366f1 passed the dataviz validator on #161b2e (tritan ΔE
  6.6, hence the 2px gap + legend).
- **Tile click = country filter on the jobs table** (`selectedCountries`, multi-select toggle, AND-ed
  with the column filters, saved in the same localStorage filter blob as `countries`, cleared by
  "Clear filters" and by the "N countries ×" chip in the card header). Each row-cache entry carries
  `country` (from `jobCountry`). The tile's small chart icon opens the timeline instead. The table
  count is lower than the tile total because "no" jobs are hidden by default.
- Each tile also has two chips: **"N matching"** (yes + maybe, all time) and **"N matching today"**.
  `showCountryMatches(country, todayOnly)` resets all filters, then sets that single country, the
  Matches column to yes+maybe, and (today) `added-days-filter` = 0, exactly like the Today card's
  "View list". Chip counts are computed with the same rules, and a browser test confirmed
  count == filtered rows for all 10 countries. Chips with 0 are disabled. Test "today" behaviour
  with Playwright's `page.clock.install(...)` when nothing has been added yet that day.
- The Day/Week/Month buttons use `.tl-gran`, NOT `.chart-gran`: the history chart's code
  re-syncs `.active` on every `.chart-gran button` and would clobber them.
- Tested locally with Playwright: Firebase Hosting's `/__/firebase/*` SDK URLs were routed to an
  auto-sign-in stub (the page only fetches data after auth), with live GitHub Pages data.

## Known blocked boards (don't re-add without a new approach)
Indeed (all countries, Cloudflare), Seek AU/NZ, JobStreet SG, JobsDB HK (Cloudflare), Naukri,
Foundit (Access Denied), Adzuna (403/429), CTgoodjobs, KeepMePosted (captcha), Eluta, Careers24,
MyCareersFuture (JS-only), JobsPlus MT (404), Guardian Jobs / Trade Me (ignore keywords).

---
Last updated: 2026-09-27

## Mobile bottom nav: filter indicator (2026-09-29)
While any filter is set (column filters, date limits, location text, LLM pills, and on the global board countries), the Filters tab shows an amber dot and its label becomes `shown/total` (e.g. 32/222), set at the end of `filterTable()` (`#mnav-filters.has-filters`). Total = rows in the table, i.e. excluding archived 'no' jobs unless shown. Also 2026-09-29: the header board link is now a prominent `.board-switch` button (icon-only on phones) on both Priya boards.

## Firestore lockdown rules actually deployed 2026-09-29
The 2026-09-28 lockdown entry above said bare calls 'now get 403', but the locked-down `firestore.rules` had never been deployed: anonymous reads still returned 200 on all four projects (priya-jobs-dashboard, priya-global-jobs, manju-jobs-dashboard, vineeth-jobs-dashboard) until 2026-09-29, when they were deployed with `firebase deploy --only firestore:rules`. Verified after deploy: anonymous GET on `shared_state/job_status` and `user_feedback` -> 403; service-account `firestore_auth.session()` -> 200; no unauthenticated Firestore calls in any repo's .py files. **To check the lockdown, test anonymous access yourself** (`Invoke-WebRequest https://firestore.googleapis.com/v1/projects/<id>/databases/(default)/documents/shared_state/job_status` should throw 403). The committed rules file alone proves nothing. Any other machine needs its `~/.secrets/<project>-sa.json` key (Manju's PC was pending at deploy time).

## Resume/apply skills ported for the global board (2026-09-29)
`tailor-resume` / `fill-form` / `find-apply-link` / `mark-job-deleted` + helpers copied from priya_jobs and
repointed (6 hard-wired `priya-jobs-dashboard` refs -> `priya-global-jobs` project /
`priya-global-jobs-dashboard` repo; all Firestore calls already used `firestore_auth`). Global-specific wording:
cover-letter paragraph 4 + profile facts (relocation to the job's country; never claim right to work; Finnish PR /
learning-Finnish lines removed; sponsorship mentioned only if the posting raises it), fill-form right-to-work =
"No — would require visa sponsorship", fill-form sanity check uses this board's countries + sponsorship/clearance
rejections. Verified on this PC: `find_repos.py` finds both repos, `job_status_store.py` reads priya-global-jobs,
`make_resume.py` + `html_to_pdf.py` render the master resume/letter PDFs. A real `/tailor-resume` run (which
commits to the private repo) not yet done — first one will be from Priya's PC.
**Priya's PC setup:** clone `priya-global-jobs-dashboard` to `C:\Users\priya\priya_global_jobs` (sibling of
`priya-jobs-private`), `python -m venv venv`, `pip install playwright requests python-dotenv beautifulsoup4
filelock google-auth`, `python -m playwright install chromium`; key already at
`C:\Users\priya\.secrets\priya-global-jobs-sa.json`. Run the skills from that folder with global job IDs.
**Known gap (also on priya_jobs):** `mark-job-deleted` writes `deletion_reason` to Firestore, but no scraper
reads it — only review.html hides such jobs; the main table and jobs.json keep them. The skill's claim that "the
scraper moves them to deleted.json" is not true on either board.
## Scraper was wiping shared_state/job_status (found + removed 2026-09-30)
`poll_firebase_feedback()` ended with `PATCH shared_state/job_status {"fields": {}}` ("clear the temporary queue")
whenever dashboard feedback produced user_review/match updates. job_status is a PERMANENT per-job store (resume
links, apply_url/apply_email, applied_date, form_filled, action_item, tailor_model, deletion_reason), so every such
run erased it for every job. priya_jobs logs show it ran 17 times (latest 2026-09-29); the doc was found empty on
2026-09-30. Removed from priya_jobs, priya_global_jobs and vineeth_jobs (manju_jobs no longer had it). Recovery:
resume/cover-letter links rebuilt with `sync_resume_links.py --upload --force` (20 jobs). NOT recoverable:
apply_url/apply_email, applied_date, form_filled, action_item, auto_fill_attempted_at (Firestore PITR is off).
applied / user_review / matches values survived because they had already been synced into jobs.json.
**Never write an empty/whole replacement document to job_status** — read-modify-write only (job_status_store.py).
## Manual deletions now processed (2026-09-30)
`poll_manual_deletions()` (ported from manju_jobs) runs every main-loop pass: jobs whose Firestore job_status entry
has `deletion_reason` (mark-job-deleted skill, job_status_store.py, review.html "missed" button) are moved from
jobs.json to deleted.json with that reason, then the flag is cleared (other fields kept). Before this, no Priya
scraper read the flag, so such jobs stayed on the main board. Tested on temp copies with a simulated job_status.
## Favourites + automatic submitting (built 2026-10-01; switch OFF until tested)
Priya's rule: starred (★) jobs are filled and left for her to review/submit; every other job may be submitted
automatically by `/fill-form` - only if `auto_submit_gate.py check` passes ALL of: dashboard switch
`shared_state/settings.auto_submit_enabled` true (doc absent = OFF); not starred (`job_status[url].favorite`);
match == yes; not applied; apply host not LinkedIn / Teamtailor / Biisoni; < 10 automatic submits today on this board;
no applied job with the same company+title on EITHER Priya board (sibling board's jobs.json, local or GitHub Pages);
no automatic submit to the same company in 7 days; no salary question (Priya's decision: salary -> review);
no required DOB; every required field answered (answers.json `placeholder` false); no required upload other than
CV/cover letter. Fill script (fill-form Step 5) re-checks the LIVE page (empty required fields / CAPTCHA / password
field) before clicking, screenshots before/after into `PRIVATE\Resumes\<id>\`, detects a confirmation page.
`auto_submit_gate.py record`: confirmed -> applied=yes + applied_date + `form_filled.status=submitted`
(`submitted_by=fill-form-auto`, review.html Auto-Submitted tab) + user_feedback applied_update (scraper syncs
jobs.json); clicked-but-unconfirmed -> `verify_submission` action item, NOT applied; refused -> pending_review.
`/fill-form auto 10` handles up to 10 jobs per run (``). Phone = resume's Finnish number.
Dashboard: ★ per row/card, "★ Favourites (N)" filter (saved, counted in the mobile filter badge), "Auto-submit:
ON/OFF" button (confirm dialog; allow-listed accounts only).
Also: `job_status_store.update_job_fields()` - field-masked writes (backtick-quoted URL field paths); `set_job_field`
and the scraper's deletion-flag clearing use it, so nothing rewrites the whole job_status doc any more.
Tested: gate scenarios (unit), record outcomes (writes intercepted), submit block on 4 local test forms
(confirm / empty required / CAPTCHA / no confirmation), dashboard star/filter/switch with an in-memory Firestore stub.
**Pending rollout (needs Priya's PC + Chrome profile):** 2-3 real `/fill-form` runs with the switch OFF (fill
only), then ONE real automatic submit with her watching, then turn the switch on and start `/loop 60m /fill-form auto 10`.
Gotcha: job_status contains a non-job `initialized` placeholder field - code iterating it must skip non-dicts.
## GitHub push auth (2026-10-03)
- `update_git()` in scraper.py pushes through Git Credential Manager first (remote URL names the account:
  `https://vinchess1989@github.com/...`; GCM supplies the stored login, `GCM_INTERACTIVE=never` so a background run
  never opens a sign-in window). `GITHUB_TOKEN` is only a fallback, inserted after stripping any user from the URL.
- Why: the user-level `GITHUB_TOKEN` went stale (401) and every scraper push failed silently from ~01:30 on
  2026-10-03 (manju_jobs 18 commits, priya_global_jobs 88 behind); then pinning the account in the remote URL made the
  old code build `https://TOKEN@vinchess1989@github.com` ("URL rejected"). The dead env var was removed.
- Only Manju_jobs_private pushes as munchnambiar; everything else as vinchess1989 (global CLAUDE.md rule).