# gd-metrics

Local observability tooling for Godot 4.

Use this addon to collect gameplay timings, gauges, counters, Godot runtime stats, and inspect them through a live terminal dashboard before deciding whether any data should be sent to an external service.

## Installation

### Via gdam

`gdam install @aviorstudio/gd-metrics`

### Manual

Copy `addon/` into `res://addons/@aviorstudio_gd-metrics/` and enable the plugin.

The plugin registers a `GdMetrics` autoload when enabled.

## Quick Start

```gdscript
func _ready() -> void:
	GdMetrics.configure(MetricsModule.MetricsConfig.new(true, 500))

func _run_expensive_step() -> void:
	var timer: int = GdMetrics.begin_timer()
	_expensive_work()
	GdMetrics.finish_timer("CombatService.resolve", timer, {"phase": "combat"})
```

Shared context can be attached once and automatically applied to timers, gauges, counters, logs, and events:

```gdscript
GdMetrics.push_context({"route": "match", "match_type": "pve"})
GdMetrics.increment_counter("BoardInput.hex_clicked")
GdMetrics.pop_context()
```

For multi-step flows, use spans and runtime checkpoints:

```gdscript
var span := GdMetrics.begin_span("Match.startup")
_build_board()
span.checkpoint("board_ready")
_build_hands()
span.finish({"result": "ok"})

GdMetrics.checkpoint_runtime("after_match_ready")
```

## Metric Kinds

- `timer`: retained duration samples in microseconds, summarized with count, avg, p50, p95, p99, max, and latest.
- `gauge`: latest numeric value, useful for current pool sizes, queue depth, node counts, etc.
- `counter`: monotonically accumulated values, useful for events like object pool creates/acquires/returns.

```gdscript
GdMetrics.finish_timer("BoardVisualService.hex_screen_resolve", timer, {"service": "BoardVisualService"})
GdMetrics.set_gauge("ObjectPool.unit.available", 42, "count", {"pool": "unit"})
GdMetrics.increment_counter("ObjectPool.unit.created", 1, "count", {"pool": "unit"})
```

Metric paths identify the value. Tags add low-cardinality grouping and filtering dimensions such as service, phase, scene, route, or pool. Tags are part of metric identity, so the same path with different tags is tracked as separate metric series.

Use tags for stable grouping values. Use fields for high-cardinality details such as IDs, counts, payload sizes, selected card names, or match IDs.

## Logs, Events, And Traces

Logs and events are explicit. The addon does not hook `print()`, `push_warning()`, or `push_error()` output.

```gdscript
GdMetrics.log("warn", "pool exhausted", {"pool": "unit"}, {"available": 0})
GdMetrics.event("match.route_loaded", {"route": "match"}, {"match_id": match_id})
```

Timers automatically become frame trace spans. The live stream emits `frame_trace` messages for frames that contain timer spans.

## Live Terminal UI

Start a local metrics WebSocket server from your game:

```gdscript
func _ready() -> void:
	GdMetrics.configure(MetricsModule.MetricsConfig.new(true, 500))
	GdMetrics.start_live_server(MetricsLiveServer.MetricsLiveServerConfig.new(true, "127.0.0.1", 8765, 250))
```

Then run the terminal UI:

```sh
cd cli
go run . watch --addr ws://127.0.0.1:8765
```

The TUI is mouse-first. Use the top tabs for `Dashboard`, `Logs`, `Events`, and `Traces`; click rows in stream tabs to inspect structured details; use mouse wheel or trackpad scrolling for long views. `ctrl+c` exits the terminal program.

Machine-readable modes:

```sh
cli/bin/gd-metrics snapshot --url ws://127.0.0.1:8765
cli/bin/gd-metrics snapshot --kind timer --path-prefix CardPresentation --sort p95 --limit 20
cli/bin/gd-metrics stream --url ws://127.0.0.1:8765
cli/bin/gd-metrics status --url ws://127.0.0.1:8765
cli/bin/gd-metrics wait --for event --name match.route_loaded --timeout 10s
cli/bin/gd-metrics assert --metric BoardVisualService.hex_screen_resolve --max-p95-usec 16666
cli/bin/gd-metrics assert --budget metrics-budget.json
cli/bin/gd-metrics diagnose --url ws://127.0.0.1:8765
cli/bin/gd-metrics capture --duration 30s --out artifacts/run-001
cli/bin/gd-metrics top timers --file artifacts/run-001/snapshot.json --sort p95 --limit 20
cli/bin/gd-metrics diff before.json after.json
```

The WebSocket protocol is read-only from the CLI perspective. Godot streams `hello`, periodic lightweight `snapshot`, explicit `log`, explicit `event`, and timer-generated `frame_trace` messages. Clients may send `snapshot_request` messages to request filtered read-only snapshots; these do not mutate game state.

Snapshots include custom metrics plus Godot runtime stats from `Performance`, including FPS, frame/process/physics timings, static memory, object/node counts, draw calls, and render memory where supported by the current platform/build. Live snapshots omit raw samples and recent frame traces by default to avoid Godot WebSocket outbound-buffer pressure; request them explicitly with `--raw` or `--traces` when needed.

`capture` writes an agent-friendly artifact bundle:

- `snapshot.json`: filtered snapshot.
- `summary.json`: top timers/counters and runtime summary.
- `diagnose.json`: warnings, tag cardinality, and transport health.
- `status.json`: compact connection/runtime status.
- `metadata.json`: command metadata.
- `summary.md`: human-readable summary.
- `stream.jsonl`: optional live stream when `--duration` is greater than zero.

Budget assertions support JSON files:

```json
{
  "timers": {
    "CardPresentationService.update_display": { "p95_usec": 2000 },
    "HandComponent.update_hand": { "p95_usec": 4000 }
  },
  "runtime": {
    "min_fps": 60,
    "max_frame_usec": 16667,
    "max_orphan_nodes": 400
  }
}
```

## Live Protocol

The WebSocket stream is JSON messages, one object per message. The client is read-only.

`hello`:

```json
{"type":"hello","version":2,"server":"gd-metrics","read_only":true}
```

`snapshot`:

```json
{
  "type": "snapshot",
  "version": 2,
  "metric_count": 3,
  "metrics": [{"kind":"timer","path":"svc.op","tags":{"phase":"load"},"p95":1200}],
  "runtime": {"fps":60.0,"frame_delta_usec":16666},
  "recent_frame_traces": []
}
```

`snapshot_request` from client to server:

```json
{
  "type": "snapshot_request",
  "request_id": "agent-1",
  "options": {
    "kind": "timer",
    "path_prefix": "CardPresentation",
    "sort": "p95",
    "limit": 20,
    "include_raw_samples": false,
    "include_traces": false
  }
}
```

The response is a `snapshot` with the same `request_id`.

`log`:

```json
{"type":"log","level":"warn","message":"pool exhausted","tags":{"pool":"unit"},"fields":{"available":0}}
```

`event`:

```json
{"type":"event","name":"match.route_loaded","internal":false,"tags":{"route":"match"},"fields":{"match_id":"abc"}}
```

`frame_trace`:

```json
{"type":"frame_trace","frame":42,"total_usec":3100,"slow":false,"spans":[{"path":"svc.op","duration_usec":3100,"tags":{}}]}
```

## API Surface

- `GdMetrics`: addon autoload for game integration.
- `MetricsModule`: RefCounted collector for tests or custom wrappers.
- `MetricsRuntimeSampler`: Godot `Performance` monitor sampler.
- `MetricsLiveServer`: localhost WebSocket snapshot server.
- `cli/`: Go Bubble Tea terminal UI.

Useful `GdMetrics` methods include `push_context()`, `pop_context()`, `context_tags()`, `begin_span()`, `checkpoint_runtime()`, `export_snapshot_filtered()`, and `diagnose_metrics()`.

## Notes

- No project settings are required beyond enabling the plugin/autoload.
- Metrics are in-memory only.
- Logs, events, and traces are stream-oriented; connected tools keep their own history.
- The live server is intended for local development/debug builds.

## Testing

`./tests/test.sh`

## License

MIT
