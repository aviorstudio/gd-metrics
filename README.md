# gd-metrics

Collect local timing and count samples in Godot 4.

Use this addon to measure slow systems, summarize timings, and inspect performance behavior before you decide whether to send anything to a telemetry service.

## Installation

### Via gdam

`gdam install @aviorstudio/gd-metrics`

### Manual

Copy `addon/` into `res://addons/@aviorstudio_gd-metrics/` and enable the plugin.

## Quick Start

```gdscript
const MetricsModule = preload("res://addons/@aviorstudio_gd-metrics/src/metrics_module.gd")

var metrics := MetricsModule.new()
metrics.configure(MetricsModule.MetricsConfig.new(true, 500))

var start_usec: int = metrics.begin_timer(true)
_run_expensive_step()
metrics.record("CombatService", "resolve", Time.get_ticks_usec() - start_usec)

print(metrics.get_summary("CombatService", "resolve"))
```

## What You Get

- `MetricsConfig`: enable sampling and cap retained samples.
- `begin_timer`: start a microsecond timer.
- `record`: add a timing/sample manually.
- `finish_void` / `finish_array`: helper wrappers for timed callables.
- `get_summary` / `get_all_summaries`: totals, averages, and percentile summaries.

## Notes

- No project settings are required.
- Metrics are in-memory only.
- Pair this with `gd-telemetry` if you want to flush summarized data elsewhere.

## Testing

`./tests/test.sh`

## License

MIT
