# Workflow 1: From BMP slices to one greyscale `.nrrd` volume

**Project:** micro-CT of BASE1–BASE4 teeth (SkyScan 1172, reconstructed in NRecon)
**Software:** 3D Slicer 5.12.4 (Windows), Python Console
**Output:** one `.nrrd` volume per scan, with the voxel size stored inside the file

---

## 1. Input data

| | Scan 1 | Scan 2 |
|---|---|---|
| Specimens | BASE1 + BASE2 | BASE3 + BASE4 |
| Settings | 80 kV, 124 µA | 85 kV, 118 µA |
| Slice folder | `D:\202609151503_BASE1_BASE2_JIGNITE_ritahardiman_Rec` | `D:\202609161017_BASE3_BASE4_JIGNITE_ritahardiman_Rec` |
| Slices | 2,480 BMPs, `…__rec0096.bmp` to `…__rec2575.bmp`, each 3360 × 3360 px | same |
| Log file | `…__rec.log` (scan and reconstruction settings) | same |
| Voxel size | 5.04569 µm, isotropic (0.00504569 mm) | same |

> **Scan 2 label mismatch.** The sample was mislabelled at scan time, so the
> slices and log in the scan 2 folder are named `BASE3_BASE3…`. The correct
> label is **BASE3_BASE4**, as in the folder name. Don't rename the files. The
> conversion accepts any prefix, and the output is named after the folder.

---

## 2. Why convert, and why greyscale is enough

- **Micro-CT measures one quantity per voxel.** Each voxel stores the X-ray
  attenuation, which rises with density and atomic number. That's a single
  greyscale number. There is no colour information in a CT scan.
- **NRecon still writes "colour" BMPs.** Each 8-bit greyscale slice is stored as
  a 3-channel BMP with **R = G = B**, so the three channels are identical copies.
- **Keeping all three channels:**
  - triples the memory (about 84 GB instead of 28 GB per scan);
  - adds no information;
  - breaks Slicer's loader. Dragging the BMPs in, *Add Data* and
    `slicer.util.loadVolume` all fail with
    `Unsupported number of components: 1 != 3`. On 6 October this cost more
    than an hour of failed loading.
- **Segmentation tools expect one value per voxel.** Threshold, Grow from seeds
  and Segment Statistics all need a single-component (scalar) volume.
- **Keeping one channel is lossless**, provided R = G = B. The conversion
  script checks this on the first, middle and last slice and stops if the
  channels differ.
- **NRRD stores the geometry with the data.** The `.nrrd` file keeps the voxel
  size and orientation alongside the image. Slicer, Python (SimpleITK, pynrrd)
  and Fiji can all open it.

> The 8-bit grey range is fixed during reconstruction by NRecon's contrast
> mapping. These are the *Minimum/Maximum for CS to Image Conversion* lines in
> the `__rec.log` file. Record them for each scan. If they differ between
> scans, grey values are not directly comparable.

---

## 3. Recommended method: the `bmp_to_nrrd.py` script

The script:
1. Finds all `*__recNNNN.bmp` slices and checks none are missing.
2. Reads the voxel size from the `__rec.log` file.
3. Keeps one colour channel.
4. Writes the slices straight into a `.nrrd` file, **one slice at a time**.

It needs only a few hundred MB of RAM, whereas the earlier in-memory method
used about 28 GB. The result is identical: the slices are copied bit for bit.

### 3.1 Save the script (once)

1. Create a folder, for example `D:\Segmentation\scripts\`.
2. Open Notepad, paste the code below, and save it as
   `D:\Segmentation\scripts\bmp_to_nrrd.py`. In the *Save as type* box, choose
   **All files** so Notepad doesn't add `.txt`.

```python
r"""
bmp_to_nrrd.py - convert a SkyScan/NRecon BMP slice stack into ONE greyscale NRRD volume.

NRecon saves each 8-bit greyscale slice as a 3-channel (R = G = B) BMP. This script
keeps one channel (lossless, because the channels are identical), reads the voxel
size from the __rec.log file, and streams the slices straight into an .nrrd file,
so it needs only a few hundred MB of RAM, not 28 GB.

Run inside the 3D Slicer Python Console:
    exec(open(r"D:\Segmentation\scripts\bmp_to_nrrd.py").read())
"""
import glob, gzip, os, re, time
import numpy as np
import SimpleITK as sitk

# ------------------------------------------------------------------ settings ---
folder   = r"D:\202609151503_BASE1_BASE2_JIGNITE_ritahardiman_Rec"     # slice folder
out_path = r"D:\202609151503_BASE1_BASE2_JIGNITE_ritahardiman_Rec.nrrd"  # output file
compress = False      # True = gzip-compressed .nrrd (smaller file, slower to write/read)
voxel_um = None       # None = read "Image Pixel Size (um)" from the __rec.log file
# --------------------------------------------------------------------------------

# 1. Find the slices. Any prefix is accepted, so mislabelled names
#    (e.g. BASE3_BASE3__rec0096.bmp) are fine.
files = sorted(glob.glob(os.path.join(folder, "*__rec[0-9][0-9][0-9][0-9].bmp")))
if not files:
    raise RuntimeError("No *__recNNNN.bmp slices found in " + folder)
numbers = [int(re.search(r"__rec(\d{4})\.bmp$", f).group(1)) for f in files]
missing = sorted(set(range(numbers[0], numbers[-1] + 1)) - set(numbers))
if missing:
    raise RuntimeError(f"{len(missing)} slice numbers missing, e.g. {missing[:5]}")

# 2. Voxel size from the NRecon log (in micrometres), converted to millimetres.
if voxel_um is None:
    for log in glob.glob(os.path.join(folder, "*__rec.log")):
        with open(log, errors="ignore") as fh:
            for line in fh:
                m = re.match(r"\s*Image Pixel Size \(um\)\s*=\s*([0-9.]+)", line)
                if m:
                    voxel_um = float(m.group(1))
                    break
    if voxel_um is None:
        raise RuntimeError("Pixel size not found in the __rec.log file - set voxel_um by hand")
sp = voxel_um / 1000.0

# 3. Read one slice as 2-D greyscale. Colour BMPs: check R = G = B, keep channel 0.
def read_grey(path, check_channels=False):
    a = sitk.GetArrayFromImage(sitk.ReadImage(path))   # (rows, cols) or (rows, cols, 3)
    if a.ndim == 3:
        if check_channels and not (np.array_equal(a[..., 0], a[..., 1]) and
                                   np.array_equal(a[..., 0], a[..., 2])):
            raise RuntimeError(f"R, G and B differ in {os.path.basename(path)}: "
                               "this is not a plain greyscale slice - stop and check")
        a = a[..., 0]
    if a.dtype != np.uint8:
        raise RuntimeError(f"Expected 8-bit slices, got {a.dtype}")
    return a

ny, nx = read_grey(files[0]).shape
nz = len(files)
for f in (files[0], files[nz // 2], files[-1]):      # spot-check the colour channels
    read_grey(f, check_channels=True)
print(f"{nz} slices of {nx} x {ny}, voxel {voxel_um} um -> {out_path}")
print(f"Output size about {nx * ny * nz / 1e9:.1f} GB (uncompressed)")

# 4. NRRD header. Orientation matches the scan-1/scan-2 conversions already done:
#    slice number increases towards S (superior), in-plane axes as in Slicer's
#    IJK-to-RAS directions (-1, -1, 1).
header = (
    "NRRD0004\n"
    "# Converted from SkyScan/NRecon BMP slices by bmp_to_nrrd.py\n"
    "type: uint8\n"
    "dimension: 3\n"
    "space: left-posterior-superior\n"
    f"sizes: {nx} {ny} {nz}\n"
    f"space directions: ({sp:.10g},0,0) (0,{sp:.10g},0) (0,0,{sp:.10g})\n"
    "kinds: domain domain domain\n"
    f"encoding: {'gzip' if compress else 'raw'}\n"
    "space origin: (0,0,0)\n"
    "\n"
)

# 5. Stream the slices into the file, one at a time.
t0 = time.time()
with open(out_path, "wb") as fh:
    fh.write(header.encode("ascii"))
    stream = gzip.GzipFile(fileobj=fh, mode="wb", compresslevel=1) if compress else fh
    for i, f in enumerate(files):
        g = read_grey(f)
        if g.shape != (ny, nx):
            raise RuntimeError(f"{os.path.basename(f)} has size {g.shape[::-1]}, expected {(nx, ny)}")
        stream.write(np.ascontiguousarray(g).tobytes())
        if i % 100 == 0 or i == nz - 1:
            el = time.time() - t0
            eta = el / (i + 1) * (nz - i - 1)
            print(f"  {i + 1}/{nz} slices  ({el / 60:.1f} min elapsed, ~{eta / 60:.0f} min left)")
            try:
                slicer.app.processEvents()   # keep Slicer responsive (ignored outside Slicer)
            except NameError:
                pass
    if compress:
        stream.close()
print(f"Done in {(time.time() - t0) / 60:.1f} min: {out_path}")
```

### 3.2 Run it for one scan

1. Open the script in Notepad and edit the two lines under `settings`:
   - `folder` = the slice folder of the scan;
   - `out_path` = where the `.nrrd` file should go, with about 30 GB free.

   Leave `compress = False` unless disk space is short. Gzip saves space but is
   slower to write and to load.
2. Start 3D Slicer and open *View → Python Console*.
3. Type this and press Enter:
   ```python
   exec(open(r"D:\Segmentation\scripts\bmp_to_nrrd.py").read())
   ```
4. Watch the progress lines:
   ```
   2480 slices of 3360 x 3360, voxel 5.04569 um -> D:\…\…_Rec.nrrd
     101/2480 slices  (1.6 min elapsed, ~37 min left)
   ```
   How long it takes depends mainly on how fast the BMPs can be read from D:.
   Earlier conversions on the lab PC took about 1.5 hours (scan 1) and about 7 hours (scan 2). If it's slow, copy the
   slice folder to an internal SSD first, then point `folder` at the copy.

### 3.3 Check the result

Paste this into the same Python Console. It relies on the `files` and
`out_path` variables left behind by the script:

```python
import numpy as np, SimpleITK as sitk
v = slicer.util.loadVolume(out_path)
a = slicer.util.arrayFromVolume(v)
print("spacing (mm):", v.GetSpacing())
print("size (slices, rows, cols):", a.shape, "  grey range:", a.min(), "-", a.max())
k = a.shape[0] // 2
b = sitk.GetArrayFromImage(sitk.ReadImage(files[k]))[..., 0]
print("middle slice identical to its BMP:", np.array_equal(a[k], b))
```

The conversion is good if all of these hold:
- the spacing is `(0.00504569, 0.00504569, 0.00504569)`;
- the size is `(2480, 3360, 3360)`;
- the middle slice comparison prints **True**.

**Note the grey range.** If the maximum is very low (for example about 33
rather than close to 255), the 8-bit range is poorly used. Thresholds will then
have only a few grey levels to work with, so check NRecon's contrast settings.

### 3.4 Repeat for scan 2

Change `folder` and `out_path` to the scan 2 paths, then run steps 3.2 and 3.3
again.

---

## 4. Orientation in the output file

- **Slice axis.** Slice number increases towards **S** (superior). So **S ↔ I**
  is the scanner's vertical, rotation axis: S is "up in the scanner".
- **In-plane axes.** **R ↔ L** and **A ↔ P** are the two directions within each
  slice.
- **Consistency with earlier conversions.** This is the same convention as the
  conversions already done, which used Slicer IJK-to-RAS directions (−1, −1, 1).
- **No anatomical meaning.** For extracted teeth these letters carry no
  anatomical meaning. Record anatomical directions (occlusal/apical,
  mesial/distal, buccal/lingual) from the teeth themselves.
- **Measurements aren't affected.** Volumes and thicknesses don't depend on
  orientation.

---

## 5. Files produced so far

| Scan | `.nrrd` file | How it was made |
|---|---|---|
| 1 | `D:\BASE1_BASE2_JIGNITE.nrrd`, or `D:\202609151503_BASE1_BASE2_JIGNITE_ritahardiman_Rec.nrrd` if renamed. Keep **one** copy. | Earlier in-memory Python method (7–8 Oct 2026), uncompressed |
| 2 | `D:\202609161017_BASE3_BASE4_JIGNITE_ritahardiman_Rec.nrrd` | Same method (8 Oct 2026), uncompressed |

Both files have about 28 GB of data with spacing 0.00504569 mm. They don't
need to be redone. Use `bmp_to_nrrd.py` for future scans, or to re-convert if
the check in 3.3 fails.

---

## 6. Troubleshooting

| Message or symptom | Cause | Fix |
|---|---|---|
| `Unsupported number of components: 1 != 3` | BMPs opened directly in Slicer | Don't load BMPs. Use the script. |
| `R, G and B differ in …` | A slice really contains colour | Stop and check the reconstruction settings. Don't convert. |
| `N slice numbers missing` | Gaps in the `__recNNNN` numbering | Re-export the missing slices from NRecon. |
| `Pixel size not found` | Log missing or renamed | Set `voxel_um = 5.04569` in the settings. |
| Very slow (several minutes per 100 slices) | Slow reads from D: or antivirus scanning each file | Copy the folder to an internal SSD first. |
| `SyntaxError` when running `exec(...)` | Path typed without `r"…"` or with mismatched quotes | Use `exec(open(r"D:\…\bmp_to_nrrd.py").read())` exactly. |

---

## 7. Alternative without a script: ImageStacks (SlicerMorph)

The **ImageStacks** module in the SlicerMorph extension can also load the BMP
stack:
1. Select the first slice.
2. Set the spacing to 0.00504569 mm.
3. Choose **greyscale** (single component).
4. Choose full resolution, and load.
5. Save the volume with *File → Save* as `.nrrd`.

This loads the whole 28 GB volume into RAM, which is fine on the 120 GB lab PC,
and it doesn't check that R = G = B. Option names may differ slightly between
SlicerMorph versions.
