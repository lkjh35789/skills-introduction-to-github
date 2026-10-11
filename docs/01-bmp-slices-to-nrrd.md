# Workflow 1: From BMP slices to one greyscale `.nrrd` volume

**Applies to:** any SkyScan/NRecon micro-CT reconstruction saved as BMP slices
**Software:** 3D Slicer (tested workflow on 5.12.4), *View → Python Console*
**Output:** one greyscale `.nrrd` per scan, with the voxel size stored inside,
plus a small text record of the scan's parameters

This is the first step of the segmentation pipeline:

**BMP slices → `.nrrd` (this file) → crop each tooth (Workflow 2) → segment
tooth structure, cracks, voids and restoration findings (Workflow 3)**

The aims are:
1. to make the data **lighter to process**: one channel instead of three, and
   one file instead of thousands;
2. to **check and record the parameters** of every scan, so mistakes and
   scan-to-scan differences are caught early, before they affect segmentation.

---

## Why greyscale is enough

- **One number per voxel.** Micro-CT measures one quantity per voxel: X-ray
  attenuation, which rises with density and atomic number. That's a greyscale
  value, with no colour information.
- **NRecon still saves colour BMPs.** It writes each 8-bit greyscale slice as a
  3-channel BMP with **R = G = B**. The channels are copies of each other.
- **Keeping all three channels:**
  - triples the memory (for example about 84 GB instead of about 28 GB for a
    3360 × 3360 × 2480 scan);
  - adds no information;
  - makes Slicer's loader fail with `Unsupported number of components: 1 != 3`.
- **Segmentation tools need one value per voxel.** Threshold, Grow from seeds,
  Islands and Segment Statistics all expect single-component volumes.
- **Keeping one channel is lossless** when R = G = B. The steps below
  **check this on every slice** and stop if it is ever not true.

---

## Parameters to check for every scan

Fill in one copy of this table per scan. Steps 2–6 print each value, and
step 9 saves them all to a text file.

| # | Parameter | Where it comes from | Expected (current scans) | If it's different |
|---|---|---|---|---|
| 1 | Sample label | Folder name | e.g. `…_BASE1_BASE2_…` | The **folder name** is used to name the output. Make sure it's correct. |
| 2 | Slice file prefix | BMP names (step 2) | Same label as the folder | Mislabelled at scan time (e.g. `BASE3_BASE3` in the BASE3_BASE4 folder). **Record it and don't rename files.** |
| 3 | Number of slices, first and last number | Step 2; log `First/Last Section` | 2,480 (`rec0096`–`rec2575`) | It varies with the scanned height. Check it matches the log. |
| 4 | Missing slice numbers | Step 2 | None | Re-export the missing slices from NRecon before converting. |
| 5 | Slice size (width × height, pixels) | Step 5; log `Result Image Width/Height` | 3360 × 3360 | It varies with the field of view or binning. All slices must be the same size. |
| 6 | **Voxel size** | Log `[Reconstruction] Pixel Size (um)` and `[Acquisition] Image Pixel Size (um)` (step 3) | 5.04569 µm | **Critical.** Every volume and length depends on it. If the two log values differ, NRecon resized the slices, so use the reconstruction value. |
| 7 | Bit depth | Step 5 | 8-bit (`uint8`) | 16-bit TIFF exports need a different workflow. Ask before converting. |
| 8 | Colour channels, R = G = B | Step 5; checked again on every slice in step 7 | 3 channels, identical | If the channels differ, stop. It isn't a plain greyscale slice. |
| 9 | Grey range actually used | Steps 5 and 8 | Close to 0–255 | A narrow range (e.g. max ≈ 33) gives few grey levels for thresholds. Check NRecon's contrast limits. |
| 10 | NRecon contrast limits | Log `Minimum/Maximum for CS to Image Conversion` | Record | If they differ between scans, **grey values aren't comparable** between scans. |
| 11 | Source voltage / current / filter | Log | e.g. 80 kV, 124 µA (scan 1); 85 kV, 118 µA (scan 2) | Changes contrast and noise. Thresholds must be set per scan. |
| 12 | Exposure, rotation step, frame averaging, random movement | Log | Record | Affects noise, which affects how small a crack or void can be trusted. |
| 13 | Smoothing, ring-artefact correction, beam-hardening correction, post-alignment | Log | Record | **Ring and streak artefacts can mimic cracks.** Keep these the same across scans where possible. |
| 14 | Output file | Step 6 | `<folder name>.nrrd`, enough free space, not overwriting | Change `output_folder`, or free some space. |
| 15 | Output checks | Step 8 | Spacing = voxel size; dimensions as above; slices identical to the BMPs | Delete the output and repeat. |

---

## How to use the code blocks

1. Open 3D Slicer, then open *View → Python Console*.
2. Copy **one block at a time** into the console and press **Enter**. If the
   prompt shows `...`, press **Enter** once more to run the block.
3. Read the output and compare it with the **✅ Check** list before moving on.
4. **For a new scan:** change the two paths in step 1, then run all the steps again.

---

## Step 1: Set the scan folder and the output folder ✏️

This is **the only block you edit**. Change the two paths for each scan.

```python
scan_folder   = r"D:\202609151503_BASE1_BASE2_JIGNITE_ritahardiman_Rec"   # <-- folder with the BMP slices
output_folder = r"D:\Segmentation\NRRD"                                   # <-- where the .nrrd will be saved
import os
os.makedirs(output_folder, exist_ok=True)
print("Scan folder found:  ", os.path.isdir(scan_folder))
print("Output folder ready:", os.path.isdir(output_folder))
```

✅ **Check**
- Both lines say `True`.
- If the scan folder says `False`, the path is wrong. In File Explorer, open
  the folder, click the address bar, copy the path, and paste it between the quotes.
- Keep the `r` before the opening quote. It lets Windows paths with `\` work.

---

## Step 2: Find the slices and check the numbering

```python
import glob, re
files = sorted(glob.glob(os.path.join(scan_folder, "*__rec[0-9][0-9][0-9][0-9].bmp")))
print("Number of slices:", len(files))
nums = [int(re.search(r"__rec(\d{4})\.bmp$", f).group(1)) for f in files]
missing = sorted(set(range(nums[0], nums[-1] + 1)) - set(nums)) if nums else []
print("First slice:", os.path.basename(files[0]) if files else "NONE FOUND")
print("Last slice: ", os.path.basename(files[-1]) if files else "NONE FOUND")
print("Missing numbers:", len(missing), missing[:10])
print("File prefix:", os.path.basename(files[0]).split("__rec")[0] if files else "-")
print("Folder name:", os.path.basename(os.path.normpath(scan_folder)))
```

✅ **Check**
- **Number of slices** is not 0. For the current scans it is 2480.
- **Missing numbers** is `0 []`.
- Compare **File prefix** with **Folder name**. If the labels differ (as with
  `BASE3_BASE3` in the BASE3_BASE4 folder), write it down. The output is named
  after the folder.

---

## Step 3: Read the scan and reconstruction settings from the log

```python
logs = glob.glob(os.path.join(scan_folder, "*__rec.log"))
print("Log file:", [os.path.basename(p) for p in logs])
wanted = ("Source Voltage", "Source Current", "Filter", "Exposure", "Rotation Step",
          "Frame Averaging", "Random Movement", "Image Pixel Size", "Pixel Size",
          "Result Image Width", "Result Image Height", "First Section", "Last Section",
          "Minimum for CS to Image Conversion", "Maximum for CS to Image Conversion",
          "Smoothing", "Ring Artifact Correction", "Beam Hardening Correction",
          "Post Alignment", "Result File Type")
log_values = {}
section = ""
with open(logs[0], errors="ignore") as fh:
    for line in fh:
        line = line.strip()
        if line.startswith("[") and line.endswith("]"):
            section = line
        elif "=" in line and line.split("=")[0].strip().startswith(wanted):
            key, val = [s.strip() for s in line.split("=", 1)]
            log_values[section + " " + key] = val
            print(f"{section:18s} {key:42s} = {val}")
```

✅ **Check** and copy these into your table (rows 3, 5, 6 and 10–13):
- There is exactly **one** log file. If the list is empty, the log is missing.
  Find it before continuing, because the voxel size comes from it.
- **First/Last Section** match the first and last slice numbers from step 2.
- **Result Image Width/Height** are the slice size, for example 3360.
- Note **Source Voltage/Current/Filter**, the **CS to Image Conversion**
  limits and the corrections. Compare them with previous scans.

---

## Step 4: Confirm the voxel size

```python
recon = [v for k, v in log_values.items() if k.startswith("[Reconstruction]") and "Pixel Size" in k]
acq = [v for k, v in log_values.items() if "Image Pixel Size" in k]
print("Reconstruction pixel size (um):", recon)
print("Acquisition pixel size (um):   ", acq)
voxel_um = float((recon or acq)[0])
spacing_mm = round(voxel_um / 1000, 10)
print("USING voxel size:", voxel_um, "um =", spacing_mm, "mm")
```

✅ **Check**
- The value is plausible: for the current scans, **5.04569 µm = 0.00504569 mm**.
- If the reconstruction and acquisition values **differ**, NRecon resized the
  slices. The block uses the **reconstruction** value, which is correct for the slices.
- If both lists are empty, or the number is wrong, set it by hand and run this:
  ```python
  voxel_um = 5.04569          # <-- type the correct value in micrometres
  spacing_mm = round(voxel_um / 1000, 10)
  print("USING voxel size:", voxel_um, "um =", spacing_mm, "mm")
  ```

---

## Step 5: Check the slices themselves (size, bit depth, colour channels)

```python
import numpy as np, SimpleITK as sitk
def check_slice(path):
    img = sitk.ReadImage(path)
    a = sitk.GetArrayFromImage(img)
    same = a.ndim == 2 or (np.array_equal(a[..., 0], a[..., 1]) and np.array_equal(a[..., 0], a[..., 2]))
    print(os.path.basename(path), "| size", img.GetSize(), "| channels", img.GetNumberOfComponentsPerPixel(),
          "| type", a.dtype, "| R=G=B:", same, "| grey range", a.min(), "-", a.max())

for f in (files[0], files[len(files) // 2], files[-1]):
    check_slice(f)
```

✅ **Check** (rows 5, 7, 8 and 9)
- **size** is the same on all three lines, and matches the log width and height.
- **type** is `uint8` (8-bit).
- **channels** is 3 (or 1), and **R=G=B** is `True` on every line.
- **grey range**: the middle slice should use a good part of 0–255. A very low
  maximum means few grey levels, so note it and check NRecon's contrast limits
  (row 10).

---

## Step 6: Set the output name and check disk space

```python
import shutil
folder_name = os.path.basename(os.path.normpath(scan_folder))
out_path = os.path.join(output_folder, folder_name + ".nrrd")
nx, ny = sitk.ReadImage(files[0]).GetSize()
nz = len(files)
size_gb = nx * ny * nz / 1e9
free_gb = shutil.disk_usage(output_folder).free / 1e9
print("Output file:", out_path)
print(f"Volume: {nx} x {ny} x {nz} voxels = {size_gb:.1f} GB   |   free space: {free_gb:.0f} GB")
print("WARNING: file already exists and will be overwritten" if os.path.exists(out_path) else "No existing file - OK")
```

✅ **Check** (row 14)
- The **file name** shows the correct sample label (taken from the folder).
- **Free space** is larger than the volume size, with a margin.
- If a file **already exists**, decide whether to overwrite it. If not, change
  `output_folder` in step 1 and rerun steps 1 and 6.

---

## Step 7: Convert (streams slice by slice, low memory)

This writes the `.nrrd` header, then adds the slices one at a time:
1. It checks **R = G = B and the slice size on every slice**.
2. It keeps one channel.
3. It writes that channel to the file.

It needs only a few hundred MB of RAM, whatever the scan size. **Speed depends
on how fast the BMPs can be read.** On the lab PC this took about 1.5 hours
for scan 1 and about 7 hours for scan 2. If it's slow, copy the slice folder to
an internal SSD first, set `scan_folder` to the copy, and start again from step 1.

```python
import gzip, time
compress = False      # True = gzip-compressed .nrrd (smaller file, slower)
header = ("NRRD0004\ntype: uint8\ndimension: 3\nspace: left-posterior-superior\n"
          f"sizes: {nx} {ny} {nz}\n"
          f"space directions: ({spacing_mm:.10g},0,0) (0,{spacing_mm:.10g},0) (0,0,{spacing_mm:.10g})\n"
          "kinds: domain domain domain\n"
          f"encoding: {'gzip' if compress else 'raw'}\nspace origin: (0,0,0)\n\n")
t0 = time.time()
with open(out_path, "wb") as fh:
    fh.write(header.encode("ascii"))
    out = gzip.GzipFile(fileobj=fh, mode="wb", compresslevel=1) if compress else fh
    for i, f in enumerate(files):
        a = sitk.GetArrayFromImage(sitk.ReadImage(f))
        if a.ndim == 3:
            if not (np.array_equal(a[..., 0], a[..., 1]) and np.array_equal(a[..., 0], a[..., 2])):
                raise RuntimeError("R, G and B differ in " + os.path.basename(f) + " - stop and check")
            a = a[..., 0]
        if a.shape != (ny, nx) or a.dtype != np.uint8:
            raise RuntimeError(f"Unexpected size/type in {os.path.basename(f)}: {a.shape[::-1]} {a.dtype}")
        out.write(np.ascontiguousarray(a).tobytes())
        if i % 100 == 0 or i == nz - 1:
            el = time.time() - t0
            print(f"{i + 1}/{nz} slices  |  {el / 60:.1f} min  |  ~{el / (i + 1) * (nz - i - 1) / 60:.0f} min left")
            slicer.app.processEvents()
    if compress:
        out.close()
print("Finished:", out_path, f"in {(time.time() - t0) / 60:.1f} min")
```

✅ **Check**
- The last line reads `Finished: …`.
- If it stops with **"R, G and B differ"** or **"Unexpected size/type"**, a
  slice is not like the others. Note which one, then **delete the incomplete
  `.nrrd`** and check that slice in NRecon.

> **Orientation.** Slice number increases towards **S** (superior), so S ↔ I
> is the scanner's vertical (rotation) axis. This matches the scan 1 and scan 2
> conversions already done (Slicer IJK-to-RAS directions −1, −1, 1). For
> extracted teeth, S/I/A/P/R/L are scanner axes, not anatomy.

---

## Step 8: Verify the new `.nrrd`

```python
v = slicer.util.loadVolume(out_path)
arr = slicer.util.arrayFromVolume(v)
print("Spacing (mm):", v.GetSpacing())
print("Dimensions (x, y, z):", v.GetImageData().GetDimensions())
print("Grey range:", arr.min(), "-", arr.max())
for k in (0, nz // 2, nz - 1):
    ref = sitk.GetArrayFromImage(sitk.ReadImage(files[k]))
    ref = ref[..., 0] if ref.ndim == 3 else ref
    print(f"Slice {k}: identical to its BMP = {np.array_equal(arr[k], ref)}")
```

✅ **Check** (row 15)
- **Spacing** equals the voxel size from step 4, for example
  `(0.00504569, 0.00504569, 0.00504569)`.
- **Dimensions** equal `nx, ny, nz`, for example `(3360, 3360, 2480)`.
- All three slices say `True`.
- The volume appears in the slice views, and the teeth look normal.

---

## Step 9: Save a record of this scan's parameters

```python
import datetime
record_path = out_path.replace(".nrrd", "_conversion_record.txt")
with open(record_path, "w") as rec:
    rec.write(f"Converted: {datetime.datetime.now():%Y-%m-%d %H:%M}\n")
    rec.write(f"Scan folder: {scan_folder}\nOutput: {out_path}\n")
    rec.write(f"Slice prefix: {os.path.basename(files[0]).split('__rec')[0]}\n")
    rec.write(f"Slices: {nz} ({os.path.basename(files[0])} to {os.path.basename(files[-1])}), missing: {len(missing)}\n")
    rec.write(f"Slice size: {nx} x {ny} px, voxel size: {voxel_um} um ({spacing_mm} mm)\n")
    rec.write(f"Grey range in volume: {arr.min()} - {arr.max()}\n")
    rec.write("\nFrom the NRecon log:\n")
    for k, val in log_values.items():
        rec.write(f"  {k} = {val}\n")
print("Record saved:", record_path)
```

✅ **Check**
- Open the `_conversion_record.txt` file next to the `.nrrd` and keep it with
  your lab notes.
- **Compare it with the records of earlier scans.** Anything different in
  rows 5, 6 or 9–13 needs attention before you compare segmentation results
  across scans.

---

## Step 10: Free memory, then do the next scan

```python
del arr
slicer.mrmlScene.RemoveNode(v)
print("Volume removed from Slicer (the file on disk is kept).")
```

**For the next scan:** go back to **Step 1**, change `scan_folder` (and
`output_folder` if you like), and run steps 1–10 again.

---

## Troubleshooting

| Message or symptom | Cause | Fix |
|---|---|---|
| `Scan folder found: False` | Wrong path | Copy the path from the File Explorer address bar. Keep the `r"…"`. |
| `Number of slices: 0` | Wrong folder, or slices aren't named `…__recNNNN.bmp` | Check the folder, and look at one file name. |
| `IndexError: list index out of range` in step 3 | No `__rec.log` in the folder | Find the log (copy it from the reconstruction PC), or set the voxel size by hand in step 4. |
| `R, G and B differ …` | A slice really contains colour | Stop. Check the NRecon export settings. |
| `Unexpected size/type …` | A slice has a different size, or isn't 8-bit | Re-export it from NRecon. Delete the incomplete `.nrrd`. |
| `Unsupported number of components: 1 != 3` | BMPs were opened directly in Slicer | Don't load BMPs. Use these steps. |
| Prompt shows `...` and nothing happens | The console waits for the end of a block | Press **Enter** once more. |
| Very slow (several minutes per 100 slices) | Slow reads from the drive, or antivirus | Copy the slice folder to an internal SSD first. |

---

## Files converted so far

| Scan | Folder | `.nrrd` | Notes |
|---|---|---|---|
| 1 | `D:\202609151503_BASE1_BASE2_JIGNITE_ritahardiman_Rec` | `D:\BASE1_BASE2_JIGNITE.nrrd` (or renamed to the folder name; keep **one** copy) | Earlier in-memory conversion, 7–8 Oct 2026. 80 kV / 124 µA. |
| 2 | `D:\202609161017_BASE3_BASE4_JIGNITE_ritahardiman_Rec` | `D:\202609161017_BASE3_BASE4_JIGNITE_ritahardiman_Rec.nrrd` | 8 Oct 2026. 85 kV / 118 µA. Slice/log prefix `BASE3_BASE3` (mislabelled; the folder name is correct). |

These two files don't need redoing. To create a parameter record for an
existing `.nrrd` without converting again:
1. Run steps 1–5 for its scan folder.
2. Run this block instead of steps 6 and 7.
3. Run steps 8, 9 and 10.

```python
out_path = r"D:\BASE1_BASE2_JIGNITE.nrrd"     # <-- the existing .nrrd for this scan folder
nx, ny = sitk.ReadImage(files[0]).GetSize()
nz = len(files)
print("Using existing file:", out_path, "| exists:", os.path.exists(out_path))
```
