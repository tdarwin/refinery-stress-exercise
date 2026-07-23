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

## What's provided

| File | Role | Change it? |
|------|------|-----------|
| `refinery-config.yaml` | Low-resource Refinery (v2), ~700MB budget, `TraceTimeout: 60s`, no `SpanLimit`, stress relief in monitor mode | **No** |
| `refinery-rules.yaml` | Deterministic sampler | **No** |
| `docker-compose.yaml` | Runs the whole stack: Refinery (capped at 896 MiB) + generator + sender | **No** (keep the Refinery `mem_limit` fixed) |
| `sender-config.yaml` | Fixed sender: 4000 events/sec, reads the generated telemetry | **No** |
| `generator-baseline.yaml` | Well-behaved telemetry — Refinery stays healthy | Your starting point |

The generator writes its templates to a shared volume that the fixed sender
reads — so once the stack is up you just regenerate and re-send.

## Run the baseline

Everything runs in Docker — the generator and sender use the published release
image, so there's nothing to build or install. From this directory:

```bash
cp .env.TEMPLATE .env   # optional: put your HONEYCOMB_API_KEY in .env (assembly pressure happens regardless)
docker compose up
```

This starts the low-resource Refinery, generates the baseline telemetry once, and
sends it at 4000 events/sec. The baseline should run cleanly — no stress relief,
no OOMKill.

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

## What to watch on Refinery

If `OTelMetrics`/`LegacyMetrics` are enabled, watch `memory_heap_allocation` vs
`memory_limit`, `collector_cache_eviction`, and the stress level. Otherwise watch
the container: `docker stats` for memory climbing toward the limit, and
`docker compose logs refinery` for stress-relief activation or an OOMKill
(exit code 137).
