# AFOP STF DDS Loader for Blender

A Blender addon that transparently decodes **Avatar: Frontiers of Pandora** (Snowdrop engine) `.dds` textures. AFOP ships textures as "STF" container files — a Snowdrop-specific wrapper that Blender's native DDS loader does not recognise. This addon intercepts any STF `.dds` opened through the normal **Open Image** dialog and decodes it silently, so the texture just appears in your material.

---

## Features

- **No import operator.** Open textures via the Material Properties tab, Shader Editor node, or anywhere else in Blender that accepts images — they decode automatically.
- **Deterministic codec selection.** The STF class byte maps to an exact codec via a ground-truth table built from 132,255 converted DDS headers. No guessing.
- **Full format coverage.** Supports BC1, BC2, BC3, BC4, BC5, BC7, and uncompressed RGBA8, BGRA8, RGBA16, RGBA16F, RGBA32F, R8, R16, RG8.
- **No C extensions.** Pure Python with optional numpy acceleration (BC1/BC4/BC5 fully vectorised; BC7 per-block). Works out of the box in any Blender Python environment.
- **Colour space auto-detection.** Normal, mask, roughness, AO, and other data maps are automatically assigned Non-Color; everything else gets sRGB.

> **BC6H (HDR, class `0x49`) is not yet supported.**

---

## Requirements

- Blender 4.0 or later
- numpy (optional, but strongly recommended for performance — already bundled with Blender)

---

## Installation

1. Download `afop_stf_dds.py` from this repository (Releases tab or direct from `main`).

2. In Blender, open **Edit → Preferences → Add-ons → Install…**

3. Navigate to the downloaded file and click **Install Add-on**.

4. Tick the checkbox next to **Import-Export: AFOP STF DDS Loader** to enable it.

   ![Addon enabled in Preferences](docs/prefs_enabled.png)

5. No restart required.

> **Arch Linux / custom Python:** if numpy is not available in Blender's Python, the decoder falls back to a pure-Python path. It is correct but significantly slower on large textures (BC7 @ 2048×2048 takes ~30 s without numpy vs ~1 s with). Install numpy into Blender's Python if needed:
> ```
> /path/to/blender/4.x/python/bin/python3 -m pip install numpy
> ```

---

## Usage

With the addon enabled, **no special workflow is needed.**

1. Select an object and open the **Material Properties** tab.
2. In a material, add an **Image Texture** node (or click the Base Color slot).
3. Click **Open** and browse to an AFOP `.dds` file.
4. The image will appear broken for one frame while the watcher fires, then decode automatically.

The same applies when opening textures in the **Shader Editor** or the **UV Editor**.

### Confirming decode

After a texture loads, you can check the decoded codec and resolution in the image's **Custom Properties** (Properties → Object Data → Custom Properties on the image, or via the Python console):

```python
img = bpy.data.images["your_texture"]
print(img["afop_stf_codec"])   # e.g. "BC7"
print(img["afop_stf_size"])    # e.g. "2048x2048"
```

If the addon printed a `[AFoP STF] decoded ...` line to the system console, the decode succeeded. Errors appear as `[AFoP STF] decode failed ...` with a reason.

---

## How it works

AFOP `.dds` files begin with the four-byte magic `STF\x02` rather than the standard `DDS `. The 76-byte STF header encodes width, height, and a class byte that deterministically identifies the texture's codec. The mip chain is stored smallest-mip-first with mip0 (the full-resolution image) last.

The addon embeds a pure-Python BCn decoder (`afop_bcn`) and a layout resolver (`afop_stf_scan`) which together:

1. Read the class byte and look it up in a ground-truth `CLASS_CODEC` table.
2. Infer the mip-chain layout from the payload size to locate mip0.
3. Decode mip0 to raw RGBA8 pixels.
4. Wrap those pixels in a minimal standard RGBA8 DDS and hand it to `bpy.data.images.load`.
5. Pack the decoded pixel data into the `.blend` and restore the original AFOP filepath on the image data-block.

A `depsgraph_update_post` and `load_post` handler scans `bpy.data.images` for the signature of a failed native load (a `FILE`-source `.dds` image with `has_data == False`) and triggers this pipeline automatically.

---

## Troubleshooting

**Texture loads as solid pink or black**
The addon failed to decode the file. Check the Blender system console (**Window → Toggle System Console** on Windows; launch from terminal on Linux/macOS) for a `[AFoP STF]` error line.

**`[AFoP STF] WARNING: decoder failed to initialise`**
The embedded decoder did not compile. This should not happen under normal circumstances — file an issue with the full error from the console.

**Texture opens but stays black (no error)**
The file may be a standard DDS (not STF) that Blender's own loader also cannot handle, or a BC6H HDR texture (class `0x49`), which is not yet supported.

**Slow decode**
numpy is not available. See the note in [Installation](#installation).

---

## Companion tools

- **AFOP STF DDS Loader for GIMP** (`AFOP_dds_io.py`) — the same decoder packaged as a GIMP 3.x plug-in. Shares the embedded `afop_bcn` and `afop_stf_scan` source.
- **Banshee Brush** — PyQt6 desktop tool for recolouring AFoP Na'vi character assets, uses the same STF decode pipeline.

---

## Licence

MIT. See `LICENSE`.
