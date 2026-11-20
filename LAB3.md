# G2-Meds Lab 3 — Load Testing

We run locally against the Compose stack (no minikube cluster).

## Start the stack

```bash
podman compose up -d
```

## Run the load driver

**Git Bash** (`MSYS_NO_PATHCONV=1` stops Git Bash from rewriting container paths):

```bash
MSYS_NO_PATHCONV=1 PROFILE=smoke ./load/run-local.sh
MSYS_NO_PATHCONV=1 PROFILE=baseline RATE=5 DURATION=2m ./load/run-local.sh
MSYS_NO_PATHCONV=1 PROFILE=spike ./load/run-local.sh
MSYS_NO_PATHCONV=1 PROFILE=newcomers ./load/run-local.sh
MSYS_NO_PATHCONV=1 PROFILE=gate ./load/run-local.sh
```

## Our SLOs / SLIs in `load/slo.json`

```json
{
  "errorRate": 0.01,
  "steps": {
    "notifications_list":        { "p95": 500, "errorRate": 0.01,  "slo": "request-success-rate" },
    "notifications_acknowledge": { "p95": 500, "errorRate": 0.005, "slo": "successfull-acknowledge-rate" }
  }
}
```

| SLO | k6 threshold | Counts |
|---|---|---|
| overall error rate (assignment) | `http_req_failed` < 1 % | all requests, incl. other services |
| request-success-rate 99 % (Lab 2) | `http_req_failed{name:notifications_list}` < 1 % | `GET /notifications/warnings` |
| successfull-acknowledge-rate 99.5 % (Lab 2) | `http_req_failed{name:notifications_acknowledge}` < 0.5 % | `POST /notifications/warnings/{id}/acknowledge` |
| list latency (new in Lab 3) | `http_req_duration{name:notifications_list}` p95 < 500 ms | `GET /notifications/warnings` |
| acknowledge latency (new in Lab 3) | `http_req_duration{name:notifications_acknowledge}` p95 < 500 ms | `POST /notifications/warnings/{id}/acknowledge` |
| scheduler-freshness, evaluation-success (Lab 2) | — | not load-testable (background scheduler, not HTTP) |

In order to evaluate the two non HTTP SLO's `scheduler-freshness` `evaluation-success` after applying load with k6 we tried to set up one-agent for our local podman-network to ingest the data. Therefore we added the missing prometheus monitoring to our service than we claimed in Lab2, in order to have that data in Dynatrace.  

Notes:

- Keys under `steps` are the k6 step names (`name` tags in `script.js`); `slo` is only a label.
- request-success-rate is approximated via `notifications_list`; acknowledge has its own, stricter SLO.
- Lab 2 defined no latency SLO; p95 < 500 ms is our own choice (locally p95 ≈ 5 ms).
- The overall error rate can fail because of another service (see newcomers below).

Changes to the load driver (`load/script.js`):

- New step `notifications_acknowledge`: acknowledges an active warning, as the frontend does. Without it, the acknowledge SLO gets no data.
- `steps.<name>.errorRate`: optional per-step error rate (original only had p95/p99 per step).
- Top-level `errorRate` is optional.

## Results

We did not manage to break the SLOs with k6 load alone, so we injected network faults into the
notification service's container with [netshoot](https://github.com/nicolaka/netshoot) (`tc netem`).

### Break the latency

```bash
# 1. Reference run without latency (20 s, 10 iterations/s, 2 users)
MSYS_NO_PATHCONV=1 PROFILE=baseline RATE=10 DURATION=20s USERS=2 WRITE_RATIO=0.3 ./load/run-local.sh

# 2. Add latency (300 ms ± 100 ms)
MSYS_NO_PATHCONV=1 podman run --rm --network container:fhlva-aidlc-app-health-notification-service-1 --cap-add NET_ADMIN \
  docker.io/nicolaka/netshoot:v0.13 \
  tc qdisc add dev eth0 root netem delay 300ms 100ms

# 3. Check that the rule is active (output should contain "netem ... delay 300ms 100ms")
MSYS_NO_PATHCONV=1 podman run --rm --network container:fhlva-aidlc-app-health-notification-service-1 \
  docker.io/nicolaka/netshoot:v0.13 \
  tc qdisc show dev eth0

# 4. Same run with latency
MSYS_NO_PATHCONV=1 PROFILE=baseline RATE=10 DURATION=20s USERS=2 WRITE_RATIO=0.3 ./load/run-local.sh

# 5. Remove the latency again
MSYS_NO_PATHCONV=1 podman run --rm --network container:fhlva-aidlc-app-health-notification-service-1 --cap-add NET_ADMIN \
  docker.io/nicolaka/netshoot:v0.13 \
  tc qdisc del dev eth0 root
```

#### Run on a healthy network

```text
  █ THRESHOLDS 

    http_req_duration{name:notifications_acknowledge}
    ✓ 'p(95)<500' p(95)=13.06ms

    http_req_duration{name:notifications_list}
    ✓ 'p(95)<500' p(95)=23.07ms

    http_req_failed
    ✓ 'rate<0.01' rate=0.00%

      {name:notifications_acknowledge}
      ✓ 'rate<0.005' rate=0.00%

      {name:notifications_list}
      ✓ 'rate<0.01' rate=0.00%


  █ TOTAL RESULTS 

    checks_total.......: 1070    41.672399/s
    checks_succeeded...: 100.00% 1070 out of 1070
    checks_failed......: 0.00%   0 out of 1070

    ✓ statistics_me 2xx
    ✓ statistics_trend 2xx
    ✓ tracking_list 2xx
    ✓ statistics_global 2xx
    ✓ notifications_list 2xx
    ✓ notifications_acknowledge 2xx
    ✓ tracking_create 2xx

    HTTP
    http_req_duration......................: avg=13.69ms p(50)=7.18ms  p(95)=17.49ms p(99)=25.24ms  max=3.09s   
      { expected_response:true }...........: avg=13.69ms p(50)=7.18ms  p(95)=17.49ms p(99)=25.24ms  max=3.09s   
      { name:notifications_acknowledge }...: avg=10.54ms p(50)=11.77ms p(95)=13.06ms p(99)=13.24ms  max=13.28ms 
      { name:notifications_list }..........: avg=11.19ms p(50)=9.15ms  p(95)=23.07ms p(99)=28.95ms  max=36.07ms 
    http_req_failed........................: 0.00%  0 out of 1078
      { name:notifications_acknowledge }...: 0.00%  0 out of 15
      { name:notifications_list }..........: 0.00%  0 out of 200
    http_reqs..............................: 1078   41.983968/s

    EXECUTION
    iteration_duration.....................: avg=47.49ms p(50)=41.5ms  p(95)=87.94ms p(99)=118.38ms max=123.86ms
    iterations.............................: 200    7.789233/s
    vus....................................: 1      min=0         max=1 
    vus_max................................: 20     min=20        max=20

    NETWORK
    data_received..........................: 1.5 MB 59 kB/s
    data_sent..............................: 848 kB 33 kB/s


running (25.7s), 000/020 VUs, 200 complete and 0 interrupted iterations
baseline ✓ [ 100% ] 000/020 VUs  20s  10.00 iters/s
```

#### Run with added latency

Both latency SLOs fail (p95 1.61 s and 2.36 s); the error rates stay at 0 %.

```text
  █ THRESHOLDS 

    http_req_duration{name:notifications_acknowledge}
    ✗ 'p(95)<500' p(95)=2.36s

    http_req_duration{name:notifications_list}
    ✗ 'p(95)<500' p(95)=1.61s

    http_req_failed
    ✓ 'rate<0.01' rate=0.00%

      {name:notifications_acknowledge}
      ✓ 'rate<0.005' rate=0.00%

      {name:notifications_list}
      ✓ 'rate<0.01' rate=0.00%


  █ TOTAL RESULTS 

    checks_total.......: 1076    42.884317/s
    checks_succeeded...: 100.00% 1076 out of 1076
    checks_failed......: 0.00%   0 out of 1076

    ✓ statistics_me 2xx
    ✓ statistics_trend 2xx
    ✓ tracking_list 2xx
    ✓ statistics_global 2xx
    ✓ tracking_create 2xx
    ✓ notifications_list 2xx
    ✓ notifications_acknowledge 2xx

    HTTP
    http_req_duration......................: avg=250.81ms p(50)=4.04ms p(95)=1.31s p(99)=1.74s max=2.44s
      { expected_response:true }...........: avg=250.81ms p(50)=4.04ms p(95)=1.31s p(99)=1.74s max=2.44s
      { name:notifications_acknowledge }...: avg=1.51s    p(50)=1.43s  p(95)=2.36s p(99)=2.42s max=2.44s
      { name:notifications_list }..........: avg=1.1s     p(50)=1.03s  p(95)=1.61s p(99)=1.7s  max=1.84s
    http_req_failed........................: 0.00%  0 out of 1084
      { name:notifications_acknowledge }...: 0.00%  0 out of 32
      { name:notifications_list }..........: 0.00%  0 out of 196
    http_reqs..............................: 1084   43.203159/s

    EXECUTION
    dropped_iterations.....................: 5      0.199277/s
    iteration_duration.....................: avg=1.36s    p(50)=1.08s  p(95)=3s    p(99)=3.51s max=3.62s
    iterations.............................: 196    7.811641/s
    vus....................................: 1      min=0         max=24
    vus_max................................: 25     min=20        max=25

    NETWORK
    data_received..........................: 1.4 MB 54 kB/s
    data_sent..............................: 855 kB 34 kB/s


running (25.1s), 000/025 VUs, 196 complete and 0 interrupted iterations
baseline ✓ [ 100% ] 000/025 VUs  20s  10.00 iters/s
time="2026-11-19T20:06:16Z" level=error msg="thresholds on metrics 'http_req_duration{name:notifications_acknowledge}, http_req_duration{name:notifications_list}' have been crossed"
```

### Break the error rate

During a single 3-minute run we changed the packet loss on the notification service step by step:
first **20 %**, then **60 %**, and then back to **0 %**. The service was therefore
only affected for a short period of the run, like a temporary network fault.

```bash
# 1. Start the run (3 min, 20 iterations/s, 1 user, every visit writes an entry)
MSYS_NO_PATHCONV=1 PROFILE=baseline RATE=20 DURATION=3m USERS=1 WRITE_RATIO=1 ./load/run-local.sh

# 2. In a second terminal, during the run: add 20 % packet loss
MSYS_NO_PATHCONV=1 podman run --rm --network container:fhlva-aidlc-app-health-notification-service-1 --cap-add NET_ADMIN \
  docker.io/nicolaka/netshoot:v0.13 \
  tc qdisc replace dev eth0 root netem loss 20%

# 3. Check (output should contain "netem ... loss 20%")
MSYS_NO_PATHCONV=1 podman run --rm --network container:fhlva-aidlc-app-health-notification-service-1 \
  docker.io/nicolaka/netshoot:v0.13 \
  tc qdisc show dev eth0

# 4. Increase to 60 % packet loss
MSYS_NO_PATHCONV=1 podman run --rm --network container:fhlva-aidlc-app-health-notification-service-1 --cap-add NET_ADMIN \
  docker.io/nicolaka/netshoot:v0.13 \
  tc qdisc replace dev eth0 root netem loss 60%

# 5. Reset to 0 % (remove the rule) before the run ends
MSYS_NO_PATHCONV=1 podman run --rm --network container:fhlva-aidlc-app-health-notification-service-1 --cap-add NET_ADMIN \
  docker.io/nicolaka/netshoot:v0.13 \
  tc qdisc del dev eth0 root
```

#### Run on a healthy environment

```text
  █ THRESHOLDS 

    http_req_duration{name:notifications_acknowledge}
    ✓ 'p(95)<500' p(95)=9.98ms

    http_req_duration{name:notifications_list}
    ✓ 'p(95)<500' p(95)=7.9ms

    http_req_failed
    ✓ 'rate<0.01' rate=0.00%

      {name:notifications_acknowledge}
      ✓ 'rate<0.005' rate=0.00%

      {name:notifications_list}
      ✓ 'rate<0.01' rate=0.00%


  █ TOTAL RESULTS 

    checks_total.......: 1089    45.966266/s
    checks_succeeded...: 100.00% 1089 out of 1089
    checks_failed......: 0.00%   0 out of 1089

    ✓ statistics_me 2xx
    ✓ statistics_trend 2xx
    ✓ tracking_list 2xx
    ✓ statistics_global 2xx
    ✓ notifications_list 2xx
    ✓ notifications_acknowledge 2xx
    ✓ tracking_create 2xx

    HTTP
    http_req_duration......................: avg=7.72ms  p(50)=4.14ms  p(95)=7.4ms   p(99)=10.62ms max=1.76s  
      { expected_response:true }...........: avg=7.72ms  p(50)=4.14ms  p(95)=7.4ms   p(99)=10.62ms max=1.76s  
      { name:notifications_acknowledge }...: avg=7.45ms  p(50)=7.06ms  p(95)=9.98ms  p(99)=15.64ms max=17.06ms
      { name:notifications_list }..........: avg=5.38ms  p(50)=5.02ms  p(95)=7.9ms   p(99)=10.11ms max=12.62ms
    http_req_failed........................: 0.00%  0 out of 1097
      { name:notifications_acknowledge }...: 0.00%  0 out of 18
      { name:notifications_list }..........: 0.00%  0 out of 201
    http_reqs..............................: 1097   46.303943/s

    EXECUTION
    iteration_duration.....................: avg=25.69ms p(50)=24.13ms p(95)=38.21ms p(99)=42.2ms  max=60.64ms
    iterations.............................: 201    8.484132/s
    vus....................................: 1      min=0         max=1 
    vus_max................................: 20     min=20        max=20

    NETWORK
    data_received..........................: 1.6 MB 68 kB/s
    data_sent..............................: 864 kB 37 kB/s


running (23.7s), 000/020 VUs, 201 complete and 0 interrupted iterations
baseline ✓ [ 100% ] 000/020 VUs  20s  10.00 iters/s
```

#### Run with packet loss (20 % → 60 % → 0 %)

Both per-step error-rate SLOs fail (list 2.28 %, acknowledge 9.15 %). The overall error rate still passes (0.44 %).

```text
  █ THRESHOLDS 

    http_req_duration{name:notifications_acknowledge}
    ✗ 'p(95)<500' p(95)=10s

    http_req_duration{name:notifications_list}
    ✗ 'p(95)<500' p(95)=4.12s

    http_req_failed
    ✓ 'rate<0.01' rate=0.44%

      {name:notifications_acknowledge}
      ✗ 'rate<0.005' rate=9.15%

      {name:notifications_list}
      ✗ 'rate<0.01' rate=2.28%


  █ TOTAL RESULTS 

    checks_total.......: 21112  116.501836/s
    checks_succeeded...: 99.55% 21019 out of 21112
    checks_failed......: 0.44%  93 out of 21112

    ✓ statistics_me 2xx
    ✓ statistics_trend 2xx
    ✓ tracking_list 2xx
    ✓ tracking_create 2xx
    ✓ statistics_global 2xx
    ✗ notifications_list 2xx
      ↳  97% — ✓ 3415 / ✗ 80
    ✗ notifications_acknowledge 2xx
      ↳  90% — ✓ 129 / ✗ 13

    HTTP
    http_req_duration......................: avg=139.28ms p(50)=2.17ms   p(95)=420.69ms p(99)=3.73s  max=10s   
      { expected_response:true }...........: avg=95.65ms  p(50)=2.17ms   p(95)=414.81ms p(99)=2.52s  max=9.94s 
      { name:notifications_acknowledge }...: avg=1.56s    p(50)=216.96ms p(95)=10s      p(99)=10s    max=10s   
      { name:notifications_list }..........: avg=766.54ms p(50)=3.31ms   p(95)=4.12s    p(99)=10s    max=10s   
    http_req_failed........................: 0.44% 93 out of 21116
      { name:notifications_acknowledge }...: 9.15% 13 out of 142
      { name:notifications_list }..........: 2.28% 80 out of 3495
    http_reqs..............................: 21116 116.523909/s

    EXECUTION
    dropped_iterations.....................: 106   0.584937/s
    iteration_duration.....................: avg=842.53ms p(50)=20.33ms  p(95)=4.72s    p(99)=10.01s max=14.56s
    iterations.............................: 3495  19.286373/s
    vus....................................: 0     min=0           max=140
    vus_max................................: 146   min=40          max=146

    NETWORK
    data_received..........................: 41 MB 228 kB/s
    data_sent..............................: 17 MB 93 kB/s


running (3m01.2s), 000/146 VUs, 3495 complete and 0 interrupted iterations
baseline ✓ [ 100% ] 000/146 VUs  3m0s  20.00 iters/s
time="2026-11-19T20:30:44Z" level=error msg="thresholds on metrics 'http_req_duration{name:notifications_acknowledge}, http_req_duration{name:notifications_list}, http_req_failed{name:notifications_acknowledge}, http_req_failed{name:notifications_list}' have been crossed"
```
