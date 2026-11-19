## What `load/profiles.js` does

[load/profiles.js](../../load/profiles.js) is the **load-shape library** for the Beer Tracker k6 driver. It is not run directly — `script.js` imports it to turn three environment variables (`RATE`, `DURATION`, `SPIKE_FACTOR`) into one k6 scenario object with hard safety caps so a mistyped parameter cannot produce an unbounded run.

### 1. Hard caps

| Constant | Value | Purpose |
|---|---|---|
| `MAX_RATE` | 200 iter/s | Upper bound on arrival rate for every profile |
| `MAX_VUS` | 300 | Upper bound on virtual users for every profile |

Any value that exceeds a cap is silently clamped (a warning is logged on the init VU) rather than throwing an error, so the test still runs in a safe range.

### 2. Helper functions

**`parseDuration(value)`** — converts k6-style duration strings (`"30s"`, `"5m"`, `"1h30m"`) into seconds. Throws if the string contains anything that does not match the pattern, ensuring bad input is caught early.

**`capDuration(name, value, capSeconds)`** — calls `parseDuration`, then clamps the result to `capSeconds` and returns the value as a plain `"<n>s"` string ready for a k6 scenario.

**`capNumber(label, value, cap)`** — validates that a number is positive and clamps it to `cap`. Used for `RATE` and the computed spike peak.

### 3. Profiles — `buildScenario(profile, env)`

The main export. Takes a profile name and the k6 `__ENV` object, builds and returns one scenario configuration.

| Profile | k6 executor | Key env vars | What it models |
|---|---|---|---|
| `smoke` | `constant-vus` | `DURATION` (default 30s, cap 120s) | A single VU walks the full journey to verify the script, auth, and all steps work |
| `baseline` | `constant-arrival-rate` | `RATE` (default 2/s), `DURATION` (default 5m, cap 4h) | Steady open-model load — arrival rate is fixed regardless of response time |
| `spike` | `ramping-arrival-rate` | `RATE`, `DURATION` (hold phase, cap 30m), `SPIKE_FACTOR` (default 5×) | Ramps from `RATE` to `RATE × SPIKE_FACTOR` in 30s, holds, then ramps back down |
| `newcomers` | `ramping-arrival-rate` | `RATE`, `DURATION` (hold phase, cap 30m), `SPIKE_FACTOR` | Like spike but runs the `newcomer` scenario — every iteration is a brand-new user hitting cold code paths |
| `gate` | `constant-arrival-rate` | `RATE`, `DURATION` (default 1m, cap 120s) | Short CI/canary check; `script.js` wires its thresholds with `abortOnFail` |

#### VU pre-allocation formula

Pre-allocated VUs = `max(5, rate × 2)`, max VUs = `max(5, rate × 20)` (or `× 10` for `gate`). The idea: pre-allocate enough for a healthy run, allow headroom for bursts, but never exceed `MAX_VUS`.

#### Spike shape

```
rate ──1m── rate ──30s─► peak ──DURATION─► peak ──30s─► rate ──1m── rate
```

#### Newcomers shape

```
rate ──30s─► peak ──DURATION─► peak ──30s─► 0
```

### 4. How `script.js` uses this module

```js
import { buildScenario } from './profiles.js';

export const options = {
  scenarios: { main: buildScenario(__ENV.PROFILE || 'smoke', __ENV) },
  thresholds: { … },
};
```

`profiles.js` owns only the shape; all threshold/SLO logic stays in `script.js`.
