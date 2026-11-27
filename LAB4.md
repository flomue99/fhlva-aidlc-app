# G2-Meds Lab 4 — Stages, Gate and a Slow Feature
- Mühleder Florian
- Mühleder Jonas
- Bergthaler Florian

## Goal

Take our service's slow feature (`health-notification-service`, flag `FEATURE_LIVE_WARNINGS`) through the
Flux delivery path (staging with k6 gate → prod canary) and find out whether our SLOs and the gate notice it.

Setup: variant A (minikube 6 GiB, staging + prod), Flux follows `main` of our fork.

## The feature

With the flag on, every `GET /notifications/warnings` first evaluates the user live: 7 sequential calls to
`beer-tracking-service` (one per day) plus database writes, then the list. Without it, the scheduler does
this every 5 minutes in the background.

**Prediction:** `notifications_list` gets 4–5× slower (course: 8 → 31 ms). With our Lab 3 SLO
(p95 < 500 ms) the gate would not notice.

## Our SLOs in `load/slo.json`

| SLO | k6 threshold |
|---|---|
| overall error rate | `http_req_failed` < 1 % |
| request-success-rate 99 % | `http_req_failed{name:notifications_list}` < 1 % |
| successfull-acknowledge-rate 99.5 % | `http_req_failed{name:notifications_acknowledge}` < 0.5 % |
| list / acknowledge latency | `http_req_duration{name:…}` p95 < **400 ms** (Lab 3: 500 ms) |

We first tightened latency to 15 ms. That also failed the release **without** the flag (p95 199 ms): the
gate starts right after the rollout, when the JVM pods are still cold. 400 ms lies between the cold
baseline (199 ms) and the feature (538–1110 ms).

## Results

| Run | Flag | `notifications_list` p50 / p95 | Gate |
|---|---|---|---|
| staging, SLO 15 ms | on | 119 ms / 796 ms (1st try 1.11 s) | ✗ rollback, retry ✗, `Stalled` |
| staging, SLO 15 ms (baseline) | off | 17 ms / 199 ms | ✗ (SLO too strict) |
| staging, SLO 400 ms (baseline) | off | 9.7 ms / 20.7 ms, 121 requests | ✓ (after one retry) |
| prod canary, SLO 400 ms | on (canary only) | 313 ms / 538 ms | ✗ rollback, retry ✗, `Stalled` |

All runs: 0 % failed requests. The gate aborts on latency (`abortOnFail`) after 1–4 s.

Gate on the prod canary (traffic goes to the canary Service only):

```text
    http_req_duration{name:notifications_list}
    ✗ 'p(95)<400' p(95)=538.41ms
    http_req_failed
    ✓ 'rate<0.01' rate=0.00%
level=error msg="thresholds on metrics 'http_req_duration{name:notifications_list}' were crossed;
  at least one has abortOnFail enabled, stopping test prematurely"
```

**Canary under prod traffic** (gate off, k6 Job, baseline 2 visits/s, 2 min, ~half of the requests hit the
canary):

| Step | p50 | p95 | p99 | max | errors |
|---|---|---|---|---|---|
| `notifications_list` | 9.6 ms | 207 ms | 983 ms | 4.37 s | 0 / 231 |
| `notifications_acknowledge` | 17 ms | **562 ms ✗** | 684 ms | 706 ms | 0 / 53 |

The median looks like the old version (the stable half). Only the tail shows the slow canary.

## Decision

**Ship the code, not the feature:** prod runs `v4` with `FEATURE_LIVE_WARNINGS` off, canary ended.
The feature is up to 50× slower in the tail, and fresher warnings (now vs. ≤ 5 min) do not justify that.
Next step would be to make it faster: one summary call for 7 days instead of 7 calls, then release again
behind the flag.

## Conclusions

- **Deploy ≠ release:** the same image `v4` passes with the flag off and fails with it on.
- **The gate only knows what it is told:** with our Lab 3 SLO (500 ms) the feature would have passed.
- **An SLO that is too strict blocks every release:** the gate measures cold pods, so 15 ms failed even the
  baseline. The threshold has to be set against the gate's own baseline, not against warm numbers.
- **Median hides a canary, tail shows it:** p50 9.6 ms vs. p95 207 ms / p99 983 ms on mixed traffic.
- **Flux rolls back by itself but retries only once,** then `Stalled`. A new attempt needs a commit to the
  HelmRelease. An SLO change in `load/` alone does not re-run the gate.
