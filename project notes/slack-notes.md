SKills Under Test

# The five skills under test

This document describes the skill corpus behind the baseline and behind experiments E and C.
Sources live in `sample-skills-report/<skill>/` (each holds a `SKILL.md`, an `eval.yaml`, and any bundled code), and all baseline numbers below come from the 8-repeat baseline in `sample-skills-results/`.

Shared eval configuration: run model `claude-sonnet-5[1m]`, judge `claude-opus-4-8`, pass threshold 0.7, `rewrite_rubric: true`.
Quality tests ran in the engine's read-only sandbox (tools `Read`, `Grep`, `Glob`, `Skill` only), so the agent DESCRIBES the prescribed procedure rather than executing it, and the judge is instructed to score a correctly described mechanism as satisfying action-shaped criteria.

## Corpus at a glance

| skill | domain | category | rubric style | invocation tests | quality tests | baseline quality mean (raw) |
|---|---|---|---|---|---|---|
| atlassian | Jira / Bitbucket / Confluence + PR event monitor | knowledge | enumerated | 10 (5 pos / 5 neg) | 4 | 7.88 |
| ioi-classifier | Bloomberg chat IOI classification | trading | prose | 10 (5 pos / 5 neg) | 5 | 9.70 |
| jenkins | CI build triage on manbuild-ci.res.m | infrastructure | enumerated | 12 (6 pos / 6 neg) | 4 | 8.78 |
| slack | Slack data access via Graph API proxy | messaging | prose | 11 (6 pos / 5 neg) | 6 | 9.00 |
| web-fetch | authenticated URL fetching and scraping | web | enumerated | 12 (6 pos / 6 neg) | 4 | 8.97 |

The corpus deliberately spans two authoring styles.
atlassian, jenkins, and web-fetch write rubrics as numbered clause lists ("A correct response must: (1) ... (2) ..."), which is what makes the E3 clause-retention measurement possible.
ioi-classifier and slack write prose rubrics, giving the enumerated/prose covariate used in E2.
The five domains do not overlap at all, which is what makes a cross-skill false pass in experiment C unambiguous.

---

## 1. atlassian

**What it is.**
Composable Python clients for Man Group's Atlassian estate: `JiraClient` (26 methods) for jira.maninvestments.com, `BitbucketClient` (37 methods) for the two Bitbucket Server instances mangit (HTTPS) and nigit (HTTP), and `ConfluenceClient` (12 methods) for manwiki.
Authentication is PAT-based, resolved from Vault at runtime.

**The behaviour it teaches.**
The skill's flagship workflow is "babysitting" a PR: after any push or PR creation the agent must arm `scripts/poll_pr_events.py` through the Monitor tool with `persistent=True` and spawn one subagent per emitted event (comments, approvals/needs_work, tasks, CI updates), and must never poll via the Bitbucket MCP.
The babysit loop drives a PR to green (CI passing, tasks resolved, approvals met) and then stops; merging is explicitly reserved for the human reviewer, and there is no `merged` event.
The skill also carries sharp scope boundaries: nigit is reachable only through this skill (MCP is mangit-only), the monitor is mangit-only so nigit PRs cannot be babysat, and Confluence user/group management is out of scope (routed to the active-directory skill).

**Bundled resources.**
`atlassian_client/` (the three clients plus `auth.py` and guardrail tests) and `scripts/poll_pr_events.py` (the unified event monitor), plus eval capture/compare helpers.

**Quality tests (4, enumerated rubrics of 4-6 clauses).**
q-01: arm the monitor for a freshly pushed PR, naming the covered event types.
q-02: review a nigit PR end to end with the correct HTTP client construction, and refuse both MCP and the (mangit-only) monitor.
q-03: a mixed-scope request that must be split - refuse the user/group management half toward active-directory, accept the page-creation half.
q-04: babysit plus a Jira follow-up, testing that the Jira transition is gated on the human's merge because the monitor emits no merged event.

**Baseline behaviour.**
Invocation: 9/10 on every one of the 8 repeats; the one consistent failure is `neg-02`, where a Forgejo URL on code.maninvestments.com wrongly triggers the skill on all 8 repeats, making it the corpus's one systematic invocation-boundary miss.
Quality raw scores over 8 repeats:

| test | scores | mean | sd | passed |
|---|---|---|---|---|
| q-01 | 8,8,9,8,9,9,9,9 | 8.62 | 0.48 | 8/8 |
| q-02 | 7,8,6,9,4,4,8,6 | 6.50 | 1.73 | 4/8 |
| q-03 | 9,9,9,9,9,9,9,9 | 9.00 | 0.00 | 8/8 |
| q-04 | 6,8,7,9,7,8,8,6 | 7.38 | 0.99 | 6/8 |

**Role in the experiments.**
This skill contributes both threshold-adjacent tests (q-02 at 0.65 and q-04 at 0.74 normalised) and, with them, essentially all of the corpus's interesting failure behaviour: q-02 is the only E1 cross-judge verdict disagreement and the only E2 verdict flip, q-04 draws a unanimous cross-judge fail, and the pair are the only real outputs that fail their own rubric in experiment C.
q-02 is also the least stable test in the baseline (sd 1.73, range 4-9).

---

## 2. ioi-classifier

**What it is.**
A pure-prompt classification skill with no bundled code: it teaches the agent to read Bloomberg IB Chat messages arriving via the BBG Chat Listener Kafka feed and decide whether each is an Indication of Interest (IOI) from a broker-dealer.
It is consumed by the ioi-classification-service through the Claude API with structured output.

**The behaviour it teaches.**
The skill defines the IOI concept (expressed trading intent with direction, instrument, and usually size/price/urgency), a strict JSON output schema (`is_ioi`, `confidence`, `ioi_type` bid/offer/two_way/unknown, `asset_class` equity/fixed_income/fx/derivatives, ticker, isin, quantity, price, urgency, notes, reasoning), positive and negative classification guidelines, and four worked examples.
The subtle content is taxonomy discipline: a Tesla convertible is fixed_income despite the equity underlier, a CDS is a derivative despite naming a stock and being a credit product, and indirect phrasing ("let me know if you're a seller") must be inverted to infer the sender's own direction.

**Bundled resources.**
`examples/` only; the skill is intentionally editable by trading domain experts rather than engineers.

**Quality tests (5, prose rubrics).**
q-01 and q-02 are regression checks against the skill's own worked examples 1 and 2.
q-03 is a near-documentation case (firm bid with separate size and price fields and a high-confidence requirement).
q-04 and q-05 are explicitly novel generalisation cases: a two-way CDS market (must land on `two_way` and `derivatives`) and an FX fix order requiring the buy-side inference.

**Baseline behaviour.**
Invocation: 9-10 of 10 across repeats.
Quality is the strongest and most saturated in the corpus:

| test | scores | mean | sd | passed |
|---|---|---|---|---|
| q-01 | 10,9,9,10,10,10,10,9 | 9.62 | 0.48 | 8/8 |
| q-02 | 9,10,10,9,9,9,10,10 | 9.50 | 0.50 | 8/8 |
| q-03 | 10,10,10,10,10,10,10,10 | 10.00 | 0.00 | 8/8 |
| q-04 | 10,10,10,10,10,10,10,10 | 10.00 | 0.00 | 8/8 |
| q-05 | 9,9,10,9,10,10,9,9 | 9.38 | 0.48 | 8/8 |

**Role in the experiments.**
Two tests pinned at 10 on all 8 repeats are the clearest examples of the score-saturation reading of baseline sd = 0.
As the prose-rubric half of the E2 covariate (with slack), its deltas were mildly negative (verbatim grading slightly harsher), and in E1 it is the domain where all three judges agree most (means 9.0-10.0).

---

## 3. jenkins

**What it is.**
A read-only Python client (`JenkinsClient`) for Man Group's Jenkins at manbuild-ci.res.m, the CI server behind Forgejo PRs on code.maninvestments.com.
v1 needs no authentication at all: job metadata, build status, stages, test reports, and logs are exposed anonymously from headnodes, and triggering/controlling builds is explicitly deferred to v2.

**The behaviour it teaches.**
Parse any build URL with `parse_build_url()` into job path and build number; read build state with `get_build()` at basic/detailed/full field granularity; extract Pipeline stages via the wfapi endpoint; list failed JUnit tests; and read logs in four token-capped modes (tail, head, grep with regex and context, byte-range) backed by a shared disk cache at `~/.cache/jenkins_client/`.
The skill's signature move is `get_failure_reason()`, which auto-routes between the test API and the log tail depending on what Jenkins knows, and a strict output contract: one line and stop for SUCCESS, filtered failure details only for FAILURE, never raw JSON dumps or unprompted full logs.
For live builds it prescribes `scripts/monitor_build.py` through the Monitor tool (events: started, progress, stage_change, done) rather than any in-context polling loop.
It pairs with the forgejo skill (which supplies the failing check's target_url) and explicitly excludes TeamCity and Forgejo Actions.

**Bundled resources.**
`jenkins_client/` (client, URL parsing, auth stub) and `scripts/monitor_build.py`.

**Quality tests (4, enumerated rubrics of 7-9 clauses - the longest in the corpus).**
q-01: end-to-end CI triage of a Forgejo PR, requiring the two-skill forgejo-then-jenkins flow.
q-02: a plain status check, mostly testing restraint (SUCCESS means one line and stop) plus correct setup and the no-auth rule.
q-03: log-grep diagnosis when no JUnit XML was published, including the insight that `{published: False}` does not mean tests passed, capped grep usage, and the cache mention.
q-04: live monitoring with correct Monitor arguments, event vocabulary, the failure-diagnosis follow-up, and TaskStop on completion.

**Baseline behaviour.**
Invocation: 10/12 on the first repeat, 12/12 on the other seven.
Quality:

| test | scores | mean | sd | passed |
|---|---|---|---|---|
| q-01 | 9,9,9,9,9,9,9,9 | 9.00 | 0.00 | 8/8 |
| q-02 | 9,9,8,9,9,9,9,9 | 8.88 | 0.33 | 8/8 |
| q-03 | 8,9,8,9,8,8,9,8 | 8.38 | 0.48 | 8/8 |
| q-04 | 9,9,9,8,9,9,9,9 | 8.88 | 0.33 | 8/8 |

**Role in the experiments.**
Its rubrics are the densest in the corpus and two of them exceed the stage-1 cap of 7 generated criteria (q-03 with 8 clauses, q-04 with 9), so jenkins contributes most of the low end of the E3 retention range (0.75-0.86).
Its rubrics also broke the naive "highest bracketed integer" clause-counting rule, because clause text contains build numbers in parentheses; the sequence-based marker rule in `e3_retention.py` exists because of this skill.
Despite the clause loss, its E2 rewrite-vs-verbatim deltas are near zero, which is part of the evidence against the clause-loss-drives-score-change hypothesis.

---

## 4. slack

**What it is.**
Access to a user's Slack data (messages, threads, channels, files, users) through Man Group's Graph API Proxy at graph-proxy.res.m, which holds the user's Slack OAuth token after a one-time Keycloak-SSO connection step.
The agent authenticates with `OidcRequestsFactory(client_id="man-ai-portal")` from `man.requests`, and the skill adapts to its host: in MAIA it uses the native `run_python_code` tool, in Claude Code it rides the python-code-exec skill.

**The behaviour it teaches.**
Search with Slack modifier syntax (`from:`, `in:`, `after:`) through the proxy's search endpoint; download files only via the proxy's `files/{id}/content` route (never `url_private`, which returns an HTML login page) and respect the 10MB limit; fetch image attachments at thumbnail resolution with a full-file fallback; resolve raw Slack user IDs to display names before attributing any message, and say so rather than guess when resolution fails; walk the 401 `slack_not_connected` onboarding flow instead of retrying; and monitor threads or channels with `scripts/poll_slack_thread.py` through the Monitor tool with `persistent=True`, delegating each new reply to a subagent.

**Bundled resources.**
`scripts/poll_slack_thread.py`.

**Quality tests (6, short prose rubrics - the tersest in the corpus, 1-3 sentences each).**
q-01 search modifiers; q-02 file download discipline; q-03 thread images actually described, not just noted; q-04 history summarisation with name resolution first; q-05 the 401 onboarding flow; q-06 thread monitoring with subagent delegation.

**Baseline behaviour.**
Invocation: 10/11 on seven repeats, 9/11 once.
Quality:

| test | scores | mean | sd | passed |
|---|---|---|---|---|
| q-01 | 9,10,9,9,10,10,9,9 | 9.38 | 0.48 | 8/8 |
| q-02 | 9,9,9,9,9,9,9,9 | 9.00 | 0.00 | 8/8 |
| q-03 | 9,9,8,9,9,9,9,9 | 8.88 | 0.33 | 8/8 |
| q-04 | 9,9,9,8,8,9,9,8 | 8.62 | 0.48 | 8/8 |
| q-05 | 9,9,9,9,9,9,9,9 | 9.00 | 0.00 | 8/8 |
| q-06 | 9,9,10,9,9,9,9,9 | 9.12 | 0.33 | 8/8 |

**Role in the experiments.**
The largest single-skill test count (6) and the prose half of the E2 covariate alongside ioi-classifier.
Uniform, comfortably passing scores (8.62-9.38) with no borderline tests; in E1 the judges differ on it only in level (gemini pins all six tests at 10), never in verdict.

---

## 5. web-fetch

**What it is.**
A pure-documentation skill (no bundled code) for fetching web content from anywhere, GET-only, teaching a routing decision across four fetch methods: plain `requests.get()` for unprotected internal domains, Kerberos (`HTTPKerberosAuth` with `mutual_authentication=OPTIONAL` inside a `kinit_using_credentials()` context manager) for 401s on .maninvestments.com/.m domains, Keycloak (`OidcRequestsFactory`) for d.res.m services, and the Delphi scraping service (`ScrapingSession` from `man.delphi`) for every external URL.

**The behaviour it teaches.**
Domain-detection logic over the internal ranges (localhost, RFC1918 blocks, .ahl/.frm/.glg/.m/.maninvestments.com/.ad.man.com/.num/.qarl/.gpm) with a graceful fallback chain; the Delphi approval workflow for external domains (submit at delphi.res.m/scraping/requests, four compliance checkboxes, auto-approval when all boxes tick, retry after approval); and a hard security rule stamped across the skill: never send internal credentials as basic auth, single sign-on flows only.

**Quality tests (4, enumerated rubrics of 7-8 clauses).**
q-01: Kerberos code for a 401 on an internal dashboard, with the basic-auth prohibition.
q-02: external scraping via Delphi including the approval prerequisite.
q-03: a universal fetch function with the full detection and fallback chain.
q-04: diagnosing a Delphi "not approved for scraping" 403 as an approval gap, not a bug, and refusing bypass workarounds.

**Baseline behaviour.**
Invocation: 11-12 of 12 across repeats.
Quality is strikingly uniform - three of four tests sat at raw 9 on every one of the 8 repeats:

| test | scores | mean | sd | passed |
|---|---|---|---|---|
| q-01 | 9,9,9,9,9,9,9,9 | 9.00 | 0.00 | 8/8 |
| q-02 | 9,9,9,9,9,9,9,9 | 9.00 | 0.00 | 8/8 |
| q-03 | 9,9,9,9,9,9,9,9 | 9.00 | 0.00 | 8/8 |
| q-04 | 9,9,9,8,9,9,9,9 | 8.88 | 0.33 | 8/8 |

**Role in the experiments.**
Contributes three of the nine sd = 0 baseline tests and the corpus's most unstable stage-1 case (q-02's generated step count ranged 5-7 across repeats, sd 0.87), showing that criteria generation can wobble even when scores do not.
q-03 is the originally flagged over-cap rubric (8 clauses against the 7-step cap), joined by the two jenkins tests once clause counting was corrected.

---

## How the corpus shaped the experiment designs

The enumerated/prose split (12 vs 11 tests) is the covariate for E2 and the reason E3 is measurable at all.
The three over-cap rubrics (jenkins q-03, q-04, web-fetch q-03) are the cases where stage 1 must lose or merge clauses, giving E3 its forced-compression cases.
atlassian q-02 and q-04 are the only threshold-adjacent tests (baseline means within 0.05 normalised of the 0.7 cutoff) and account for every verdict-level event observed in E1, E2, and C.
The complete non-overlap of the five domains (PR management, trading chat classification, CI triage, Slack access, web fetching) is what gives experiment C's cross-skill arm its force: a rubric that passed a donor answer would have passed an answer about an entirely different system.


# Chapter 5 analysis memo — read this first

Companion to the four cleared documents: (1) invocation baseline CSV, (2) quality baseline CSV, (3) Experiment C report, (4) Experiment E report. Everything below is interpretation of those files or provenance; it contains no data not already in them. Written 2026-08-28, before a week away. Trust this memo over your memory, and where it corrects a sentence inside the reports, trust the memo.

---

## 1. Provenance — FILL THESE IN BEFORE LEAVING (not recorded anywhere else)

- Run model for all baseline runs: claude-sonnet-5[1m] (confirmed yes)
- Judge for all baseline runs: claude-opus-4-8, threshold 0.7, rewrite_rubric true
- **Judge normalisation decision:** slack's eval.yaml originally specified gpt-5-5 as judge; I overrode judge_model to claude-opus-4-8 on every baseline run so the corpus shares one instrument. The original heterogeneity became the motivation for E1.
- **jenkins ran with the forgejo skill passed as `supporting_skills`** because its q-01 rubric requires cross-skill routing. No other skill needed one. Methodological detail for 5.1.
- Baseline run dates: first pass 2026-08-27 16:01–16:06, repeats 2–8 [Verified yes]
- Concurrency used: 10, engine version 2!202608200807+n7582b9fb4546
- **Validation gate outcome for the E harness:** [FILL: paste the verdict — from memory, all 23 within baseline min–max band; web-fetch q-03 showed one −1-point deviation on a σ=0 test, accepted under the tolerance rule as scattered single-point scatter]
- Frozen inputs for C and E: canonical repeat 0 (the unsuffixed JSON per skill)
- Corpus selection: convenience sample — the five skills that had eval.yaml files at the time. Say so in 5.1; it's a respectable frame if named.
- Temperature: the engine requests temperature=0 on judge calls; the man.ai gateway DROPS the parameter for claude-opus-4-8 (Anthropic deprecated it on Opus 4.7+). So the baseline judge sampled at its provider default. gpt-5-5 and gemini-3-1-pro did receive temperature=0 (per-call sampling_params_sent in the E jsonls confirms).

## 2. Status of open items — UPDATE BEFORE LEAVING

- [ ] **Atlassian q-04 anomaly check (see §5): done? outcome?** [FILL]
ran the wrong one, the correct result values are 6-8

- [ ] Author ruling on atlassian neg-02 (real over-trigger bug vs over-strict negative)? [FILL]
Author ruling on atlassian neg-02: RESOLVED — real over-trigger bug. The atlassian skill covers Bitbucket hosts only (mangit/nigit); the org has migrated to Forgejo, so Forgejo requests are out of scope by design and are served by a separate forgejo skill (the jenkins eval already routes to it). The negative is legitimate; the 8/8 trigger is a genuine activation defect whose practical impact grows as Forgejo becomes the default host.

- [ ] Slack jane/john contradiction reported to author / fixed / re-run? [FILL]
- [X] Mutation study: decided [run / dropped]. If dropped, 5.7 must name "no controlled sensitivity study" as a limitation. (DROPPED)
- [ ] Verbosity covariate (len(output) vs score) and stage-1 step-count stability: planned, NOT in the E report — check whether computed. If not, don't hunt for them; list as not done.
- [ ] Hand-coding of clause→criterion mapping (retained/merged/softened/dropped): mine to do, not done as of writing.

## 3. The findings, ranked, with where the evidence sits

**F1. Judge shopping (the centrepiece — E report, per-judge summary table).**
Per-judge grand means: opus 8.80, gpt 9.36, gemini 9.68 — a monotonic ~0.9-point spread in grading standards. sd of those three means ≈ 0.36 points against sd_judge 0.46, so the majority of judge variance is a FIXED OFFSET, not noise. Offsets don't average out with repeats. Since eval.yaml lets the AUTHOR set judge_model against an absolute threshold of 7, switching opus→gemini buys ~0.9 free points: a skill at 6.5 under opus passes under gemini by editing one line. This is the quantified version of the governance complaint ("no guardrails"). Recommendation: pin the judge model platform-side, or calibrate per-judge thresholds. Goes in 5.4 and reorders Ch6 priorities.
*Do not lead with sd_judge/sd_test = 0.48 as the E report does; the offset is the story.*

**F2. The ceiling masks the bias (E report, disagreement section).**
Only 1/23 verdicts disagreed across judges DESPITE the 0.9-point offset — because ~84% of gradings sit at 9–10, three points clear of the cutoff. The naive reading ("96% agreement, judges are interchangeable") is wrong: agreement is an artefact of saturation, and the one threshold-adjacent test (atlassian q-02) is exactly where judges split. On a less saturated corpus the offset would flip verdicts routinely.

**F3. Detector, not meter (C report + quality CSV).**
Floor: all three distractor arms (null, generic, cross-skill) scored EXACTLY raw 1 on all 23 rubrics, sd 0. Mean margin real-vs-best-distractor 7.78 points, min 5. Ceiling: baseline mean 0.89, ~84% of rows at 9–10, only 2/23 tests ever fail. Synthesis: near-perfect gross discrimination, near-zero resolution among competent answers. Also note the bimodality: observed judge outputs are {1} ∪ {~5–10}; the 2–4 range never appears. The judge behaves like a relevance gate followed by a generous grader.
Caveat to state: the distractors were trivially separable (off-topic/empty). Plausible-but-wrong (violating one prohibition) was never tested — that's the untested middle.

**F4. Flip band (quality CSV, flip-rate + distanceFromThreshold columns).**
Every test ≥0.1375 from the threshold: flip rate exactly 0 across all 28 pairwise comparisons. The two tests within 0.05: pairwise flip 0.57 and 0.43 — near coin-toss. Given the 0.1 score grid, "near threshold" = "judge oscillates between raw 6 and 7". Justifies a margin-band CI gate.

**F5. Saturation, not determinism (E report limitations, last bullet).**
The baseline judge had sampling variance available (no temperature control reached Opus) and 9 of 23 tests STILL held sd=0 across 8 repeats. That's score saturation. NEVER claim the judge ran at temperature 0 — earlier drafts of my thinking did, and it's wrong.

**F6. Quantisation critique (derivable).**
Only 10 score values exist (1–10 integer, normalised). pass_threshold is a float — illusory precision: 0.61–0.70 are all the same test. A threshold sweep from the reconstructed histogram is flat near 100% until ~0.8 and still ~84% at 0.9. One figure + one paragraph in 5.2.

**F7. Invocation asymmetry (invocation CSV, recompute in minutes).**
Positives 222/224 rows correct (99.1%); negatives 191/216 (88.4%). 25 of 27 failures are negatives. Mirrors the quality side, where both weak tests (atlassian q-02, q-04) turn on NEGATIVE constraints (don't use Bitbucket MCP for nigit; don't auto-merge). Cross-dimension claim: the system detects required-content-present far better than forbidden-content-absent — and prohibitions are where the safety rules live.

**F8. The Slack contradiction (invocation CSV: slack pos-04 vs neg-05).**
"Who is @jane.doe" = positive, triggers 8/8, passes. "Who is @john.smith" = negative, triggers 8/8, fails. Identical template, deterministic both ways. The EVAL is self-contradictory; the skill is consistent and possibly flawless; one row can never pass. Best single exhibit for the eval-health proposal: a defect in a production eval.yaml, caught by the system's own output. Also: ioi neg-01 starts "hanks for the call" (typo) — second authoring-hygiene datapoint.

**F9. atlassian neg-02 (invocation CSV).**
0/8 — always triggers on a Forgejo-host PR-diff request the author marked out of scope. Deterministic, reproducible. EITHER a real over-triggering bug OR an over-strict negative — depends on the author's ruling (§2). Don't guess in the report.

**F10. Free-pass negatives (invocation CSV, triggeredCount column).**
20 of 27 negatives never fired in any of 8 runs (trivially out of scope: calendar, Fed statement, TeamCity...). They contribute zero information at one subprocess each. Invocation-side twin of C: authoring guidance = negatives should probe the boundary; eval-health could flag never-firing negative sets.

**F11. Borderline cases hold the variance (both CSVs).**
ioi neg-04 ("We bought 200k AAPL yesterday... good fill" — a COMPLETED trade, past tense) sits at 4/8, flip 0.57: the genuine semantic edge case. Parallel to atlassian q-02 on the quality side (~0.5, highest σ). Claim: instability concentrates on genuinely ambiguous items, i.e. the non-determinism is meaningful, not uniform noise. ioi pos-01 is SKILL.md's own worked Example 1 and failed to trigger once in eight — one sentence of gentle irony.

**F12. E2 falsifies half the library's own docs (E report, per-test table).**
Docs claim the rewrite "helps a loose rubric and hurts a precise one". Data: enumerated rubrics, delta ≈ 0 (exactly 0.00 on 10 of 12 once the two threshold-adjacent atlassian tests are set aside) — the rewrite does NOT hurt precise rubrics. Prose rubrics: 4 of 11 moved, ALL in the same direction (skipping the rewrite lowers the score: ioi q-01 −1.00, ioi q-04 −0.67, slack q-01 −0.33, slack q-03 −1.00), zero moved the other way — the rewrite mildly helps loose rubrics. Unidirectionality (4/4) is what makes it signal. Recommendation: KEEP rewrite_rubric=true as default (I earlier suggested the opposite; the data overruled me). Reliability argument for turning it off is dead: both arms sd 0.06.

**F13. E3 is UNDERPOWERED, not falsified — override the report's sentence.**
The E report says "the data does not support the hypothesised causal chain". Don't copy that. The retention=1.0 group is n=4, ALL atlassian, and includes both threshold-adjacent tests; retention spans only 0.75–1.0. Skill identity, threshold proximity and retention are confounded. Honest write-up: the test lacked power and spread; the two large deltas are better explained by threshold proximity. A limitation of the experiment, not a fact about the world.

**F14. Small residue.**
gpt-5-5 at temperature 0 had the LARGEST sd_repeat (0.06 vs 0.02) — the explicitly-greedy judge was least reproducible. 3 repeats, so flag-only. Zero errors across all experiments (184 + 440 + 92 + 345 gradings/rows) — infrastructure claim for 5.1.

## 4. Corrections to sentences INSIDE the cleared reports

1. E report "Variance decomposition (the headline)": the headline is F1 (offsets), not the 0.48 ratio.
2. E report E3 conclusion: replace per F13.
3. C report is sound as written; add the bimodality observation (F3) which it doesn't draw out.

## 5. The q-04 anomaly (E report contains the numbers but never flags them)

atlassian q-04 baseline: mean 7.38, 6/8 passed. In E, under identical config (opus, same criteria path): E1 opus 6.00×3, E2 arm A 6.00×3 — every fresh draw at the baseline's floor, verdict inverted across sessions. Either (a) genuine cross-session irreproducibility — a strong finding — or (b) subtle harness divergence on atlassian rubrics (note q-01 also sat at its baseline min). Discriminator: re-grade q-04 and q-01 with 8 repeats through the harness, compare DISTRIBUTIONS to baseline (~16 calls). Do not write E up confidently without resolving; record the outcome in §2.

## 6. Chapter 5 skeleton (agreed structure)

5.1 Method: the meta-question; measurement-validity spine (reliability / discrimination / floor–ceiling / judge dependence → experiments B, C, A, E); corpus + protocol + provenance once; offline test suite in one paragraph as code-correctness; tool-enabled mode excluded as a DECISION justified by the containment analysis.
5.2 Reliability and dynamic range (A+B): F5, F6, F4, ceiling; figures = score histogram (not means), flip-rate vs distance.
5.3 Discriminative power (C): F3; honest caveat re trivially separable distractors.
5.4 Judge dependence (E): F1, F2, F12, F13, q-04 per §5 outcome.
5.5 What the system found in practice: F8, F9, F10 (+ mutation study if run).
5.6 Synthesis: F7 as the cross-cutting claim; the cheap boolean dimension out-informed the expensive judged one; re-prioritisation — discrimination test = screen for bad rubrics (yield depends on rubric quality), judge pinning = cheap immediate fix, judge v2 = resolution fix. Feeds corrected Ch1.4 + Ch6.
5.7 Limitations: convenience sample, one house style; no controlled sensitivity study (if so); no human agreement — judge consistency ≠ judge correctness, the unvalidated criterion; trivially separable distractors; E1 self-enhancement confound (opus scored opus-authored criteria) + sampling asymmetry (opus has no temperature knob); read-only grades narration; single time point against drifting gateway aliases.
5.8 Ethics/professional: teaching-to-the-test is now concrete (judge shopping); containment analysis grounds the tool-enabled exclusion.
Thread through the chapter: "what does a passing score mean?" — not-null (5.3), under this judge (5.4), away from the cutoff (5.2), on an eval that isn't self-contradictory (5.5).

## 7. Numbers to recompute from the CSVs on return (script them, don't trust memory)

- Quality: mean of per-test means (~0.89); count of σ=0 tests (9); share of rows at raw 9–10 (~84%, stated in E/C reports); threshold sweep.
- Invocation: 222/224 and 191/216; 25/27 failures negative; 20/27 never-fire; per-skill pos/neg accuracy.
- E1 offset share: sd{8.80, 9.36, 9.68} ≈ 0.36 vs sd_judge 0.46.
- Round float noise (0.1625000000000001 → 0.163) on the way into any table.

## 8. Redaction plan reminder

Internal hostnames, repo paths, project keys and the IOI trading content appear throughout the prompts in both CSVs. Consistent pseudonym scheme (S1–S5, host placeholders) decided ONCE and applied to every figure; publish per-skill characterisation tables, not content; describe mutations/edits as operations on structure; any illustrative skill shown in full must be one I wrote (demo skill), not a corpus skill.


The thing all of E is about
Your baseline gave you one number per test, and that number is supposed to be a property of the skill. E asks how much of it is actually a property of the judging apparatus.
To ask that, E freezes the agent completely. You cached 23 outputs; those never change again. Every call in E re-grades the same fixed text. So any score movement you observe cannot be the skill, cannot be the agent, cannot be the prompt. It can only be the judge.
That isolation is the whole design. The baseline measured the pipeline end to end. E takes the last component out and puts it on the bench alone.
What the judge actually is
Two LLM calls in series:
rubric ──[stage 1]──> criteria (3–7 items) ──[stage 2, + output]──> 1–10 scoreStage 1 is a translator: your prose rubric becomes a checklist. Stage 2 is the examiner: it reads the answer against that checklist and gives a mark.
Both stages are failure points, and they fail differently. Stage 1 can lose or soften a requirement, so the examiner is grading the wrong thing. Stage 2 can mark generously or inconsistently, so the examiner is unreliable about the right thing. E1 targets stage 2, E2 targets whether stage 1 should exist at all, E3 targets what stage 1 does to the rubric.
E1 — does the score depend on which model marks it?
Question. If you swap the examiner and change nothing else, does the mark change?
Method. Hold the output fixed (cached from baseline) and hold the criteria fixed (reuse the baseline's evaluationSteps). Then run stage 2 alone under three different models, three repeats each. 207 calls.
The criteria-freezing is the subtle part. With rewrite_rubric: true, one model normally drives both stages, so if Opus and Gemini disagreed you couldn't tell whether they built different checklists or marked the same checklist differently. Freezing the criteria removes stage 1 from the experiment entirely, so any disagreement is attributable to marking alone.
    Held fixed Varied     E1 output, criteria, prompt, rubric stage-2 model, repeat   How it's measured. Three variance components in raw 1–10 points:

sd_test — how much per-test means differ from each other. This is the signal: variation you want, because tests genuinely differ in quality.
sd_judge — how much judges differ from each other on the same test. This is noise from the instrument.
sd_repeat — how much one judge differs from itself on the same test.
Interpretation. If sd_judge approaches or exceeds sd_test, then swapping the judge moves the score about as much as changing the skill does, and the score is not a property of the skill in any useful sense. That's the claim worth making, and it's the sort of thing your Ch2.3 literature discusses but rarely quantifies on a real deployed pipeline.
Alongside it, the pass/fail disagreement rate at 0.7 across the 23 tests. My prediction is disagreement concentrates on atlassian q-02 and q-04, the two tests whose baseline means sat within 0.05 of the cutoff. If it does, E1 confirms your flip-band result from an independent direction: judge identity only matters near the threshold, which is exactly the argument for a margin band on any CI gate.
E2 — is stage 1 helping or hurting?
Question. Your engine defaults to rewriting the rubric before grading. Is that default correct?
Method. One judge (Opus), same fixed outputs, two arms:

Arm A: stage 1 runs, then stage 2 grades against the generated criteria
Arm B: stage 1 skipped, stage 2 grades against your rubric verbatim as a single criterion
Three repeats each, 138 calls.
    Held fixed Varied     E2 output, prompt, rubric, judge model whether stage 1 runs   How it's measured. Per test, delta = mean(B) − mean(A). Then split by rubric style, which is where it gets interesting. Roughly 12 of your rubrics are enumerated (1)...(8) and 11 are prose. Your own docs claim the rewrite helps loose rubrics and hurts precise ones. Nobody has tested it.
The prediction is a crossover: negative delta on prose rubrics (the rewrite adds structure they lacked), positive delta on enumerated ones (the rewrite destroys structure they already had). If that appears, you've found an interaction, which is a much better result than a main effect, and it yields a concrete recommendation: make rewrite_rubric conditional on rubric style rather than a global default.
Also compare σ between arms. Arm B has one stochastic stage instead of two, so if it's tighter, that's a reliability argument for turning the default off independent of any accuracy argument.
E3 — what does stage 1 do to the rubric?
Question. When stage 1 compresses your rubric into criteria, what survives?
Method. No calls at all. evaluationSteps is already in all 40 baseline files. Pure parsing plus hand-coding.
How it's measured. For each enumerated rubric, count the clauses and count the generated criteria. Retention = criteria / clauses. web-fetch q-03 has eight clauses against a hard cap of seven, so at least one requirement provably vanished, and you can name it.
Then the free extra: evaluationSteps exists in all eight repeats, so you can compute the σ of criterion count per test. If the same rubric yields five criteria one run and seven the next, stage 1 is itself non-deterministic, meaning two runs of the same eval are literally grading against different checklists. As far as I know nobody has measured that on a G-Eval-style pipeline.
The hand-coding matters and is yours to do: for each original clause, mark it retained, merged, softened, or dropped. Automating it would be a judgement call disguised as a metric.
The join, which is the actual contribution
E3 gives you a retention ratio per test. E2 gives you a score delta per test. Plot one against the other.
If the tests where stage 1 lost the most clauses are also the tests where skipping stage 1 changed the score most, you have traced a causal chain: rubric specificity → clause loss in stage 1 → measurable score change. That's a mechanistic finding about a two-stage judge, not a restatement of the G-Eval paper, and it's the piece of E most likely to be genuinely novel.
How E fits the rest
Your baseline said the instrument has a ceiling and a narrow flip band. C said the floor holds, so it's a good detector and a poor meter. E explains why by decomposing where the resolution is lost: is it the examiner being generous (E1), the translation step throwing away requirements (E2, E3), or both.
And it does all that for about 350 HTTP calls and no agent runs, which is why it's the cheapest chapter you'll write.Dylan Lim  [1:12 PM]
Results Obtained:
A - we have the csv thing quality baseline and invocation baseline
B - we have the csv thign as well part of it
C - we got the 4 test cases, and thei rresults
E - we got the md fileexplaining all details[1:12 PM]Chapter 5: Evaluation and Reflection — structure
Overall arc: I built a measuring instrument; here is my calibration of it; here is what that calibration revealed; here is what it changes about the design. The reframe from the skeleton is important: this is a measurement study of the evaluator, and the findings are mostly about the instrument, not the skills.

5.1 Method: how do you evaluate an evaluator? (~1.5 pages)
Open with the meta-question, then the measurement-validity framing: reliability, discriminative power, floor/ceiling, judge dependence, each mapped to an experiment. This spine is what stops the chapter reading as a bag of scripts.
Then the setup, once, so no later section repeats it: the five-skill corpus and how it was chosen (convenience sample, say so), 23 quality tests / 55 invocation rows, protocol (8 repeats, judge normalised to Opus, read-only), provenance (engine version, model ids, dates), and the redaction note. Fold the offline test suite into one paragraph here as "correctness of the code, not validity of the scores" — demote it from the skeleton's 5.1.
State the scoping decision: tool-enabled mode built but excluded, justified by your own containment analysis. One paragraph, framed as a decision, not an omission.

5.2 Reliability and dynamic range (~2 pages) — A + B
The instrument's descriptive statistics. Ceiling effect (mean 0.89, ~84% of rows at 9–10, effectively three-valued); the quantisation critique (0.1 grid makes the float threshold illusory, threshold-sweep figure); the flip band (instability confined to ±0.05 of the cutoff, near coin-flip inside it); saturation (σ=0 tests under free sampling — the temperature story goes here, briefly); the invocation contrast (~87% of rows fully deterministic, variance concentrated on semantically borderline prompts, ioi neg-04 as the exhibit).
Key figures: score histogram (not means), flip rate vs threshold distance.

5.3 Discriminative power: detector, not meter (~1.5 pages) — C
Floor established at three null arms, 0/23 each. Then the synthesis sentence the chapter pivots on: near-perfect gross discrimination, near-zero resolution among competent answers — the ceiling in 5.2 isn't the judge saying 9 to everything, it's compression at the top.
Honest paragraph: the null arms were trivially separable; plausible-but-wrong was not tested; this bounds the claim.

5.4 Judge dependence (~2 pages) — E. The centrepiece.
Lead with the offset, not the ratio: monotonic per-judge grading standards spanning ~0.9 points, ~80% of judge variance is fixed bias, which repeats cannot average away. Then the consequence: judge shopping — judge_model is author-settable against an absolute threshold, so one line of YAML buys a pass. Quantified gameability, direct answer to the governance complaint, specific fix (pin the judge platform-side).
Then: why the ceiling masks the bias (1/23 verdict disagreements only because everything sits 3 points clear); the rewrite ablation falsifying half your own docs' claim (neutral on precise rubrics, mildly helpful and unidirectional on prose → keep the default on); E3 reported as underpowered, not as a null result; the q-04 cross-session anomaly, written up according to whichever way your 16-call check resolved it.

5.5 What the system found in practice (~1.5 pages) — the defects
The Slack contradiction (deterministic both ways, eval bug not skill bug, headline pass rate indicts the wrong artifact); atlassian neg-02 with the author's ruling; the free-pass negatives (20/27 never fire); the mutation study here if you ran it, converting "found pre-existing defects" into "detects controlled degradation."

5.6 Synthesis: two distinct failure modes, two distinct fixes (~1 page)
The cross-cutting claims: the negative-constraint asymmetry across both dimensions; the cheap boolean dimension outperforming the expensive judged one; and the re-prioritisation your data forces — the discrimination test is a screen for bad rubrics (yield depends on rubric quality), judge pinning is the cheap immediate fix, judge v2 addresses resolution. This section is what feeds the corrected Ch1.4 and Ch6.

5.7 Limitations and threats to validity (~1 page)
Five skills, convenience-sampled, one house style; no controlled sensitivity study (if you didn't run the mutations); no human agreement, so "the judge is consistent" ≠ "the judge is right" — name this as the unvalidated criterion; null arms trivially separable; E1's self-enhancement confound and the sampling-config asymmetry; read-only grades narration; single point in time against drifting gateway aliases.

5.8 Ethical, social and professional considerations (~1 page)
Keep the skeleton's content, but ground it in your findings now: teaching-to-the-test is concrete once judge shopping is demonstrated; the containment discussion justifies the scoping decision in 5.1.

Two mechanics. Each results section (5.2–5.5) works as claim → evidence → figure → so-what, one paragraph of interpretation per finding rather than a data dump. And thread one question through the chapter — what does a passing score mean? — opening each section by answering another piece of it: it means the answer wasn't null (5.3), under this judge (5.4), on a test that isn't near the cutoff (5.2), against an eval that isn't self-contradictory (5.5).