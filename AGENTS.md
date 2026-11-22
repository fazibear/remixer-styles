# Creating ReMixer styles

This repository contains styles only. ReMixer is closed source. Refer users to
https://remixer.fazibear.me for installation, previews and downloads; do not link
to private source repositories or require a ReMixer source checkout.

## Workflow

- Copy an existing style directory to a new directory alongside the others.
  Start with Studio One for compact strips, 90s for segmented meters, Analog for
  photographed controls, Reaper Relic for layered artwork, or Nocturne for flames.
- Keep `style.yaml`, PNGs, local fonts and their licenses together. Change `name`
  and edit settings, fonts, drawing, labels and hit targets in the YAML.
- Put the directory in the user's ReMixer Styles folder and choose **Refresh**.
  A failed reload preserves the previous style; fix the reported YAML/WGSL error.
- Test in ReMixer at narrow/wide widths, supported heights and display scales;
  check clipping, scrolling, pinned master/receive tracks, selection, all controls
  and empty/long FX/send lists. Preview through the website when available.
- Keep source assets here; do not add build output, ZIPs or bundling scripts.
  The website build fetches this repository and bundles each style directory.

## YAML options

```yaml
name: My Style
track_layout: strips
expand_fx: true
min_height: 420
# max_height: 900
fonts:
  body: { family: monospace, size: 14, glow: 0 }
  # title: { file: title.ttf, size: 24 }
settings:
  background: "#101010"
  spacing: 8
strings:
  mute: M
assets:
  knob: knob.png  # Optional alias; include the actual file.
shader: |
  // Adapt the complete entry points from an existing style.
```

- `shader` is required. The other top-level fields are optional; unknown fields
  are rejected. Settings are finite numbers or quoted `#RRGGBB` colors.
- `min_height`/`max_height`: positive logical content heights; minimum <= maximum.
  Equal limits fix the height. These do not set a maximum window width.
- `track_layout: strips`: host positions strips, scrolling and pinned panes.
  `track_layout: shader` (default): WGSL places tracks freely, including horizontal
  rows or custom arrangements. The shader must implement placement and picking.
- `expand_fx: true`: strip content can grow with the viewport; adapt FX rows and
  control placement using the supplied track rectangle height.
- Fonts: required `size` 1–128 logical pixels; optional `file`, `family`, `glow`
  (0–4 logical pixels, default 0). Use local TTF/OTF for portable results; system
  families and generic `monospace`, `sans-serif`, `serif` are also supported.
- Generated constants: `FONT_BODY`, `SETTING_BACKGROUND`, `SETTING_SPACING`,
  `TEXT_MUTE`, `ASSET_KNOB`. Mapping keys start with an ASCII letter and contain
  only ASCII letters, digits or underscores. Avoid case-insensitive collisions.
  IDs are generated: always use names instead of hardcoded font/image/text IDs.
- Settings colors become **linear RGBA** for artwork. Glyphon label ink uses
  **sRGB RGBA** in 0–1; use `label_ink(0xRRGGBBu)` for label colors.

## Drawing, layout and behavior

The host supplies the vertex entry point `vs`, bindings and helpers. Do not
redeclare them. Each embedded shader implements:

- `@fragment fn fs(@builtin(position) pixel: vec4<f32>) -> @location(0) vec4<f32>`:
  draw backgrounds, artwork, meters, controls, indicators and animation. Convert
  physical pixels with `let screen = pixel.xy / frame.viewport.z;`.
- `@compute @workgroup_size(1) fn measure()`: call
  `host_measure(vec3<f32>(strip_width, master_width, content_height))`.
  Widths must be 32–1024 and height 128–4096 logical pixels.
- `@compute @workgroup_size(1) fn emit_labels()`: first call
  `host_begin_layout(vec2<f32>(max_scroll_x, max_scroll_y))`, then emit labels.
- `@compute @workgroup_size(1) fn pick()`: first call `host_clear_hit()`, then
  `host_action(track_index, action_kind, slot, value)` for a hit. Track indices
  and FX/send slots are zero-based. Use the same geometry for drawing and picking.

All graphics may be procedural WGSL, layered PNGs, font labels or a combination.
There is no prescribed widget tree: layouts, shapes, meter segments/flames,
scales, pan/volume controls, header/footer artwork, animation and visual feedback
belong to the style. Native dialogs and supported action meanings remain host
behavior. A style cannot add arbitrary REAPER API calls, shader bindings or
additional render passes via YAML.

Useful supplied helpers:

- `contains(bounds, point)`; bounds are `[x, y, width, height]`.
- `style_image(id, uv)` samples a PNG (nearest texel). Composite its alpha yourself;
  existing styles also define their own `paint`/`artwork` helpers. Use explicit
  interpolation when scaling needs smoother filtering.
- `host_label(track, slot, font, bounds, ink, centered)` and
  `host_label_clipped(..., clip)`; clips are `[left, top, right, bottom]`.
- `host_strip_label(track, slot, font, local_bounds, ink, centered, header_height)`
  offsets by the track rectangle and clips to its pane; `strip_visible(track)`
  tests visibility. Check existing styles for header offsets.
- `label_ink(rgb)` converts a packed RGB integer into label RGBA.

Labels use Glyphon, not bitmap `font.png`. Slots: `TRACK_NAME` 0; FX 1–16
(`FX_NAME + slot`); sends 17–32 (`SEND_NAME + slot`); `GAIN_LABEL` 33;
`PAN_LABEL` 34; `ROUTE_LABEL` 35; `TRACK_INDEX` 36; `METER_L_LABEL`/`METER_R_LABEL`
37/38 (28-cell live bars); `LEVEL_L_LABEL`/`LEVEL_R_LABEL` 39/40 (live dB).
Use generated `TEXT_*` IDs for static YAML captions. Maximum 64 labels per track.
Layout labels are cached until viewport/scroll/track state changes; animate in
`fs`, not by relying on `emit_labels` running each frame. Dedicated meter labels
update from live peaks.

## Host data available in WGSL

- `frame.viewport`: width, height in logical pixels, display scale, track count.
- `frame.mouse`: current x/y, drag starting y, drag starting value.
- `frame.dragging.xyz`: dragged track index + 1 (0 means none), action kind, slot.
  `bitcast<f32>(frame.dragging.w)` is elapsed animation time in seconds (wraps at 4096).
- `frame.scroll`: normal-pane x/y, receive-pane x/y offsets. `frame.panes`: master
  y offset, master-right enabled, receive-tracks-right enabled, reserved.
- `frame.background`, `panel`, `text`, `low`, `mid`, `high`: host color vectors.
- `tracks[i].rect`: x/y/width/height; `.clip`: left/top/right/bottom.
- `.values`: gain dB, pan (-1..1), stereo width, FX-enabled slot bitmask stored
  numerically as f32 (use `u32(values.w)`, not bitcast).
- `.state`: flags, automation mode, monitoring mode, FX count (up to 16).
  Flag bits 0–10: master, selected, mute, solo, phase, record-arm, FX enabled,
  master send, mono, folder parent, explicit track color.
- `.counts`: send count (up to 16), track index, receive count, hardware-output count.
- `.color`: track RGBA. `.peaks`: current left/right dB, held left/right dB.
- Optional send levels: declare `@group(0) @binding(4) var<storage, read>
  send_levels: array<array<f32, 16>>;` for per-track/per-send dB (absent: -150).
  Use host label helpers rather than decoding packed `.labels` manually.

## Supported actions

Pass these kinds to `host_action`; unused slot/value arguments can be 0.

| Kind | Action / value |
| --- | --- |
| 1 (`SET_VOLUME_DB`) | Track gain, -60..12 dB; draggable |
| 2 | Pan, -1..1; draggable |
| 3 / 4 / 5 | Toggle mute / solo / phase |
| 6 / 7 | Toggle record arm / cycle monitoring; non-master |
| 8 / 9 | Open FX chain / routing dialog |
| 10 / 11 / 12 | Input / input FX / envelopes dialogs (10: non-master) |
| 13 | Open FX at slot; Shift toggles that FX's bypass |
| 14 | Send gain at slot, -60..12 dB; draggable |
| 15 (`SELECT_TRACK`) | Select track; host handles selection modifier |
| 16 / 17 | Track context menu / clear held peaks |
| 18 | Toggle master mono; master only |
| 19 | Toggle track FX enabled |
| 20 | Toggle master send; non-master |

## Assets, packaging and checks

- Every PNG directly beside `style.yaml` is included in the GPU atlas. For example,
  `pan-knob.png` generates `IMG_PAN_KNOB`. Different filenames must not normalize
  to the same identifier. Optional `assets` aliases reference these PNGs.
- PNG and font references are plain filenames in the style directory. Keep assets
  self-contained; avoid symlinks and parent/absolute paths. Preserve font licenses.
- Limits: 64 PNGs; 16 MB per file; 4 megapixels per image; 64 MB decoded image data;
  4096×4096 atlas. ZIP packages: 1024 members and 128 MB uncompressed total.
- ZIPs contain `style.yaml` and assets at the top level (one enclosing directory
  is also accepted). ReMixer scans ZIPs in the styles folder; folders with the same
  name take precedence. Gzip alone is not a style package.
- First 128 tracks plus master and first 16 FX/sends per track are exposed.
  Use label helpers for text; legacy packed shader labels are ASCII-only.
- Parenthesize mixed `&&`/`||` expressions. Avoid WGSL reserved words (e.g. `active`),
  duplicate supplied declarations, unbounded loops and mismatched drawing/hitboxes.
- Verify loading, rendering, interaction, resize limits, asset paths and font
  portability in the app. Do not claim runtime verification from YAML inspection.
