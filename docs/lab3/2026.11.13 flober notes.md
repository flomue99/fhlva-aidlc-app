## podman compose up


### needed to delete some old files

[root@37ddbed58f8b sre3_g2_beer]# podman compose up
Error: current system boot ID differs from cached boot ID; an unhandled reboot has occurred. Please delete directories "/run/containers/storage" and "/run/libpod" and re-run Podman


### delete old files

rm -rf /run/libpod /run/containers/storage /run/containers/networks


### now docker compose up works

[root@37ddbed58f8b sre3_g2_beer]# podman compose up
>>>> Executing external compose provider "/usr/bin/podman-compose". Please see podman-compose(1) for how to disable this message. <<<<

3d6cab9c2941df2441024a2f02a65ebe1b8916c3cbe7b9527f62627accea8004
...

## What `load/script.js` does

[load/script.js](../../load/script.js) is a **k6 load-test driver** for the Beer Tracker. It replays the frontend's main journey against the public API, measures latency and errors per step, and checks them against the group's SLOs in [load/slo.json](../../load/slo.json).

### 1. Configuration (env variables, passed via `-e`)

- `PROFILE`: `smoke` (default), `baseline`, `spike`, `newcomers`, `gate`
- `BASE_URL` (default `http://localhost:8080`), optionally per service: `AUTH_URL`, `TRACKING_URL`, `STATISTICS_URL`, `NOTIFICATIONS_URL`
- `USERS`: size of the pre-created user pool (default 20, max 500)
- `WRITE_RATIO`: share of visits that also log a beer (default 0.3)
- `REQUEST_TIMEOUT` (default 10s): a slower request counts as failed, so a hanging dependency shows up as error rate instead of a stuck run
- `SLO_FILE` (default `./slo.json`), `RUN_NAME`

### 2. Thresholds = SLOs as k6 pass/fail rules

- **Error rate:** `smoke` must have 0 errors; all other profiles use `http_req_failed rate < SLO.errorRate` (the error budget, currently 1 %)
- **Latency:** each step in `slo.json` gets `p(95)`/`p(99)` thresholds on `http_req_duration{name:<step>}`
- **`gate` profile:** `abortOnFail`, so the run stops early once an SLO is clearly violated (for CI or canary checks)
- **`newcomers` profile:** always-true thresholds, only so the summary lists those steps
- **Placeholders:** `slo.json` still holds placeholder values (p95 1000 ms everywhere, 1 % errors). They need to be replaced with our own SLOs.

### 3. Load shape (from [load/profiles.js](../../load/profiles.js))

| Profile | Executor | What it does |
|---|---|---|
| `smoke` | 1 VU, 30s | Checks that the script, auth and every step work |
| `baseline` | constant-arrival-rate | Steady `RATE` visits/s (open model), default 5m |
| `spike` | ramping-arrival-rate | 1 min at `RATE` → ramp to `RATE × SPIKE_FACTOR` → hold → back down |
| `newcomers` | ramping-arrival-rate | Every iteration is a brand-new user |
| `gate` | constant-arrival-rate, ≤2 min | Short check that aborts when a threshold fails |

Hard caps (200 iterations/s, 300 VUs, max durations per profile) protect against a mistyped parameter.

### 4. Helpers

- Every request gets a `name` tag. Thresholds and the summary are grouped by that tag.
- Every request also gets an `x-dynatrace-test` header (VU, source, test name, script, step), so Dynatrace can tie traces to the load test.
- `register`, `login`, `createEntry` and `expiry` (reads `exp` from the JWT) are small API wrappers.

### 5. `setup()`: user pool outside the measured load

- Registers and logs in `USERS` test users (`load-<runId>-<n>@example.com`).
- Creates one entry per user and calls `/statistics/me` once, so the statistics caches are **warm** when measuring starts.
- Setup requests are tagged `phase: setup` and use their own step names, so they don't count against the per-step latency SLOs. They do still count toward `http_req_failed`.
- A failed registration or login aborts the run, usually because `BASE_URL` is wrong.

### 6. Token handling (`tokenFor`)

- Each VU reuses one pool user's token.
- It renews the token at a random point in the last 20–30 % of the token's life, so renewals are spread out instead of arriving as one burst.
- A `401` forces a renewal on the next iteration.

### 7. Scenarios (the functions VUs run)

**`visit`**: one dashboard and history visit, the way the frontend makes it:

1. `GET /statistics/me`
2. `GET /statistics/me/trend?days=30`
3. `GET /tracking/entries?page=0&size=20`
4. with probability `WRITE_RATIO`: `POST /tracking/entries`
5. `GET /statistics/global`
6. `GET /notifications/warnings`

Each step is checked for 2xx. Only `smoke` sleeps 1s between iterations; the other profiles are paced by their arrival rate.

**`newcomer`**: a new user's first session: register → login → log one beer → first statistics call (`statistics_me_first`). It hits the cold, first-access code paths that the warm pool deliberately avoids.

### Typical call

```bash
k6 run -e PROFILE=baseline -e RATE=5 -e DURATION=5m -e BASE_URL=http://localhost:8080 script.js
```
