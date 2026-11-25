# Sample Chapter — After the Test: Reading k6 Output Without Fooling Yourself

*From the Performance Testing Heuristics Pack (in production — see the pre-order page for the full contents).*

---

Your test finished. k6 printed a wall of numbers, some of them green. The natural move is to scan for red, find none that scares you, and write "passed" in the report. That move is how bad results get shipped with confidence. The summary at the end of a k6 run is not lying to you on purpose, but it will let you lie to yourself in specific, predictable ways: a failure rate that reads zero while half your responses were wrong, a checkmark that means almost nothing, an average that looks better than the experience most users actually had, a percentile built on too few requests to mean anything, and a hold period whose first minute was never steady state to begin with. This chapter names those traps, in the order you should read the output, so the verdict you write is the verdict the run actually earned.

## Read checks first. Always.

The single most useful habit in this chapter is an ordering rule: **read the checks line before you read `http_req_failed`.**

The obvious order is the wrong one. Most people look at `http_req_failed` first because it sounds like the failure metric. It is not. `http_req_failed` is a connectivity signal. It tells you whether requests got a response at the network and HTTP level. A server that answers every request with 401 Unauthorized, or 404, or a redirect you did not expect, still reports `http_req_failed: 0.00%`. The server responded. The response was wrong. `http_req_failed` does not care.

Checks care. Checks are the assertions your script wrote: was the status 200, did the body contain the field, did the session token come back. Checks are the only part of the summary that says whether the service did what it was supposed to do. `http_req_failed` tells you the pipe worked. Checks tell you the truth about the application.

So the reading order is:

1. **Checks percentage.** Below 95% is a problem. Below 80% is critical.
2. **Check breakdown by group.** Which scenario or endpoint is failing?
3. **`http_req_failed`.** Now combine the two. Checks failing and `http_req_failed` at 0% means the application is returning wrong answers through a healthy pipe. Checks failing *and* `http_req_failed` above 0% means infrastructure is in the story too.
4. **Latency**, starting with `http_req_duration` p(95) against your SLO or baseline.
5. **VUs.** Was the concurrency realistic? Perfect numbers at 2 VUs are not a result. They are an absence of one.

Everything else in this chapter is a consequence of reading in this order — or of not reading in it.

## Trap 1: "0% failures, half the checks failing — partial pass"

This is the most common misread in k6 output, and it looks like this:

```
✓ http_req_failed................: 0.00%  ✓ 0         ✗ 658
checks...........................: 50.03% ✓ 657       ✗ 656
```

What people conclude: "No failures. Half the checks passed. Partial pass — some scenarios need work."

What is actually happening: the server responded to every single request, and roughly half of those responses were not what the script expected. Wrong status code. Wrong body. Auth error. Redirect. The infrastructure is fine. The application is not. There is no partial credit here, because the half that failed did not fail slowly or partially — it failed completely, on every other request, for the whole run.

The usual causes are mundane: a token that is missing, expired, or wrong (401/403); a wrong endpoint path or the wrong environment (404); a redirect not followed (301/302); or a 200 with the wrong content in the body. None of these show up in `http_req_failed`. All of them show up in checks. If you read `http_req_failed` first and stopped there, you reported a healthy run of a broken service.

Diagnosis move: when checks fail and `http_req_failed` is 0%, log `res.status` and `res.body` temporarily, and look at the `{ expected_response:false }` latency line separately so the failing population does not contaminate the latency of the requests that worked.

## Trap 2: The checkmark that means almost nothing

Look at this line, from a run where more than a third of all requests failed:

```
✓ http_req_failed................: 37.26% ✓ 51801     ✗ 87216
```

There is a checkmark next to a 37.26% failure rate. That checkmark is not k6 being wrong. It is k6 telling you something much narrower than you think: **a threshold passed, or no threshold was defined.** The ✓/✗ symbol on a metric line reports threshold status. It does not report health. If you never defined a threshold for `http_req_failed`, k6 has nothing to judge the value against, and the line wears a ✓ by default while the service falls over.

This trap is dangerous precisely because it survives a quick scan. A wall of output with checkmarks reads as a wall of good news. Train yourself to read the number and ignore the symbol until you have confirmed that a real threshold exists behind it.

The fix is upstream, in the script:

```javascript
thresholds: {
  http_req_failed: ["rate<0.01"],
  http_req_duration: ["p(95)<500"],
  checks: ["rate>0.99"],
}
```

With explicit thresholds, the symbol starts to mean something. Without them, it is decoration.

## Trap 3: When the average is lower than the median, your failures are hiding in it

Normal latency distributions have a long right tail: a minority of slow requests pulls the average *up*, so average sits above median. When you see the reverse — average *below* median — something unusual is in the data:

```
http_req_duration..............: avg=118.35ms  med=173.05ms
  { expected_response:true }...: avg=182.45ms  med=184.48ms
```

What people conclude: "Average response time is 118 milliseconds. Fine."

What is actually happening: there are two populations mixed together — a bimodal distribution. One population is real requests doing real work, centered around 173ms. The other is a large set of very fast requests pulling the average down. Under load, very fast requests are usually failures: the server rejects them in a few milliseconds instead of doing the work. In the run above, 37% of requests failed. The "118ms average" is the average of honest work and instant rejection. It describes neither.

The `{ expected_response:true }` line separates the populations. Successful requests averaged 182ms — over 50% worse than the headline average suggested. That is the number your users experienced.

Rule: whenever avg < med on `http_req_duration`, stop quoting the combined average. Read the `expected_response:true` line for user experience, and go find out why the fast population exists. It is almost never good news.

## Trap 4: A p95 built on almost no data

Percentiles feel precise. p(95) = 35.62ms reads like a measurement. Whether it is one depends entirely on how many requests stand behind it.

A percentile is a rank in a sorted list. With about 45 requests in a window, the p95 is roughly the second-worst value in the whole run — one outlier *is* the percentile. With about 95 requests, it is around the fifth-worst: usable. With 145 or more, it starts to stabilize. The number prints with the same two decimal places in all three cases. k6 does not warn you which case you are in. You have to know your sample size before you quote the percentile.

This bites hardest in two places. First, smoke tests and low-VU steps: a clean p95 at 2 VUs is a percentile over a handful of requests against an unloaded system. It is not evidence about performance; it is evidence the service was reachable. Second, the early steps of a breakpoint test, where each step's sample is smaller than the last steps' and the p95 jumps around for statistical reasons that have nothing to do with the service.

Rule: before you cite a p95, ask how many requests it summarizes. If the answer is in the dozens, treat the percentile as an anecdote, not a measurement — and say so in the report instead of letting the decimal places speak for you.

## Trap 5: Judging the hold by its first minute, and measuring the ramp-down

A hold stage does not start at steady state. When the last virtual user spawns at the end of a ramp, it does not reach its full, rhythmic contribution immediately — depending on whether your script makes its first request immediately or sleeps first, the last VU's contribution settles roughly 58–60 seconds after it spawns. Throughput climbs and wobbles during that first minute. This is the **stabilization window**, and it is transitional data: neither ramp nor steady state.

Two mistakes follow from ignoring it:

**Mistake one: judging the hold by its opening.** If you slice the first minute of a hold and find throughput below target or latency inflated — early requests often pay cold-connection costs — you may be looking at VUs still falling into rhythm, not at the service. The effective measurement window of any hold is the hold minus about a minute. A 10-minute hold gives you roughly 9 minutes of trustworthy data; a 20-minute hold, roughly 19. Short holds are mostly transition, which is one more reason short holds produce numbers that do not reproduce.

**Mistake two: treating ramp-down as measurement.** When a stage ramps down, VUs are de-spawned one by one from a rhythm that already exists. Throughput declines gradually because there are fewer contributors, not because the service changed. Ramp-down is a cool-down. Nothing measured there describes the service at any real load level. Stop measuring when ramp-down begins.

k6 will not protect you from either mistake. Its summary aggregates the whole run — ramp, stabilization, ramp-down tail, all of it — and does not slice by stage. On long holds the contamination is small (about 5% of a 20-minute hold) and can be documented as acceptable noise. On short holds it can be the story. Which brings us to the practical rule for the tool that *can* slice: your APM.

## The measurement-window rule

k6 tells you what happened in aggregate. Your APM, with per-request spans, tells you what happened at a specific step, in a specific window. Use it for per-step evidence, and set the window deliberately:

```
APM query window per step = [hold_start + 90s] → [hold_end]
```

Start the query 90 seconds after the hold begins. That covers the full stabilization window plus a safety margin, so first-iteration and cold-connection requests stay out of your evidence. End the query when the hold ends — which is also where ramp-down begins. Inside that window, the spans are steady state: request counts and p95 from the APM can stand next to the k6 aggregate without the two tools appearing to contradict each other. (They measure different things — k6 from the client side, end to end; the APM from the server side, per request — and most "the tools disagree" arguments are really window and measurement-point arguments. The full pack takes that apart in detail.)

## Two runs, two verdicts

Everything above, applied. These are the two reference runs used throughout this chapter.

**Run A — smoke test, 2 VUs, short run.** The scan-friendly reading: latency is beautiful (p95 of 35.6ms), `http_req_failed` is 0.00%, most lines carry checkmarks.

```
checks...........................: 50.03% ✓ 657       ✗ 656
✓ http_req_failed................: 0.00%  ✓ 0         ✗ 658
http_req_duration................: avg=33.08ms med=30.91ms p(95)=35.62ms
vus..............................: 2
```

Read in the chapter's order, it falls apart in the first line. Checks: 50.03%. One global check — "service is up (status 200)" — failed on all 656 of its attempts while the grouped checks passed, pointing at a specific non-grouped request returning non-200 responses. `http_req_failed` at 0% confirms the server answered everything; it answered wrongly. And the lovely p95 is a percentile over 658 requests from 2 VUs against no load — Trap 4 in a clean shirt.

**Verdict Run A earns:** the service is reachable and responds quickly, and it is returning errors for the request that matters most. This is an application bug found by a smoke test. It is not a passing performance result, and it is not evidence about performance at all.

**Run B — stress test, 7 VUs, one-hour soak.** The scan-friendly reading: checks improved to 62.73%, average latency 118ms, checkmark on the failure line.

```
checks...........................: 62.73% ✓ 174424    ✗ 103601
✓ http_req_failed................: 37.26% ✓ 51801     ✗ 87216
http_req_duration................: avg=118.35ms med=173.05ms p(95)=221.89ms max=14.94s
  { expected_response:true }...: avg=182.45ms med=184.48ms p(95)=232.09ms
```

In order: checks at 62.73% are critical, full stop. The breakdown localizes it — setup, login, and session checks all pass; only the business endpoint degrades, which rules out auth and connectivity and points down the business path. `http_req_failed` at 37.26% (wearing its meaningless ✓, Trap 2) says real failures appear under sustained load that the smoke test never provoked. avg (118ms) below med (173ms) is Trap 3: failed requests returning in milliseconds are dragging the average down, and the successful-request average of 182ms is the honest latency figure. The max of 14.94 seconds — invisible at 2 VUs, where the max was 411ms — says the tail under stress is a different world.

**Verdict Run B earns:** the platform holds (auth, sessions, connectivity) and the business endpoint does not. It degrades under sustained load, fails over a third of its requests, and its real latency is half again worse than the headline average. This run is a clear fail with a clear address — which is exactly what a stress test is for.

Same tool. Same output format. Opposite conclusions from the ones a quick scan produces — in both directions. That is the whole argument for reading in order and knowing the traps by name.

## Quick reference — the checklist card

**Reading order (every run, no exceptions)**

1. Checks % — below 95% is a problem, below 80% is critical
2. Check breakdown by group — which endpoint or scenario
3. `http_req_failed` — combine with checks: checks fail + 0% failed = application error; both fail = infrastructure too
4. The ✓/✗ symbols — only meaningful if you defined explicit thresholds
5. `http_req_duration` p(95) vs SLO/baseline — then avg vs med
6. `http_req_waiting` vs `http_req_receiving` — server-side vs payload/bandwidth
7. VUs and total requests — was there enough load and enough data to conclude anything

**Trap list**

- ☐ 0% `http_req_failed` + failing checks ≠ partial pass. The app is failing through a healthy pipe.
- ☐ ✓ next to a bad number = a threshold passed or none was set. Read the number, not the symbol.
- ☐ avg < med = fast failures are skewing the average. Quote `{ expected_response:true }` instead.
- ☐ p95 over a few dozen requests is an anecdote, not a measurement. Check the sample before citing it.
- ☐ First ~60 seconds of any hold = stabilization. Not steady state. Do not judge the hold by it.
- ☐ Ramp-down = cool-down. Stop measuring when it begins.
- ☐ Clean numbers at 2 VUs = the service was reachable. Nothing more.

**Measurement window (APM, per step)**

```
[hold_start + 90s] → [hold_end]
```

## What the full pack adds

This chapter is one of five. The full pack works the rest of the job in the same register: the decisions *before* the test (which test answers the question you actually have, workload modeling and VU math via Little's Law, and when each test type is the wrong tool); a reporting system for explaining results to leadership — a two-tier report with the verdict first in plain language, acceptance bands, and research-backed frameworks for turning metrics into a decision, labeled as frameworks rather than war stories; enablement — how testing practice actually gets adopted by teams, including cadence policy and per-deploy risk scoring, and why mandates and quality gates fail in predictable ways; and a section on reviewing AI-generated tests and analysis — an eight-point verification checklist, what AI gets wrong in test design, and how to make the AI check its own work so the human keeps the verdict instead of redoing the arithmetic. If this chapter changed how you read your last run, the rest is written for the same reader.
