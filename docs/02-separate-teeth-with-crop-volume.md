# Workflow 2: Separate the teeth in each scan with Crop Volume

**Input:** one `.nrrd` per scan from Workflow 1, plus its `_conversion_record.txt`. Each scan contains **two teeth**.
**Output:** for each tooth, a **full-resolution crop** and a **2× downsampled working copy**, plus the crop box (ROI), all saved as files.
**Software:** 3D Slicer (tested workflow on 5.12.4), **Crop Volume** module (built in)

**BMP slices → `.nrrd` (Workflow 1) → crop each tooth (this file) → segment
tooth structure, cracks, voids and restoration findings (Workflow 3)**

---

## 1. Why crop, and why two versions of each tooth

Cropping does three jobs:
1. **Separates the teeth.** Each crop box holds one tooth.
2. **Cuts the data down.** One tooth is a few GB instead of about 28 GB for the
   whole scan, so segmentation tools run comfortably.
3. **Sets up a smooth segmentation workflow.** Each tooth gets a consistent set
   of files, named the same way for every scan.

What you're looking for decides the resolution:

| Version | How | Voxel size (current scans) | Smallest feature you can trust\* | Use for |
|---|---|---|---|---|
| **Full resolution** (`<label>_5um`) | Interpolated cropping **off**: voxels are copied exactly, with no resampling | 5.05 µm | about 10–15 µm | **Cracks, voids, restoration/base findings, interface gaps.** This is the master copy. |
| **Working copy** (`<label>_10um`) | Interpolated cropping **on**, spacing scale **2** | 10.09 µm | about 20–30 µm | Faster segmentation of the bulk structure: enamel, dentine, pulp, and the restoration outline |

\* A feature needs to be about 2–3 voxels across to be measured reliably.
Downsampling blurs and can erase cracks that are only 1–2 full-resolution
voxels wide. **Always keep the full-resolution crop** for crack and void analysis.

**Example from scan 1, first tooth (box about 9.1 × 9.6 × 5.4 mm):**
- full resolution: 1802 × 1900 × 1078 voxels, about 3.7 GB;
- working copy: 901 × 950 × 539 voxels, about 0.46 GB, spacing 0.01009138 mm.

> If a future scan has a **different voxel size**, name the files by their
> actual voxel size. For example, at 8 µm you'd use `_8um` and `_16um`. The
> check in section 9 prints the real spacing.

---

## 2. Parameters to check for every scan and tooth

| # | Check | Where | Expected (current scans) | If it's different |
|---|---|---|---|---|
| 1 | Input spacing | Crop Volume → *Volume information → Input volume → Spacing* | Same as the voxel size in the scan's `_conversion_record.txt` (0.00504569 mm) | Wrong file loaded, or the conversion used the wrong voxel size. **Stop and check.** |
| 2 | Input dimensions | Same panel | Same as the conversion record (3360 × 3360 × 2480) | Wrong file, or an incomplete conversion |
| 3 | Which tooth is which | Scan notes and the slice views | Recorded: upper/lower → label | Never guess. The image and file names can't tell you. |
| 4 | Box contains the **whole** tooth and **none** of the other | All three slice views | About 0.5 mm margin around the tooth | Move the box handles and check again |
| 5 | Full-res crop: interpolated cropping **off** | *Advanced* | Output spacing **equals** input spacing | If it's on, the data are resampled. Untick it and crop again. |
| 6 | Working copy: interpolated **on**, scale **2** | *Advanced* | Output spacing = 2 × input (0.01009138 mm) | Correct the settings and crop again |
| 7 | Output dimensions and size | *Volume information → Cropped volume*; the section 9 printout | Plausible for one tooth (a few GB at full resolution) | If very large, the box is too big. If very small, the box is cutting the tooth. |
| 8 | New output name each time | *Output volume → Create new Volume as…* | `<label>_5um`, `<label>_10um` | If left as an existing volume, **Apply overwrites it.** |
| 9 | Files saved | *File → Save* dialog | `.nrrd` volumes and `.mrk.json` boxes, in the right folder | Save again before deleting anything |

---

## 3. Load the scan

1. Start Slicer. From the toolbar layout button, choose **Four-Up**.
2. Go to *File → Add Data*, select the scan's `.nrrd`, and click **OK**.
   Loading 28 GB takes from about 30 seconds to a few minutes.
3. Adjust the contrast by **left-click and drag** in a slice view.
4. **Don't** turn on Volume Rendering for the full scan. The graphics card
   can't hold it, and the 3D view will show only a tiny blob. If it's on, open
   the **Volume Rendering** module and click the eye icon to turn it off.
5. Open the scan's `_conversion_record.txt` beside Slicer, for checks 1 and 2.

**Scan 1 layout (example):** the two teeth are **stacked crown to crown**. The
upper crown points down and the lower crown points up, with an **air gap**
between them. The faint circle and vertical lines are the holder or jig.
Future scans may be arranged differently. The method is the same: one box per
tooth.

---

## 4. Full-resolution crop of the first tooth (`<label>_5um`)

1. Open the module menu and choose **Crop Volume** (or use the magnifier and type "Crop").
2. Under **IO**:
   1. Set **Input volume** to the scan.
   2. In **Input ROI**, choose **Create new ROI**.
   3. Click **Fit to Volume**. A box with round handles appears. If you can't
      see it, click the **eye** next to *Display ROI*.
3. **Check 1 and 2:** open *Volume information → Input volume* and compare the
   spacing and dimensions with the conversion record.
4. Adjust the box (**check 4**):
   1. In the **green** and **yellow** views, drag the box edge facing the other
      tooth into the air gap. For the upper tooth that's the bottom edge; for
      the lower tooth it's the top edge.
   2. Scroll the **red** view through the tooth. Drag the side handles in,
      leaving about **0.5 mm** around the tooth.
   3. Check all three views again: the whole tooth is inside, and none of the
      other tooth.
5. Under **Advanced**, **untick Interpolated cropping**. The spacing scale is
   then greyed out, because voxels are copied exactly.
6. **Check 5:** under *Volume information → Cropped volume*, the spacing is the
   **same as the input** (0.00504569 mm). Note the dimensions.
7. Click **Output volume** and choose **Create new Volume as…**. Name it
   `<label>_5um`, for example `BASE1_5um`.
8. Click **Apply**.
9. Rename the box: in the **Data** module, double-click `Crop Volume ROI` and
   type `ROI_<label>`, for example `ROI_BASE1`.

## 5. Working copy of the same tooth (`<label>_10um`)

Make this from the full-resolution crop, using the **same box**, so both
versions cover exactly the same region:

1. Set **Input volume** to `<label>_5um`, and **Input ROI** to `ROI_<label>`.
2. Under **Advanced**, **tick Interpolated cropping** and set **Spacing scale** to **2.00x**.
3. **Check 6:** the cropped spacing is **2 × the input** (0.01009138 mm).
4. Click **Output volume** → **Create new Volume as…** and name it `<label>_10um`.
5. Click **Apply**.

## 6. Second tooth

1. Set **Input volume** back to the **full scan**.
2. In **Input ROI**, choose **Create new ROI**, then click **Fit to Volume**.
3. Repeat sections 4 and 5 for the second tooth, with its own label, for
   example `BASE2_5um`, `BASE2_10um` and `ROI_BASE2`.
   ⚠ Choose **Create new Volume as…** every time (check 8).

## 7. Look at each crop

1. Scroll through each new volume in the slice views. Look for:
   - the complete tooth, with nothing missing at the box edges;
   - any obvious cracks, voids or restoration material, which you'll segment
     in Workflow 3.
2. Optional 3D view: open **Volume Rendering**, select a **cropped** volume
   (the `_10um` copy is quicker), turn on the eye icon and choose a CT preset.
   Drag **Shift** until the tooth surface appears.

## 8. Save, then free memory

1. Go to *File → Save*. **Tick only:**
   - `<label>_5um.nrrd` and `<label>_10um.nrrd` for both teeth, in format **NRRD (.nrrd)**;
   - `ROI_<label>.mrk.json` for both teeth.

   **Leave unticked** the full scan (it's already saved) and any `.vp.json`
   rendering presets.
2. Set the folder with **…** or *Change directory for selected files*. Use one
   folder per scan, named after the scan folder:
   ```
   D:\Segmentation\
   ├── NRRD\        <scan>.nrrd   <scan>_conversion_record.txt        (Workflow 1)
   └── Crops\
       └── <scan folder name>\
           ├── BASE1_5um.nrrd   BASE1_10um.nrrd   ROI_BASE1.mrk.json
           └── BASE2_5um.nrrd   BASE2_10um.nrrd   ROI_BASE2.mrk.json
   ```
3. Click **Save** (**check 9**).
4. Open the **Data** module, right-click the full scan and choose **Delete**.
   This removes it from Slicer only; the file stays on disk.

## 9. Print a check of all loaded volumes

Paste this into *View → Python Console* after cropping. Copy the output into
your notes:

```python
for v in slicer.util.getNodesByClass("vtkMRMLScalarVolumeNode"):
    d = v.GetImageData().GetDimensions()
    sp = v.GetSpacing()
    print(f"{v.GetName():55s} spacing {tuple(round(s, 8) for s in sp)} mm | dims {d} | "
          f"size {d[0]*sp[0]:.2f} x {d[1]*sp[1]:.2f} x {d[2]*sp[2]:.2f} mm | {d[0]*d[1]*d[2]/1e9:.2f} GB")
```

✅ **Check**
- Each `_5um` crop has the **same spacing as its scan**.
- Each `_10um` crop has **exactly twice** that spacing.
- The physical sizes (mm) of a tooth's `_5um` and `_10um` versions **match**.
- Each tooth is a sensible size, a few mm in each direction.

## 10. Repeat for the next scan

Load the next scan's `.nrrd` and repeat sections 3–9 with that scan's labels.
For example, scan 2 is BASE3 and BASE4. Assign upper and lower from the scan
notes. Scan 2's slice files say BASE3_BASE3, but the correct label is BASE3_BASE4.

---

## 11. Orientation letters

| Letters | Meaning | In these scans |
|---|---|---|
| **S / I** | Superior / Inferior | Up / down along the scanner's vertical (rotation) axis. Slice number increases towards S. |
| **A / P** | Anterior / Posterior | One in-slice direction |
| **R / L** | Right / Left | The other in-slice direction |

For extracted teeth these are **scanner axes, not anatomy**. Volumes,
thicknesses, cracks and voids don't depend on them. Record anatomical
directions from the tooth itself, for example "S = occlusal for the lower tooth".

---

## 12. Record table (one row per tooth)

| Scan folder | Tooth label | Upper/lower | ROI file | Full-res spacing (mm) | Full-res dims | Working-copy spacing (mm) | Working-copy dims | Notes (cracks, voids, restoration seen?) |
|---|---|---|---|---|---|---|---|---|
| 202609151503_BASE1_BASE2_… | BASE1 | | ROI_BASE1.mrk.json | 0.00504569 | | 0.01009138 | | |
| 202609151503_BASE1_BASE2_… | BASE2 | | ROI_BASE2.mrk.json | 0.00504569 | | 0.01009138 | | |

---

## 13. Troubleshooting

| Problem | Fix |
|---|---|
| Can't see the box or its handles | Click the **eye** next to *Display ROI*. Click inside a slice view, then drag a round handle. |
| Spacing scale is greyed out | Interpolated cropping is off. That's correct for the full-resolution crop. Tick it only for the working copy. |
| Box clips the tooth | Drag that face outwards, then **Apply** again with a new output name (or overwrite the bad crop on purpose). |
| A previous crop was overwritten | *Output volume* wasn't set to a new volume. Re-crop it with **Create new Volume as…**. The saved ROI file reproduces the same box. |
| Slow, or out of memory | Delete unneeded volumes in the **Data** module. Keep the full scan loaded only while cropping. |
| 3D view shows a tiny blob | Volume Rendering is on the full scan. Turn it off, and render only the crops. |
