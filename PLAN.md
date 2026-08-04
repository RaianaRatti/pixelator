# Pixel Room Converter - Project Plan

## Contents

- [Goal](#goal)
- [Design Principles](#design-principles)
- [Tech Stack](#tech-stack)
- [Repository Layout](#repository-layout)
- [Pipeline](#pipeline)
- [Stage Details](#stage-details)
- [CLI Design](#cli-design)
- [Presets](#presets)
- [Testing Plan](#testing-plan)
- [Milestones](#milestones)
- [Implementation Notes](#implementation-notes)
- [Known Issues](#known-issues)
- [Path to v1.0](#path-to-v10)
- [Future Extensions](#future-extensions)

---

## Goal

Take a real photo of a room or interior and turn it into pixel art with a retro arcade or video game look, in the style of Undertale interiors.

Requirements:

- The user picks the pixel grid dimensions, for example 32x24 or 64x48 blocks
- Colors are chosen per object, not globally, so a busy object like a sofa does not eat most of the color budget and end up noisy instead of flat and blocky
- Optional borders drawn around each detected object, not as a uniform grid. Two objects sitting next to each other share one border line, not two
- Bright, saturated colors
- Everything runs locally with plain code and established libraries. No external APIs, no paid services, no network calls, no downloaded model weights

---

## Design Principles

**Use established segmentation, not hand written region merging.** OpenCV ships classical, decades old computer vision algorithms in `cv2`. These are not machine learning models and they run entirely on the local machine, which keeps the project dependency light and predictable.

**Segment before downscaling.** A 48x32 block grid is far too coarse to find clean object edges. Segmentation runs on the full resolution photo, and the block grid is built afterward from the resulting label map.

**Labels are categories, not values.** A label map cannot be resized with normal image interpolation, since averaging two label IDs produces a meaningless third ID. Downscaling uses a majority vote per block instead.

**Quantize per region, not globally.** Each region gets its own palette drawn from its own pixels, so regions never compete for shared palette slots.

---

## Tech Stack

| Component | Use |
|---|---|
| Python 3.10+ | Runtime |
| Pillow | Image loading, resizing, saving, border drawing, per-region quantization |
| NumPy | Array-based pixel manipulation, majority vote downscaling, color variation math |
| OpenCV (`opencv-python`) | Segmentation via `cv2.pyrMeanShiftFiltering` and `cv2.connectedComponents`. Pillow has no segmentation tools, which is why OpenCV is needed for this step specifically |

No API keys, no runtime internet access, no model weight downloads.

---

## Repository Layout

```
pixel-room-converter/
  PLAN.md
  README.md
  requirements.txt
  src/
    main.py               Entry point, CLI argument parsing
    pipeline.py           Runs the full conversion pipeline in order
    config.py             Centralized default values
    segmentation.py       Full resolution segmentation with cv2
    resize.py             Downscale and upscale, Image.resize and Image.NEAREST
    label_resize.py       Majority vote downscaling of the label map
    color_budget.py       Per-region color count allocation
    palette.py            Per-region quantization via Image.quantize
    border_straighten.py  Least-squares line fitting for boundaries
    color_boost.py        Saturation and contrast via HSV and ImageEnhance
    dither.py             Ordered Bayer dithering, NumPy only
    borders.py            Border drawing at region boundaries via ImageDraw
    light_extract.py      Luminance extraction and reapplication
    presets.py            Predefined style presets
  examples/
    input/                Sample room photos for testing
    output/               Generated results for comparison
  tests/
    test_resize.py
    test_segmentation.py
    test_label_resize.py
    test_color_budget.py
    test_palette.py
    test_pipeline.py
```

---

## Pipeline

| Stage | Step | Module |
|---|---|---|
| 1 | Load the input image | `main.py` |
| 2 | Extract luminance pattern (optional) | `light_extract.py` |
| 3 | Segment the full resolution photo into a region label map | `segmentation.py` |
| 4 | Downscale the color image to the block grid using area averaging | `resize.py` |
| 5 | Downscale the label map to the same grid using majority vote | `label_resize.py` |
| 6 | Merge undersized regions into their nearest neighbor | `pipeline.py` |
| 7 | Straighten jagged region boundaries | `border_straighten.py` |
| 8 | Allocate the color budget across regions by variation and size | `color_budget.py` |
| 9 | Quantize each region independently, then reassemble the grid | `palette.py` |
| 10 | Boost saturation and contrast | `color_boost.py` |
| 11 | Apply ordered dithering (optional) | `dither.py` |
| 12 | Upscale with nearest neighbor to preserve hard edges | `resize.py` |
| 13 | Draw borders at region boundaries (optional) | `borders.py` |
| 14 | Reapply the luminance pattern (optional) | `light_extract.py` |
| 15 | Save the final image | `main.py` |

---

## Stage Details

### Full Resolution Segmentation

`segmentation.py`

1. Resize the input so its longest side is `--segmentation-max-dimension` pixels using `Image.resize` with `Image.LANCZOS`. This is purely for speed, since mean shift filtering is slow on large images and the result gets downscaled to the block grid anyway.
2. Convert from Pillow RGB to OpenCV BGR with `cv2.cvtColor(np.array(image), cv2.COLOR_RGB2BGR)`.
3. Call `cv2.pyrMeanShiftFiltering(src, sp=spatial_radius, sr=color_radius)`. This flattens the image so pixels close in both position and color collapse to one exact matching color, pre-grouping the photo into flat blobs.
4. Find every unique color left in the flattened result with `np.unique(flattened.reshape(-1, 3), axis=0)`.
5. For each unique color, build a binary mask with `np.all(flattened == color, axis=-1).astype(np.uint8)` and run `cv2.connectedComponents(mask, connectivity=4)`. This splits same colored but spatially separate areas into distinct regions, so two different pillows do not merge just because a third same colored object exists elsewhere in the room.
6. Combine the results into one master label array, offsetting label numbers from each call so they stay unique across the image.

Output: a full resolution 2D NumPy array of integer labels, one per pixel, aligned with the resized working image.

### Downscaling

`resize.py`, `label_resize.py`

- Downscale the original color photo to `width x height` blocks using `Image.resize` with `Image.BOX` or `Image.LANCZOS`. This defines the final block colors.
- Downscale the label map to the same grid separately. For each output block, gather the label map pixels inside that block and take the most common label with `np.bincount(block_labels.ravel()).argmax()`.

Output: a `width x height` grid of labels aligned one to one with the block grid of colors.

### Small Region Cleanup

- Count blocks per label using `np.bincount` on the flattened label grid.
- Any region below `--min-region-size` is reassigned to the neighboring region with the closest average color, so tiny slivers left over from mean shift filtering do not become their own object with their own border.

### Color Budget Allocation

`color_budget.py`

- Measure each region's internal color variation as the standard deviation of its original, non quantized block colors.
- Flat regions such as a plain wall or a single colored cushion get a count near `--min-colors-per-region`.
- High variation regions such as a patterned rug or a shadowed corner get more, up to `--max-colors-per-region`.
- Distribute the total `--colors` budget proportionally to variation and region size, so the sum stays close to the requested total rather than ballooning.

### Per-Region Quantization

`palette.py`

- Collect only the block colors belonging to a region, wrap them in a small temporary Pillow image, and call `Image.quantize(colors=assigned_count, method=Image.MEDIANCUT)` on that image alone.
- Map the reduced colors back onto that region's blocks in the main grid.

This is the key improvement over global quantization. A busy sofa with 2 assigned colors gets those 2 colors chosen specifically to represent the sofa's own range, instead of competing with the rest of the room.

### Reassembly

- Walk the block grid and replace each block's color with its region's quantized result.
- Output remains at block grid resolution with per-region flattened colors.

### Saturation and Contrast Boost

`color_boost.py`

- Convert to HSV with `image.convert("HSV")`, split channels, multiply the S channel by `--saturation`, clip to valid range, merge, convert back to RGB.
- Apply `ImageEnhance.Contrast(image).enhance(contrast)` afterward.

### Dithering

`dither.py`

- Apply an ordered Bayer matrix dither as a NumPy operation, either before quantization or blended into it.
- Adds textured retro shading across color transitions inside a region instead of perfectly flat blocks.

### Upscaling

`resize.py`

- Scale the block grid back up with `Image.resize` using `Image.NEAREST` only. Any other resampling method blurs the hard pixel edges.

### Border Drawing

`borders.py`

- Reuse the label grid from the earlier stages. There is no need to redetect boundaries after quantization.
- For every block, compare its label to the block directly to its right and the block directly below.
- If the labels differ, record that edge as a border.
- After upscaling, convert each recorded boundary into pixel coordinates and draw it once with `ImageDraw.Draw(image).line(...)`, using `--border-size` and `--border-color`.

Because each boundary is checked once in two directions rather than four, two touching objects share exactly one line and never a doubled line. With `--borders` off, the pipeline produces plain per-region colored blocks with no outlines.

---

## CLI Design

Basic usage:

```
python src/main.py --input room.jpg --output room_pixel.png --width 48 --height 32
```

### Required

| Flag | Description |
|---|---|
| `--input` | Path to the source image |
| `--output` | Path to save the result |
| `--width` | Number of pixel blocks across |
| `--height` | Number of pixel blocks down. Auto-calculated from width if omitted |

### Color budget

| Flag | Default | Description |
|---|---|---|
| `--colors` | 16 | Total color budget shared across the whole image |
| `--min-colors-per-region` | 1 | Smallest count any single region can be assigned |
| `--max-colors-per-region` | 6 | Largest count any single region can be assigned, so no busy object eats the whole budget |

### Segmentation

| Flag | Default | Description |
|---|---|---|
| `--spatial-radius` | 12 | Passed to `cv2.pyrMeanShiftFiltering` as `sp`. How far apart two pixels can be and still merge |
| `--color-radius` | 20 | Passed as `sr`. How different two colors can be and still merge |
| `--min-region-size` | 2 | Minimum block count after downscaling before a region is merged into its nearest neighbor by color distance. Filters stray single blocks |
| `--segmentation-max-dimension` | 500 | Longest side of the working photo before mean shift filtering. Speed only |

### Borders

| Flag | Default | Description |
|---|---|---|
| `--borders` | off | Enable region borders |
| `--border-size` | 2 | Border thickness in final pixels |
| `--border-color` | black | Border line color |
| `--max-deviation` | 2 | Maximum pixel distance for border straightening |
| `--min-boundary-length` | 4 | Minimum boundary length before straightening is attempted |

### Color adjustment

| Flag | Default | Description |
|---|---|---|
| `--saturation` | 1.4 | Saturation multiplier |
| `--contrast` | 1.2 | Contrast multiplier |

### Dithering

| Flag | Default | Description |
|---|---|---|
| `--dither` | off | Enable ordered dithering |
| `--dither-strength` | 0.08 | Strength of the effect |

### Light extraction

| Flag | Default | Description |
|---|---|---|
| `--blur-radius` | 10 | Radius used to extract the lighting pattern |
| `--light-strength` | 1.0 | How much lighting is removed from the photo before segmentation |
| `--reapply-strength` | 1.0 | How much lighting is reapplied to the final image |

### Output

| Flag | Default | Description |
|---|---|---|
| `--scale` | 16 | Final output size multiplier per block |
| `--preset` | none | Named preset that sets several flags at once |

---

## Presets

| Preset | Intent |
|---|---|
| `undertale` | Dark background palette, 16 total colors, low max colors per region, borders on, high saturation on accents |
| `gameboy` | Four shade green monochrome, 1 color per region, borders off |
| `nes` | 32 total colors, moderate max per region, no dithering, borders on |
| `arcade-bright` | 24 total colors, high saturation, borders on with thin border size, no dithering |

Any flag passed after a preset overrides that preset's value.

---

## Testing Plan

Unit tests per module:

| Module | Test |
|---|---|
| `resize` | Output dimensions match the requested grid |
| `segmentation` | A flat colored test image yields exactly one label. An image with two clearly different halves yields exactly two |
| `label_resize` | The correct majority label is picked for a block straddling two regions |
| `color_budget` | Flat test regions get the minimum count, high variation regions get more, and the total stays near the requested budget |
| `palette` | Per-region quantization only uses colors present in that region's own pixels |
| `borders` | Borders appear only at recorded region boundaries, never as a full grid |

Beyond unit tests, a small set of sample photos in `examples/input/` is used to visually check output quality after each pipeline change. Style accuracy is judged by manual comparison against real Undertale room screenshots, which stays manual because it is subjective.

---

## Milestones

| # | Milestone | Status |
|---|---|---|
| 1 | Basic pipeline end to end, one shared color count, no segmentation, no borders | Done |
| 2 | Full resolution segmentation with `cv2.pyrMeanShiftFiltering` and `cv2.connectedComponents` | Done |
| 3 | Label map downscaling via majority vote | Done |
| 4 | Small region cleanup | Done |
| 5 | Color budget allocation from per-region variation | Done |
| 6 | Per-region quantization and reassembly | Done |
| 7 | Saturation and contrast boost | Done |
| 8 | Border drawing from the stored label grid | Done |
| 9 | Dithering option | Done |
| 10 | Presets framework in the CLI | Done, definitions pending |
| 11 | README with usage instructions | Done |
| 12 | Border straightening with line fitting | Done |
| 13 | Light extraction | Partial, reapply-only mode works |

---

## Implementation Notes

### Border Straightening

`border_straighten.py`

Uses least-squares line fitting to straighten jagged region boundaries. Lines are fit to detected boundaries in both vertical and horizontal directions, and points within `--max-deviation` pixels are snapped to the fitted line. Canonical ordering keeps boundary detection consistent even when small notches are present. Controlled by `--max-deviation` and `--min-boundary-length`.

### Light Extraction

`light_extract.py`

Extracts luminance patterns from the original image using Gaussian blur, separating lighting from object color while preserving hue and saturation. Controlled by `--blur-radius`, `--light-strength`, and `--reapply-strength`.

The useful configuration is reapply-only. Setting `--light-strength 0.0` with `--reapply-strength 1.0` skips the initial filtering, so no luminance is stripped before segmentation and fine detail is preserved, while the extracted lighting pattern is still layered back onto the output. This works well in rooms with a lot of light, where lamps, windows, and brightness falloff carry much of the scene's character.

Raising `--light-strength` above 0 removes luminance before quantization and tends to flatten detail. That path is still experimental.

### Architecture Decisions

| Decision | Reason |
|---|---|
| Majority vote label downscaling | Label IDs are categories and cannot be averaged |
| Per-region color budgeting | Prevents high variation regions from consuming the whole palette |
| Regional quantization | Avoids global palette conflicts between unrelated objects |
| Boundary post-processing after downscaling | Straightening on the block grid gives better results than on the full resolution map |
| Two-phase lighting | Extraction happens before segmentation, reapplication after upscaling |

---

## Known Issues

| Issue | Detail | Workaround |
|---|---|---|
| Light extraction degradation | Extracting luminance before segmentation reduces detail visibility | Set `--light-strength 0.0` and keep `--reapply-strength 1.0` |
| Missing dependency | `requirements.txt` does not list `opencv-python`, which segmentation requires | Add it before release |
| Presets not implemented | The CLI framework exists but preset definitions are not written in code | Implement in `presets.py` |
| Over-straightened borders | Complex curved boundaries can be straightened too aggressively | Adjust `--max-deviation` and `--min-boundary-length` |

---

## Path to v1.0

1. Add `opencv-python` to `requirements.txt`
2. Implement the preset definitions (undertale, gameboy, nes, arcade-bright) in `presets.py`
3. Run end to end tests on the sample images to verify every feature works
4. Document the reapply-only light extraction configuration as the recommended default, and decide whether full extraction is worth keeping
5. Add before and after example images to the repository

---

## Future Extensions 

Not required for v1:

- Batch mode to convert a whole folder at once
- Simple Tkinter GUI as an alternative to the CLI, still with no external services
- Palette locking to an existing image so multiple outputs share one consistent look
- `cv2.watershed` with markers as an alternative segmentation backend, for cases where two touching objects share nearly identical colors and mean shift filtering merges them
- Manual boundary nudging and region merging, for cases where automatic segmentation splits or merges objects against the user's intent
- Interactive preview of parameter changes before final export