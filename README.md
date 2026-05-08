# Rounded Card Extractor

Contour-based extraction pipeline for scanned playing cards and trading cards with rounded corners.

Designed for flatbed-scanned TIFF batches using a red separation background.

The extractor performs:

* background segmentation
* contour isolation
* perspective rectification
* rounded-edge alpha masking
* transparent TIFF export

while preserving internal red artwork printed on the cards.

---

# Processing Pipeline

## 1. TIFF Frame Decoding

The tool supports:

* `.tif`
* `.tiff`
* multi-page TIFF batches

Frames are decoded using Pillow and converted into RGB NumPy arrays for OpenCV processing.

---

# 2. Red Background Segmentation

The background is isolated in HSV space.

Two hue ranges are required because red wraps around the HSV hue wheel.

## HSV Thresholds

```python id="3bprg7"
lower_red1 = [0,   60, 40]
upper_red1 = [12, 255,255]

lower_red2 = [168, 60, 40]
upper_red2 = [180,255,255]
```

The resulting masks are merged:

```python id="m3shmj"
red_mask = mask1 OR mask2
```

## Why HSV Instead of RGB

HSV provides significantly better robustness against:

* scanner brightness variation
* shadows
* uneven illumination
* paper texture variation

Hue remains relatively stable even when saturation/value shift.

---

# 3. Morphological Cleanup

After thresholding, the background mask is stabilized using morphological operations.

## Kernel

```python id="bphmta"
cv2.getStructuringElement(
    cv2.MORPH_ELLIPSE,
    (9, 9)
)
```

## Operations

### Closing

Fills holes and reconnects fragmented background regions.

```python id="5myn9d"
cv2.morphologyEx(
    mask,
    cv2.MORPH_CLOSE,
    kernel,
    iterations=2
)
```

### Opening

Removes small isolated noise regions.

```python id="xixy0x"
cv2.morphologyEx(
    mask,
    cv2.MORPH_OPEN,
    kernel,
    iterations=1
)
```

Elliptical kernels are used instead of rectangular kernels to avoid introducing angular artifacts near rounded corners.

---

# 4. Foreground Isolation

Foreground objects are computed by inverting the red background mask:

```python id="rm4s5u"
foreground = bitwise_not(red_mask)
```

Important:

The extractor does NOT remove red pixels from the card itself.

Red is used only to locate object boundaries.

This prevents corruption of:

* hearts
* diamonds
* red logos
* UI elements
* artwork
* decorative borders

inside the card face.

---

# 5. Contour Extraction

Contours are extracted using:

```python id="4trrja"
cv2.findContours(
    foreground,
    cv2.RETR_EXTERNAL,
    cv2.CHAIN_APPROX_SIMPLE
)
```

## Retrieval Mode

`RETR_EXTERNAL`

Only outer contours are retained.

This avoids:

* internal artwork contours
* printed shapes
* suit symbols
* text regions

from affecting extraction.

---

# 6. Contour Filtering

Detected contours are filtered by area:

```python id="kj8i3l"
min_area = 20000
```

This removes:

* dust
* scan artifacts
* small disconnected regions
* paper fibers

The threshold should scale with scan DPI.

Typical values:

| DPI     | Recommended Min Area |
| ------- | -------------------- |
| 300 DPI | 15k–30k              |
| 600 DPI | 60k–120k             |

---

# 7. Card Orientation + Rectification

Each contour is fitted using:

```python id="g7np2e"
cv2.minAreaRect(contour)
```

This computes:

* center point
* rotation angle
* minimal enclosing rectangle

The rectangle is expanded slightly to preserve rounded corners:

```python id="m93kxa"
padding = 12px
```

Expansion occurs radially from the rectangle center.

---

# 8. Perspective Transform

Perspective correction is computed using:

```python id="wmsy3q"
cv2.getPerspectiveTransform(src, dst)
```

and applied with:

```python id="0v69n9"
cv2.warpPerspective(...)
```

Interpolation:

```python id="c1w2j0"
cv2.INTER_CUBIC
```

Cubic interpolation improves edge quality and minimizes stair-stepping artifacts.

---

# 9. Rounded Corner Preservation

The alpha mask is NOT generated from chroma keying.

Instead:

1. The outer contour is rasterized
2. A filled contour mask is generated
3. The mask is warped together with the image
4. The warped mask becomes the alpha channel

## Contour Mask

```python id="j6cl8k"
cv2.drawContours(
    mask,
    [contour],
    -1,
    255,
    thickness=-1
)
```

This preserves:

* rounded corners
* edge curvature
* die-cut geometry
* irregular border shapes

with pixel accuracy.

---

# 10. Alpha Channel Generation

Final RGBA output:

```python id="x7r9lu"
rgba = np.dstack([
    warped_rgb,
    warped_mask
])
```

Transparent regions are fully removed while preserving anti-aliased edges.

---

# 11. TIFF Export

Cards are exported individually as:

```text id="qv91my"
card_001.tiff
```

using:

```python id="3yd2v3"
compression='tiff_lzw'
```

Features:

* lossless compression
* alpha transparency
* high archival quality
* editor compatibility

---

# Recommended Scan Conditions

Optimal conditions:

| Parameter    | Recommendation |
| ------------ | -------------- |
| Background   | Matte red      |
| DPI          | 300–600        |
| Lighting     | Even           |
| Shadows      | Minimal        |
| Card spacing | ≥1 cm          |
| Scan format  | TIFF           |
| Compression  | Lossless       |

---

# Failure Cases

Potential issues include:

* cards touching each other
* reflective holographic foil
* weak background contrast
* red/orange scan backgrounds too close to artwork colors
* severe perspective distortion
* motion blur during scanning

---

# Possible Future Improvements

## Detection

* adaptive HSV calibration
* LAB color space support
* edge-based segmentation
* Hough edge refinement
* contour smoothing

## Geometry

* subpixel corner refinement
* border snapping
* curvature fitting
* lens distortion correction

## Performance

* multiprocessing
* CUDA/OpenCL acceleration
* tiled processing for ultra-high-resolution scans

## ML Extensions

* segmentation models
* card-type classification
* automatic orientation detection
* OCR integration

---

# Tech Stack

* Python
* OpenCV
* NumPy
* Pillow

---
