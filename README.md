# gd-metrics

In-memory metrics and timing helpers for Godot 4.

## Installation

### Via gdpm
`gdpm install @aviorstudio/gd-metrics`

### Manual
Copy this directory into `addons/@aviorstudio_gd-metrics/` and enable the plugin.

## Quick Start

```gdscript
const MetricsModule = preload("res://addons/@aviorstudio_gd-metrics/src/metrics_module.gd")

var metrics := MetricsModule.new()
metrics.configure(MetricsModule.MetricsConfig.new(true, 500))
var start_usec: int = metrics.begin_timer(true)
metrics.record("CombatService", "resolve", Time.get_ticks_usec() - start_usec)
```

## API Reference

- `MetricsConfig`: enable flag and max sample limit.
- `record`, `begin_timer`, `finish_void`, `finish_array`: timed capture helpers.
- `get_summary`, `get_all_summaries`: aggregate percentiles and totals.

## Configuration

No project settings are required.

## Testing

`./run_tests.sh`

## License

MIT
