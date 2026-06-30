# ZMK Pointer 2-Sensor Mixer

A ZMK module that combines input from two mouse sensors to create cursor and scroll movements. This module calculates
the 3D geometry of sensor positions on a ball and maps their individual movements to a common reference frame.

Inspired by https://github.com/badjeff/zmk-input-processor-mixer

## Installation

Add this module to your ZMK config by including it in your `west.yml`:

```yaml
manifest:
  remotes:
      ...
      - name: efogdev
        url-base: https://github.com/efogdev

  projects:
      ...
      - name: zmk-pointer-2s-mixer
        remote: efogdev
        revision: main
```

## Device Tree Configuration

### 1. Include required headers

```c
#include <dt-bindings/zmk/p2sm.h>
#include <input/processors/p2sm.dtsi>
```

### 2. Define the mixer device

```c
&zip_2s_mixer {
    status = "okay";
	sync-report-ms = <4>; 
	sync-report-yaw-ms = <8>;

	yaw-interference-thres = <24>; // CPI and sync-report-yaw-ms dependent
	yaw-thres = <36>; // CPI and sync-report-yaw-ms dependent

	// arbitrary bounding box (0, 0, 0) - (255, 255, 255)
	ball-radius = <102>; // maximum = 127
	sensor1-pos = [31 4B 2D];  // X Y Z
	sensor2-pos = [C1 3C 2D];
};
```

### 3. Configure input listeners

```c
trackball_primary_listener: trackball_primary_listener {
    compatible = "zmk,input-listener";
    device = <&trackball_primary>;

    listener {
        layers = <DEFAULT>;
        input-processors = <&zip_2s_mixer INPUT_MIXER_SENSOR1>;
        process-next;
    };
};

trackball_secondary_listener: trackball_secondary_listener {
    compatible = "zmk,input-listener";
    device = <&trackball_secondary>;

    listener {
        layers = <DEFAULT>;
        input-processors = <&zip_2s_mixer INPUT_MIXER_SENSOR2>;
        process-next;
    };
};
```

### 4. Define output node

```c
output_node {
    compatible = "zmk,input-listener";
    device = <&zip_2s_mixer>;
};
```

## Example Usage

See the [complete example](https://github.com/efogdev/trackball-zmk-config) in `efogtech_trackball_0.dts` board.

## Kconfig Options

Enable the module in your configuration:

```
CONFIG_ZMK_POINTER_2S_MIXER=y
```

The module is automatically enabled when `CONFIG_ZMK_POINTING=y` is set.

## Shell commands

```
p2sm status                         # show current configuration
p2sm sens pointer       get|set [v] # global cursor sensitivity (v = coef * 1000)
p2sm sens twist         get|set [v] # global scroll sensitivity (v = coef * 1000)
p2sm sens sensor1       get|set [v] # per-sensor cursor gain, sensor 1 (v = gain * 1000)
p2sm sens sensor2       get|set [v] # per-sensor cursor gain, sensor 2 (v = gain * 1000)
p2sm sens twist_sensor1 get|set [v] # per-sensor twist gain, sensor 1 (v = gain * 1000)
p2sm sens twist_sensor2 get|set [v] # per-sensor twist gain, sensor 2 (v = gain * 1000)
p2sm sma  on|off|window set <n>     # moving-average smoothing
```

Per-sensor gain scales one sensor's contribution to the cursor (`sensorN`) or to the
twist/scroll detector (`twist_sensorN`). The two are independent: `sensorN` does not
affect scroll, and `twist_sensorN` does not affect the cursor. Use them to rebalance
two sensors that track a given ball with different effective gain — e.g. a steel ball
one sensor reads "hot" (uneven cursor tracking, or uneven clockwise vs counterclockwise
scroll). Values persist to flash. Boot defaults (percent, 100 = unity):
`CONFIG_POINTER_2S_MIXER_DEFAULT_SENSOR1_GAIN` / `..._DEFAULT_SENSOR2_GAIN` for the
cursor and `..._DEFAULT_TWIST_SENSOR1_GAIN` / `..._DEFAULT_TWIST_SENSOR2_GAIN` for twist.

## Runtime parameters

When `CONFIG_ZMK_RUNTIME_CONFIG` is enabled, the mixer registers tunables under the
`p2sm/` namespace, adjustable live via the `rtcfg` shell (`rtcfg list` to view all keys
with their current/default/range, `rtcfg set <key> <value>` to change). Defaults come
from the corresponding Kconfig options; values are not persisted unless your runtime-config
build saves them. The parameters below cover the twist/scroll smoothing features.

### Direction filter

- `p2sm/dir_filter_en` — enable (`1`) or disable (`0`) the twist direction filter at
  runtime. Range `[0,1]`, default inherits `CONFIG_POINTER_2S_MIXER_DIRECTION_FILTER_EN`.
  - The filter rejects the first frame whose twist direction flips, to suppress
    accidental reverse twists. On a noisy or asymmetric ball it can flip spuriously
    mid-spin, and each flip resets the twist EMA and starts a debounce blackout —
    a common cause of stutter during a continuous spin. Disable it for smoother
    free-spin scrolling at the cost of slightly looser reverse-twist rejection.

### Twist inertia smoothing

Replaces per-tick proportional scroll with a heavily low-pass-filtered scroll *velocity*
that coasts through detection dropouts, so a heavy free-spin produces continuous, smoothly
decaying scroll. Off by default; trades responsiveness and accuracy for smoothness.

- `p2sm/twist_smooth_en` — master on/off. Range `[0,1]`, default `0`.
- `p2sm/twist_smooth_tc` — velocity smoothing time constant (ms); the main smoothness
  dial. Higher = smoother but laggier. Range `[1,2000]`, default `150`.
- `p2sm/twist_coast_tc` — coast decay time constant (ms); how strongly the scroll glides
  through gaps and how long the inertia tail lasts. Higher = longer glide. Range
  `[1,5000]`, default `400`.
- `p2sm/twist_coast_max` — hard cap (ms) on coasting with no twist input before the scroll
  is forced to a stop. Lower = settles sooner when you grab the ball. Range `[50,5000]`,
  default `600`.

Tuning for maximum smoothness: enable it, raise `twist_smooth_tc` until the spin is as
smooth as you like, then raise `twist_coast_tc` / `twist_coast_max` until a heavy spin
glides without stalling. Disabling `p2sm/dir_filter_en` helps, since the smoother no
longer has to recover from direction-filter resets.

### Pointer movement during the scroll tail

- `p2sm/ptr_after_scroll` — milliseconds the pointer stays suppressed after twisting,
  used together with `p2sm/scroll_dis_ptr`. Range `[0,5000]`, default
  `CONFIG_POINTER_2S_MIXER_POINTER_AFTER_SCROLL_ACTIVATION`.
  - With twist smoothing on, pointer suppression keys off the last *real* twist
    detection rather than the last emitted wheel tick, so you can move the cursor while
    the inertia tail is still scrolling. This value then doubles as the delay before the
    pointer re-activates once you stop twisting — lower it (e.g. `20`–`30`) for a snappier
    spin-to-pointer handoff.

Note: scroll output also passes through the downstream `zip_scroll_accel` curve from the
`zmk-acceleration-curves` module, which is managed separately via the `curve` shell command
(`curve dump scroll`, `curve destroy scroll`, `curve monitor`) and is not part of `rtcfg`.
