# FRE Technical Exercise — Crash Refinery by Shape, Not Volume

Your task:

> Stand up a Refinery instance that processes data from the telemetry-gen-and-send
> tool **without** hitting stress relief or being OOMKilled, then modify the
> generated telemetry so that it **does** crash Refinery — **without sending
> (significantly) more events per second than before**.

You are given a fixed, low-resource Refinery and a fixed sender. Keep them (and
Refinery's memory limit) unchanged. Start from the healthy baseline below, then
figure out how to reshape the *generated telemetry* so the same Refinery falls
over at roughly the same event rate.

Everything you send flows to **Honeycomb** so you can watch it: the generated
traces (forwarded by Refinery) and Refinery's own metrics and logs all land in
one environment.

## Prerequisites: a free Honeycomb team

You need a Honeycomb team to send data to. A free team is plenty for this
exercise.

1. Sign up for a free team at <https://www.honeycomb.io/> (free tier).
2. In the team, pick (or create) an **Environment** — e.g. `refinery-exercise`.
3. Create an **Ingest API key** for that environment
   (Environment settings → API Keys → Ingest), with permission to send events.
   Copy the key.
4. Put the key in your `.env`:

   ```bash
   cp .env.TEMPLATE .env
   # edit .env and set HONEYCOMB_API_KEY=<your ingest key>
   ```

This one key is used for everything: the sender presents it so Refinery can
forward the kept traces to your environment, and Refinery uses it to send its
own metrics and logs there too. `docker compose up` will fail fast if it's unset.

> **EU teams:** the config points at the US API (`https://api.honeycomb.io`). If
> your team is on the EU instance, change `Network.HoneycombAPI`,
> `HoneycombLogger.APIHost`, and `OTelMetrics.APIHost` in `refinery-config.yaml`
> to `https://api.eu1.honeycomb.io`.

## What's provided

| File | Role | Change it? |
|------|------|-----------|
| `refinery-config.yaml` | Low-resource Refinery (v2), ~700MB budget, `TraceTimeout: 60s`, no `SpanLimit`, stress relief in monitor mode; ships its own metrics + logs to Honeycomb | **No** |
| `refinery-rules.yaml` | Deterministic sampler | **No** |
| `docker-compose.yaml` | Runs the whole stack: Refinery (capped at 896 MiB) + generator + sender | **No** (keep the Refinery `mem_limit` fixed) |
| `sender-config.yaml` | Fixed sender: 4000 events/sec, reads the generated telemetry | **No** |
| `generator-baseline.yaml` | Well-behaved telemetry — Refinery stays healthy | Your starting point |

The generator writes its templates to a shared volume that the fixed sender
reads — so once the stack is up you just regenerate and re-send.

## Run the baseline

Everything runs in Docker — the generator and sender use the published release
image, so there's nothing to build or install. With your `.env` in place (see
Prerequisites above), from this directory:

```bash
docker compose up
```

This starts the low-resource Refinery, generates the baseline telemetry once, and
sends it at 4000 events/sec. The baseline should run cleanly — no stress relief,
no OOMKill — and the data should appear in your Honeycomb environment.

## Your task

Modify the telemetry that `generator-baseline.yaml` produces so that, sent
through the **unchanged** `sender-config.yaml` at the same ~4000 events/sec, it
drives this same Refinery into stress relief or an OOMKill. The sender's rate
limiter throttles by span count, so you have room to change the *shape* of the
telemetry without changing its *rate*. How you do that is the exercise.

To try a change, edit `generator-baseline.yaml` and re-run the stack:

```bash
docker compose down        # keeps the built image; resets Refinery + volume containers
docker compose up
```

## What you'll see in Honeycomb

In your environment you'll get three streams of data:

- **Generated traces** — forwarded by Refinery into per-service datasets
  (`api-gateway`, `user-service`, …). Confirms Refinery is assembling and
  keeping traces.
- **`Refinery Metrics`** dataset — Refinery's own metrics. Watch
  `memory_heap_allocation` vs `memory_limit`, `collector_cache_eviction`, and
  the stress level. On the baseline these stay calm; a successful attack drives
  memory toward the limit and the stress level up.
- **`Refinery Logs`** dataset — Refinery's operational logs, including
  stress-relief activation and errors.

Also watch the container itself: `docker stats` shows memory climbing toward the
896 MiB cap, and an OOMKill shows up as exit code 137 (`docker compose ps` /
`docker compose logs refinery`). Note that if the container is OOMKilled, the
last few seconds of metrics/logs may not have flushed to Honeycomb — the stress
build-up beforehand is the tell.
