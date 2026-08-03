# columbo-status-app

Static shell for a private status page. Contains no data: the app fetches its payload from a private repo with a read-only token carried in the viewer's bookmark fragment.

---

## `data.json` — the payload contract

The publisher (`deploy/status/publish_status.py`, on the droplet) writes `data.json`
into the private status repo. This app renders it. The two ship independently, so
**every key below is optional and the app is presence-gated**: a section appears only
when its own data is present, and a field that is missing, `null`, or the wrong type
is left out entirely rather than rendered as `undefined` / `NaN` / an empty box.

The app never branches on `schema` — it renders whatever fields are actually there.
Schema 1 (no additive keys) and schema 2 both render a complete page; a future bump
cannot blank it.

### Schema 1 — shipping today

`schema` `generated_at` `verdict` `reasons` `last_cycle_age_s` `heartbeat_age_s`
`cycles_today` `evaluations_today` `evaluations_total` `settlements_total` `gaps_24h`
`gaps_24h_by_family` `last_evaluation_age_s_by_family` `run_spent` `credit_cap`
`monthly_used` `monthly_remaining` `credits_captured_at` `loop_started_at` `alerts`

Renders the verdict card, the ledger and the field notes. Unchanged in meaning.

### Schema 2 — additive; the sections below light up as these appear

```jsonc
{
  "schema": 2,

  // ---- THE CASE — one entry per family, keyed by family name -------------
  // Render order is sports, weather, econ, then any other key alphabetically.
  "case": {
    "sports": {
      "phase": "in_sample",              // REQUIRED, else the family is skipped
      "window_open": "2026-08-03",       // YYYY-MM-DD (UTC calendar day)
      "window_close": "2026-09-27",
      "out_of_sample_opens": "2026-08-20",
      "day": 2,                          // day number WITHIN the current phase
      "phase_days": 20,                  // length of the current phase, in days
      "v1_count": 0,                     // out-of-sample above-threshold evals
      "v1_target": 200,
      "v1_clusters": 0                   // independent events behind v1_count
    },
    "weather": {
      "phase": "in_sample",
      "window_open": "2026-08-03",       // era-2 day 0; the 60-day window runs from here
      "window_close": "2026-10-01",
      "out_of_sample_opens": "2026-08-21"
    }
  },

  // ---- THE SCORECARD — ordered; rendered as given ------------------------
  "scorecard": [
    { "id": "V1", "label": "Observations",          "state": "counting", "value": "0 / 200" },
    { "id": "V2", "label": "Net edge, 90% CI",      "state": "sealed"   },
    { "id": "V3", "label": "Brier vs market",       "state": "sealed"   },
    { "id": "V4", "label": "Mean convergence",      "state": "sealed"   },
    { "id": "V5", "label": "Outlier concentration", "state": "sealed"   },
    { "id": "V6", "label": "Fillable size",         "state": "diagnostic" },
    { "id": "V7", "label": "Null self-check",       "state": "running"  }
  ],

  // ---- LATEST OBSERVATION — the most recent evaluation record ------------
  "latest_observation": {
    "ticker": "KXMLBGAME-26JUL262210SEALAD-SEA",
    "model_probability": 0.5412,         // 0–1, rendered to 4 dp
    "market_probability": 0.5500,        // 0–1, rendered to 4 dp
    "net_edge_cents": -0.88,             // cents per contract, signed
    "model_id": "power consensus median",
    "age_s": 14,
    "pre_event": true
  },

  // ---- THE LEDGER -------------------------------------------------------
  "streak_days": 8,                      // consecutive days with no missed cycle
  "evaluations_in_window": 11614         // evaluations inside the accumulation window
}
```

### Charts and the intercept rail — additive keys, sections stay dark without them

| Key | Shape | Renders |
|---|---|---|
| `hourly_evaluations_24h` | `{family: [24 ints]}`, UTC hours | SURVEILLANCE — per-family 24h intake lanes |
| `daily_evaluations` | `{family: [["YYYY-MM-DD", n], …]}` (≤35 rows read) | EVIDENCE LOCKER — cumulative area per family |
| `nbp_schedule` | `{init_hours: [1,7,13,19], fetchable_min: 66, envelope_min: 120}` | INTERCEPT SCHEDULE — the radar clock. Rendered ONLY from this key so the rail can never drift from the client's real schedule |
| `case.<fam>.phase = "exempt"` | string | The dashed dormant block (econ) |

The docket (`ON THE DOCKET`) is NOT payload: it reads `milestones.json` from this
repo, same-origin, so it updates with a Pages push and no droplet deploy.

### Field rules

| Field | Type | If absent or malformed |
|---|---|---|
| `case` | object keyed by family | The case section is hidden |
| `case.<fam>.phase` | `not_started` \| `in_sample` \| `out_of_sample` \| `closed` | **Required.** Missing → that family is skipped. An unrecognised value renders the dashed dormant block with the value as its wording, never numbers |
| `case.<fam>.window_open` / `window_close` / `out_of_sample_opens` | `YYYY-MM-DD` | The sentence that needs the date is dropped or falls back to a dateless wording. Rendered as `aug 20` |
| `case.<fam>.day` / `phase_days` | integers ≥ 1 | No `day N / M` label and no tick strip. `day` is clamped into `1..phase_days`. Phases longer than 60 days compress the strip proportionally; the label still states the true numbers |
| `case.<fam>.v1_count` | number | The big counter is omitted |
| `case.<fam>.v1_target` | number | Counter renders without the `/ N needed` half |
| `case.<fam>.v1_clusters` | number | The `· N events` denominator after the counter is omitted |
| `case.<fam>.note` | string | Only used by the dormant block; omitted if absent |
| `scorecard` | array (max 12 rendered) | The scorecard section is hidden |
| `scorecard[].id` / `.label` | strings | Row skipped. The row is one nowrap line at 26rem, so a `label` over 32 characters is clipped with an ellipsis — keep them short |
| `scorecard[].state` | `counting` \| `sealed` \| `diagnostic` \| `running` | Missing/unknown → renders a **sealed** chip. Fail-closed |
| `scorecard[].value` | string or number | **Only read when `state` is `counting`**; renders `–` if absent. See the seal, below |
| `latest_observation` | object | Section hidden. Shown when it has a `ticker` or at least one number; each of the three numbers is an independent column |
| `latest_observation.pre_event` | boolean | Segment omitted (a string `"yes"` is not a boolean and is dropped) |
| `latest_observation.age_s` | number ≥ 0 | Segment omitted |
| `streak_days` | number ≥ 0 | The streak line is hidden |
| `evaluations_in_window` | number | The sub-line under Observations is omitted |

### The seal

**Sealed criteria carry no value field at all.** V2–V5 are pre-registered and must not
be readable before the accumulation window closes; the seal is enforced **at the
publisher**, which simply does not put the number in the file. The app is the second
lock on the same door: for any row whose `state` is not `counting` it does not read
`value`, so a publisher bug cannot leak a sealed number onto the page.

`model_id` is rendered verbatim — send it in the form you want displayed.

All payload text reaches the DOM via `textContent`; nothing in `data.json`
(alerts included) can inject markup.
