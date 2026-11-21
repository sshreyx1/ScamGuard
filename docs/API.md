# API & decision rules

[← Back to the overview](../README.md) · see [ARCHITECTURE.md](ARCHITECTURE.md) for the system design and [MODELS.md](MODELS.md) for per-model scores and training.

## Gateway API

`POST /v1/utterance`, one line of a call at a time:

```json
{"call_id": "abc-123", "seq": 1, "speaker": "caller", "text": "Transfer the money now", "lang": "en", "t_ms": 22000}
```

| Field | Required | Meaning |
|---|---|---|
| `call_id` | yes | Same value for every line of one call (1-128 characters). |
| `seq` | no (recommended) | Line number within the call: 1, 2, 3... (caller and receiver lines share one count). Echoed back. A caller line resent with a `seq` that was already scored is not scored again (`"duplicate": true`). Lines may arrive out of order. |
| `speaker` | yes | `caller` = the person who rang (scored). Anything else (e.g. `receiver`) is not scored. |
| `text` | yes | What was said in this line (max 10,000 characters). |
| `lang` | no | `en` or `zh` (`zh-SG`, `en-US`... also work). If left out, text with more than 30% Chinese characters goes to the ZH workers, otherwise EN. |
| `t_ms` | no | Milliseconds since the call started, as the phone measured it. Used for the 15 s warm-up; without it the aggregator uses its own clock since the call's first line. A scripted replay sends it so lines sent a second apart still count as spoken 20 s apart. |

Answer for a caller line:

```json
{"call_id": "abc-123", "seq": 1, "scored": true, "duplicate": false, "verdict": "SCAM", "combined_score": 0.80,
 "reasons": [{"signal": "money", "score": 1.0, "text": "Asked for your OTP or money"},
             {"signal": "impersonation", "score": 0.94, "text": "Claims to be police, a bank or the government"}],
 "action": "Hang up now. Call the organisation back on its official number, or ScamShield on 1799.",
 "warming_up": true, "quorum_met": true, "language": "en", "language_source": "phone",
 "scores": {"urgency": 0.47, "money": 1.0, "impersonation": 0.94}, "missing": [],
 "recent_scores": {"urgency": 0.47, "money": 1.0, "impersonation": 0.94}, "fast_score": 0.80, "bert_score": null,
 "bert_status": "skipped", "caller_lines": 1, "latency_ms": 29}
```

- `scores`: this line's score per worker; `null` = that worker didn't answer in time, and it is listed in `missing`.
- `verdict` is for the **whole call so far**, decided by the aggregator from each signal's highest score
  over the call's recent caller lines (`recent_scores`); see [Aggregator](#aggregator) for the rules.
  `reasons` (strongest first) and `action` are empty when the verdict is `SAFE`.
- `warming_up`: the line is in the call's first 15 s, when the aggregator stays quiet unless a score is
  0.9 or higher. `quorum_met: false`: fewer than 2 workers answered this line, so the verdict is
  `UNSURE` with the reason "Not enough data".
- A receiver line returns `"scored": false` with the call's last answer (`null` verdict before the first
  caller line), and no worker is called.
- A resent caller line (same `call_id` + `seq`) returns `"duplicate": true, "scored": false` with the
  call's current answer, and no worker is called. If scoring a line fails (502), its `seq` is forgotten
  so the phone can retry it.
- If no worker answers at all, the line still gets an answer: `UNSURE` "not enough data" (or `SCAM` if
  the call is already red).
- Errors are JSON `{"error": "..."}`: 400 for a bad request, 502 if the aggregator failed (e.g. Redis down).
- `GET /healthz` returns `{"status": "ok", "calls_in_memory": N}`. The older one-shot
  `POST /submit-transcript` (`{"transcript", "language"}`, no call memory) still works and now also
  returns `reasons` and `action`.
- `model_versions` in each answer: the model version each worker that answered used, e.g. `{"urgency": "v5"}`.
  Workers that don't report a version yet are left out.
- `POST /v1/report`: after a call, the user shares it with consent:
  `{"call_id", "kind": "scam"|"fine", "app_verdict", "language": "en"|"zh", "consent": true, "consent_version": 2,
  "lines": [{"seq", "text"}]}` (caller lines, masked on the phone). Forwarded to the internal review service, which
  masks again and stores the lines encrypted. After 30 days it deletes them, except lines a reviewer approved when
  the user agreed to that (`consent_version` 2). 400 without `consent: true`; 503 if `REVIEW_URL` isn't set.

Gateway settings (environment variables; defaults match `docker-compose.yml`):

| Variable | Default | |
|---|---|---|
| `URGENCY_EN_ADDR`, `MONEY_EN_ADDR`, `IMPERSONATION_EN_ADDR`, `URGENCY_ZH_ADDR`, `MONEY_ZH_ADDR`, `IMPERSONATION_ZH_ADDR` | `http://<signal>_<lang>:50051` | Worker addresses. `http://` is added if missing, so `urgency-en:50051` works in Kubernetes. |
| `AGGREGATOR_ADDR` | `http://aggregator:50052` | |
| `WORKER_TIMEOUT_MS` / `AGGREGATOR_TIMEOUT_MS` | `1400` / `550` | Together they keep every answer within 2 s. The aggregator's share includes the BERT second opinion (`BERT_TIMEOUT_MS` 300) + Redis. |
| `BERT_CONTEXT_LINES` | `5` | Caller lines per call kept in the gateway's RAM and sent with each aggregate request for BERT (never logged; forgotten with the call). `0` = never send text. |
| `CALL_TTL_SECS` / `MAX_CALLS` | `1800` / `10000` | How long and how many calls the gateway remembers (seen `seq`s + last answer). |
| `LOG_FORMAT` | plain | `json` for one JSON object per log line. |
| `REVIEW_URL` | (off) | Where `POST /v1/report` is forwarded (`http://review:8090`). Unset = reporting off (503). |

Logs: one `timing` line per worker call (`stage="worker" ... signal="money" ms=6 outcome="ok"`), one per
aggregator call and one per request. Transcript text is never logged. The decision memory lives in Redis
(any gateway or aggregator copy gets the same verdict); the gateway itself only remembers which `seq`s it
has scored and the call's last answer (for receiver lines and resends), so with several gateway replicas,
sticky routing on `call_id` keeps those two working.


## Aggregator

For each caller line the gateway sends that line's scores (only the workers that answered, possibly none),
the `call_id`, `seq` and `t_ms`. The aggregator loads the call from Redis, applies the rules below, saves
it back and returns the verdict, reasons and action. The rules live in
[backend/aggregator/src/policy.rs](../backend/aggregator/src/policy.rs) (with a unit test per rule).

**Timing policy**: how decisions are made, checked in this order:

| # | Rule | Default | Setting |
|---|---|---|---|
| 1 | **Window**: each signal's recent score is its highest over the call's last N caller lines (a scammer says "police", "act now" and "give me the OTP" in different sentences). Receiver lines don't count. | 6 lines | `AGG_WINDOW_LINES` |
| 2 | **Live red alert**: red is provisional. Every move to red is reviewed by BERT; if BERT is down, at least two different signals must be ≥ 0.8. A red alert clears only after BERT disagrees twice in a row. | strong fallback 0.8, 2 disagreements | `AGG_FALLBACK_STRONG_SIGNAL_THRESHOLD`, `AGG_RED_DISAGREEMENTS_TO_CLEAR` |
| 3 | **Quorum**: fewer than N workers answered *this* line → `UNSURE` "Not enough data" (also during the warm-up). | 2 | `AGG_MIN_QUORUM` |
| 4 | **Weighted vote**: combined = Σ wᵢ·sᵢ / Σ wᵢ over the signals that have a recent score, so a missing worker's weight is shared among the others. | ⅓ each | `AGG_W_URGENCY`, `AGG_W_MONEY`, `AGG_W_IMPERSONATION` |
| 5 | **Warm-up**: in the call's first N seconds stay quiet (`SAFE`) unless some recent score is ≥ the override. This beats the money rule too; the scores stay in the window, so they still count on the first line after the warm-up. | 15 s, override 0.9 | `AGG_WARMUP_SECS`, `AGG_WARMUP_OVERRIDE` |
| 6 | **Alarmed**: fast worker score ≥ 0.6 always asks BERT before red. BERT ≥ 0.6 confirms `SCAM`; BERT 0.3–0.6 produces `UNSURE`. | 0.6 | `AGG_SCAM_THRESHOLD`, `AGG_BERT_SCAM_THRESHOLD`, `AGG_BERT_SAFE_THRESHOLD` |
| 7 | **BERT unavailable**: keep the small models' level, but cap the result at `UNSURE` unless two different signals are ≥ 0.8. | 0.8 | `AGG_FALLBACK_STRONG_SIGNAL_THRESHOLD` |
| 8 | **Unsure**: fast score 0.45–0.6 → `UNSURE` and asks BERT. Fast score below 0.45 → `SAFE` unless another rule applies. | 0.45 / 0.6 | `AGG_UNSURE_THRESHOLD`, `AGG_SCAM_THRESHOLD` |
| 9 | Otherwise `SAFE` (resets the streak). | | |
| S | **Spot check** (cascade): every 5th caller line also asks BERT, even if the line looks safe or is in the warm-up, so a scam that keeps the workers quiet still gets checked. Never once red. | every 5 | `AGG_BERT_SPOT_EVERY` (0 = off) |
| M | **Mode** (for comparison): `cascade` is everything above; `fast` never asks BERT; `bert_only` asks BERT on every caller line (lines 1-2 too) and BERT alone decides with the same warm-up, thresholds and streak, no quorum or money rule. BERT down in `bert_only` → `UNSURE` "The scam check did not answer". | cascade | `AGG_MODE` |
| B | **Decision table**: calm (<0.45) → 🟢 `SAFE`; unsure (0.45–0.6) → 🟡 `UNSURE`, ask BERT; unsure/alarmed with BERT scam (≥0.6) → 🔴 `SCAM`; alarmed with BERT unsure (0.3–0.6) or safe (<0.3) → 🟡 `UNSURE`, never straight to green. The live alert can change during the call; the final verdict is separate and belongs to the end-of-call review. | BERT timeout 300 ms | `BERT_ADDR`, `BERT_TIMEOUT_MS`, `AGG_BERT_SAFE_THRESHOLD`, `AGG_BERT_SCAM_THRESHOLD` |
| | **Reasons**: every signal with a recent score ≥ 0.5 when not `SAFE`, strongest first: "Asked for your OTP or money", "Claims to be police, a bank or the government", "Pressuring you to act now". Plus one action: `SCAM` "Hang up now. Call the organisation back on its official number, or ScamShield on 1799.", `UNSURE` "Don't share any codes or transfer money. Hang up and check with the official number." | 0.5 | `AGG_REASON_THRESHOLD` |
| | **Memory expiry**: a call is deleted from Redis this long after its last line. | 1 hour | `AGG_CALL_TTL_SECS` |

A request without `call_id` (the gateway's `/submit-transcript`) is judged alone: no memory, no warm-up,
no streak (combined ≥ 0.6 → `SCAM` straight away); quorum, the money rule and BERT still apply.

The response (and the verdict log) also carries `fast_score` (the workers' vote alone), `bert_score` and
`bert_status` (`ok` / `timeout` / `error` / `skipped` = not needed / `off` / `no_context`).

All settings are environment variables (listed with their defaults in `docker-compose.yml`, later a
Kubernetes ConfigMap): change them and `docker compose up -d aggregator` to tune without rebuilding. An
invalid value (not a number, threshold outside 0-1, unsure above scam, ...) stops the aggregator at
start-up with a message instead of being silently ignored. Also `REDIS_URL` (default `redis://redis:6379`),
`AGGREGATOR_GRPC_ADDR`, `AGGREGATOR_HTTP_ADDR` and `LOG_FORMAT`.

Redis keys:

- `call:<call_id>`: hash with `started_at_ms`, `window` (last N lines' scores), `streak`, live `red`,
  `red_disagreements`, `caller_lines`, `verdict`, `combined`, `version`. Two aggregator copies updating the same call at once
  can't overwrite each other: the save is a Lua script that only writes if `version` is unchanged, and the
  loser re-reads and retries.
- `calllog:<call_id>`: the verdict log, one JSON entry per caller line
  `{call_id, seq, time_ms, verdict, combined, rule, scores, recent}`, never text. The same entry is logged
  as one `verdict` line.

Both expire with the call. If Redis is down, the aggregator answers gRPC `UNAVAILABLE` (the gateway
returns 502) rather than judging a line without its call's memory; `GET /health` on port 8082 reports it.

```bash
docker compose exec redis redis-cli HGETALL call:demo-1
docker compose exec redis redis-cli LRANGE calllog:demo-1 0 -1
```

The gateway and aggregator build with **Rust 1.90** (pinned in their Dockerfiles) and a committed
`Cargo.lock`, so dependency updates can't break the build.

To start only the six workers (no gateway/aggregator):

```bash
docker compose up -d --build urgency_en money_en impersonation_en urgency_zh money_zh impersonation_zh
```

Every worker has the same shape (`server.py`, `predict.py`, `text_norm.py`, `model/`,
`requirements.txt`, `Dockerfile`) and follows the same rules:

- **No model, no start.** A worker without its model exits at start-up instead of
  serving a fake score.
- **gRPC health service** (`grpc.health.v1`): `SERVING` only once the model has loaded.
  The Kubernetes template probes it.
- **Errors are errors.** If scoring fails, the worker returns gRPC `INTERNAL` and the
  gateway leaves that score out; it never sends a made-up score.
- **No transcript text in logs**, only scores.

See [MODELS.md](MODELS.md#worker-models) for each worker's version, test scores and weak spots.

