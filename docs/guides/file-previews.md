# File Previews

BDrive previews many file types directly in the browser, so you can check what a
file contains without downloading it. Click a file in the browser to open the
preview modal; the arrows in its header move between previewable files in the
same folder.

## Supported formats

| Type | Extensions | Notes |
|------|-----------|-------|
| Images | `jpg`, `jpeg`, `png`, `gif`, `webp` | Scaled to fit |
| Photoshop | `psd` | Layer toggling — see below |
| PDF | `pdf` | |
| Office | `doc`, `docx`, `ppt`, `pptx`, `xls`, `xlsx` | Rendered by an external viewer |
| Video | `mp4`, `mkv`, `mov`, `webm`, `avi`, `flv`, `wmv`, `m3u8` | Streamed with range requests |
| Audio | `mp3`, `m4a`, `aac`, `wav`, `ogg`, `oga`, `opus`, `flac` | |
| Code & text | `c`, `cpp`, `cs`, `css`, `diff`, `html`, `java`, `js`, `jsx`, `json`, `log`, `py`, `rs`, `sh`, `srt`, `toml`, `ts`, `tsx`, `txt`, `vtt`, `vue`, `yaml`, `yml` | Syntax highlighted |
| E-books | `epub` | |
| Apple Wallet | `pkpass` | Rendered the way Wallet does |
| 3D Models | `fbx`, `gltf`, `glb`, `obj`, `stl`, `ply`, `dae` | Interactive three.js viewer — see below |
| Unity packages | `unitypackage` | Virtual folder browsing with asset extraction |

## Photoshop files

A `.psd` opens showing the flattened image Photoshop saves alongside the layer
data, so the preview appears without waiting for every layer to be decoded.

### Toggling layers

The **layers** button in the bottom-right corner opens a layer list. Each row has
an eye toggle that shows or hides that layer, and the image re-composites as you
click. Groups can be expanded and collapsed, and hiding a group hides everything
inside it.

Hiding a group does not change the individual layers within it — unhide the group
and each child returns to the state it had before. **Reset** returns every layer
to the visibility it had when the file was saved.

Layer data is only decoded when you first open the panel, so files you just want
to look at never pay for it.

### What is and isn't reproduced

The preview is a faithful approximation, not a Photoshop-accurate render. It
honours layer groups, per-layer opacity, layer masks, and clipping layers, along
with the blend modes a browser can express: Normal, Multiply, Screen, Overlay,
Darken, Lighten, Color Burn, Color Dodge, Soft Light, Hard Light, Difference,
Exclusion, Hue, Saturation, Color, and Luminosity.

These are drawn as Normal instead, and the layers panel marks each affected
layer: Linear Burn, Linear Dodge (Add), Vivid Light, Linear Light, Pin Light,
Hard Mix, Subtract, Divide, Darker Color, Lighter Color, and Dissolve.

Layer effects (drop shadows, strokes, glows), vector masks, and adjustment layers
are not rendered. Groups are always composited in isolation, so a layer inside a
group blends against the group rather than the image behind it.

### Limits

Photoshop files can be very large, so previewing is capped to keep a browser tab
responsive:

| Limit | Value | Effect when exceeded |
|-------|-------|----------------------|
| File size | 256 MB | No preview; download the file instead |
| Document size | 64 megapixels (about 8000 × 8000) | No preview |
| Layer count | 250 | Preview works, layer toggling is disabled |
| Decoded layer data | 192 MB | Preview works, layer toggling is disabled |

When layer toggling is unavailable the button is disabled and the reason names
the actual numbers for that file. On devices reporting 4 GB of memory or less,
the layer limits are halved.

Files saved without Photoshop's **Maximize Compatibility** option carry no
flattened image. Those are rebuilt from their layers automatically, which takes
a moment longer. If such a file is also past the layer limits, it can't be
previewed at all.

`.psb` (Large Document Format) files are not supported.

## 3D Models

Clicking a 3D model file opens an interactive viewer built on three.js. The model
loads into a scene with orbit, zoom, and pan controls.

### Supported formats

`fbx`, `gltf`, `glb`, `obj`, `stl`, `ply`, `dae` (Collada).

### Viewer controls

The toolbar above the viewport offers:

| Control | Description |
|---------|-------------|
| **Lit / Unlit / Wireframe** | Switch between standard lighting, flat unlit, or wireframe rendering |
| **Dark / Light / Alpha** | Change the background colour, or set it transparent |
| **Light slider** | Adjust lighting intensity from 0 to 3 |
| **Rotate** | Toggle automatic turntable rotation |
| **Reset** | Return the camera to its initial framing |
| **Fullscreen** | Toggle the viewer to fill the screen |

An info bar below the viewport always shows vertex count, face count, and
bounding-box dimensions.

The model is auto-framed on load so it fits comfortably in the viewport, and a
grid floor is scaled to the model's size for spatial reference.

## Zip files

Zip archives open as virtual folders in the file browser. Click a `.zip` file to
navigate into it as if it were a directory — the same file browser UI shows the
archive's contents, complete with folders you can click into.

### How it works

The server reads the zip's central directory without extracting the full archive,
then serves file listings at each depth. Individual files inside the zip can be
previewed or downloaded: images, PSD files, 3D models, and all other supported
formats work exactly as they would outside the archive.

### Configuration

Zip browsing is enabled by default. Operators can disable it by setting
`files.enableZipBrowsing: false` in the config.

### Limits

| Limit | Value |
|-------|-------|
| Maximum zip size | 500 MB |

Zip files larger than 500 MB are available for download but cannot be browsed.

## Unity packages

`.unitypackage` files open as virtual folders, the same way zip files do. The
server parses the package's internal tar.gz structure and reconstructs the
original file tree from Unity's GUID-based layout, so you see human-readable
paths like `Assets/Textures/wood.png` instead of raw GUIDs.

Double-click a `.unitypackage` to browse it. Individual assets can be previewed
(images, 3D models, etc.) or downloaded using the same viewers available
elsewhere in the file browser.

### How it works

A Unity package is a gzipped tar archive. Each asset lives under a GUID-named
folder containing a `pathname` file (the real path), an `asset` file (the
payload), and optionally `asset.meta`. The server reads these entries and serves
a reconstructed directory listing.

### Limits

| Limit | Value |
|-------|-------|
| Maximum package size | 500 MB |

Packages larger than 500 MB are available for download but cannot be browsed.
