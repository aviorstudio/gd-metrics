# gd-metrics

Game-agnostic metrics/timing helpers for Godot 4.

- Package: `@aviorstudio/gd-metrics`
- Godot: `4.x` (tested on `4.4`)

## Install

Place this folder under `res://addons/<addon-dir>/` (for example `res://addons/@aviorstudio_gd-metrics/`).

- With `gdpm`: install/link into your project's `addons/`.
- Manually: copy or symlink this repo folder into `res://addons/<addon-dir>/`.

## Files

- `plugin.cfg` / `plugin.gd`: editor plugin entry (no runtime behavior).
- `src/metrics_module.gd`: in-memory timing sample recorder.

## Usage

```gdscript
const MetricsModule = preload("res://addons/<addon-dir>/src/metrics_module.gd")

var start := MetricsModule.begin_timer(true)
# ... do work ...
MetricsModule.finish_void(MetricsModule, "MyService", "do_work", start)

var metrics := MetricsModule.get_metrics()
```

## Notes

- Metrics are only recorded when both config.enabled and the per-call flag are true.
- Samples per metric are bounded by `max_samples_per_metric`; older entries are evicted first.
