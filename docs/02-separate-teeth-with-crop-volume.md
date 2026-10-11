# Workflow 2: Separate the two teeth in each scan with Crop Volume

**Input:** one `.nrrd` per scan (from Workflow 1). Each contains **two teeth**.
**Output:** one cropped `.nrrd` per tooth (`BASE1_10um.nrrd`, `BASE2_10um.nrrd`,
`BASE3_10um.nrrd`, `BASE4_10um.nrrd`), ready for segmentation.
**Software:** 3D Slicer 5.12.4, **Crop Volume** module (built in)

Cropping does two jobs at once:
1. It **separates the teeth**, because each box contains only one tooth.
2. It makes each tooth's volume **small enough to segment comfortably**.

---

## 1. Settings at a glance

| Setting | Value | Why |
|---|---|---|
| Interpolated cropping | ✔ ticked | Needed to change the voxel size |
| Spacing scale | **2.00x** | 5.05 µm → **10.09 µm** voxels. Fine enough for enamel, dentine and pulp; one tooth is about 0.5 GB instead of about 4 GB. |
| Isotropic spacing | unticked | Voxels are already isotropic |
| Interpolator | Linear | Default |
| Fill value | 0 | Default |
| Output volume | **Create new Volume as…** → `BASEx_10um` | A new, named volume for each tooth |

**Example from scan 1 (first tooth):** cropped spacing 0.01009138 mm in all
three directions, dimensions 901 × 950 × 539 voxels (about 9.1 × 9.6 × 5.4 mm,
about 0.46 GB).

> **Full 5 µm resolution** may be needed later for fine features, such as voids
> in the base material or interface gaps. For that, use **Spacing scale 1.00x**
> with **Interpolated cropping unticked**. Voxels are then copied exactly, with
> no resampling. The same box is then about 3.7 GB.

---

## 2. Load the scan

1. Start Slicer. From the toolbar layout button, choose the **Four-Up** layout.
2. Go to *File → Add Data*, select the scan's `.nrrd`, and click **OK**. Loading
   28 GB takes from about 30 seconds to a few minutes.
3. Adjust the contrast by **left-click and drag** in a slice view.
4. Don't turn on Volume Rendering for the full 28 GB scan. The graphics card
   can't hold it, and the 3D view will show only a tiny blob. If it's already
   on, open the **Volume Rendering** module and click the eye icon to turn it off.

### What the scan looks like (scan 1)

- In the green (coronal) and yellow (sagittal) views, the two teeth are
  **stacked crown to crown**. The upper tooth's crown points **down** and the
  lower tooth's crown points **up**, with an **air gap** between the occlusal
  surfaces.
- The faint circle in the axial view and the vertical lines in the other views
  are the sample holder or jig. Cropping removes most of it.

**Decide which tooth is which before cropping.** Use your scan notes to record
whether BASE1 is the upper or lower tooth. The image and file names can't tell
you. This matters especially for scan 2, because of the BASE3_BASE3
mislabelling.

---

## 3. Crop the upper tooth

1. Open the module menu and choose **Crop Volume**. Or click the magnifier and type "Crop".
2. Under **IO**:
   1. Set **Input volume** to the scan, for example `202609151503_BASE1_BASE2_JIGNITE_ritahardiman_Rec`.
   2. In **Input ROI**, choose **Create new ROI**.
   3. Click **Fit to Volume**. A box outline with round handles appears in
      every view. If you can't see it, click the **eye** icon next to
      *Display ROI*.
3. Adjust the box:
   1. In the **green** view, drag the handle on the **bottom edge** up into the
      air gap, just below the upper tooth's crown.
   2. Check the **yellow** view. The whole crown should be inside the box, and
      none of the other tooth.
   3. Scroll the **red** view through the upper tooth. If there's empty space
      around it, drag the side handles in, leaving about **0.5 mm** around the tooth.
4. Under **Advanced**, apply the settings in section 1:
   - Interpolated cropping ✔
   - Spacing scale **2.00x**
5. Under **Volume information → Cropped volume**, check that **Spacing** reads
   **0.01009138 mm** and that the **Dimensions** look sensible.
6. Click **Output volume** and choose **Create new Volume as…**. Type the
   name, for example `BASE1_10um` (or `BASE2_10um`, according to your notes).
7. Click **Apply**, and wait a minute or two. The cropped tooth replaces the
   full scan in the slice views.

## 4. Crop the lower tooth

1. In **Input ROI**, choose **Create new ROI** again, then click **Fit to Volume**.
2. In the green and yellow views, drag the **top edge** of the box down into
   the air gap, just above the lower tooth's crown. Tighten the sides as before.
3. **Choose Create new Volume as… again**, and name it `BASE2_10um`.

   > ⚠ Don't skip this step. If *Output volume* still shows the first tooth,
   > **Apply** overwrites it.
4. Click **Apply**.

## 5. Check each cropped tooth

1. Use the **Volumes** module or the slice views to scroll through each new
   volume. Check that the whole crown is inside and no part of the other tooth.
2. Optional 3D check: open **Volume Rendering**, select the cropped volume
   (not the full scan), turn on the eye icon, and choose a CT preset. Drag
   **Shift** until the tooth surface appears.

## 6. Save, then free the memory

1. Rename the ROIs `ROI_BASE1` and `ROI_BASE2` in the **Data** module
   (double-click the name).
2. Go to *File → Save*. **Tick only:**
   - `BASE1_10um.nrrd` and `BASE2_10um.nrrd`, in format **NRRD (.nrrd)**;
   - `ROI_BASE1.mrk.json` and `ROI_BASE2.mrk.json`. These record exactly which
     region was cropped, which is useful for your methods or for re-cropping.

   **Leave unticked:**
   - the full scan (it's already on D:, and re-saving would write another 28 GB);
   - `US-Fetal.vp.json`, which is only a rendering preset.
3. Check the **Directory** column, using **…** or *Change directory for
   selected files*. A suggested folder layout:
   ```
   D:\Segmentation\
   ├── scripts\bmp_to_nrrd.py
   ├── Scan1\  BASE1_10um.nrrd  BASE2_10um.nrrd  ROI_BASE1.mrk.json  ROI_BASE2.mrk.json
   └── Scan2\  BASE3_10um.nrrd  BASE4_10um.nrrd  ROI_BASE3.mrk.json  ROI_BASE4.mrk.json
   ```
4. Click **Save**.
5. Open the **Data** module, right-click the full scan and choose **Delete**.
   This removes it from Slicer only; the file stays on D:.

## 7. Repeat for scan 2

Load `D:\202609161017_BASE3_BASE4_JIGNITE_ritahardiman_Rec.nrrd`. Then repeat
sections 2–6 with the names `BASE3_10um` / `BASE4_10um` and
`ROI_BASE3` / `ROI_BASE4`. Assign BASE3 and BASE4 to the upper and lower teeth
from your scan notes.

---

## 8. Orientation letters (S, I, A, P, R, L)

Slicer labels its axes with medical directions:

| Letters | Meaning | In these scans |
|---|---|---|
| **S / I** | Superior / Inferior | Up / down along the scanner's vertical, rotation axis. Slice number increases towards S. |
| **A / P** | Anterior / Posterior | One in-slice direction |
| **R / L** | Right / Left | The other in-slice direction |

For extracted teeth, these letters are **scanner axes, not anatomy**. There is
no "correct" orientation, and volumes and thicknesses don't depend on it.
Record anatomical directions from the tooth itself, for example "S = occlusal
for the lower tooth". Use the A/P/L/R/S/I buttons in the 3D view's pin menu to
look from a given direction.

---

## 9. Record for each tooth

| Tooth | Scan | Position in scan (upper/lower) | ROI file | Spacing scale | Cropped spacing (mm) | Dimensions (voxels) | Notes |
|---|---|---|---|---|---|---|---|
| BASE1 | 1 | | ROI_BASE1.mrk.json | 2 | 0.01009138 | | |
| BASE2 | 1 | | ROI_BASE2.mrk.json | 2 | 0.01009138 | | |
| BASE3 | 2 | | ROI_BASE3.mrk.json | 2 | 0.01009138 | | |
| BASE4 | 2 | | ROI_BASE4.mrk.json | 2 | 0.01009138 | | |

---

## 10. Troubleshooting

| Problem | Fix |
|---|---|
| Can't see the box or its handles | Click the **eye** next to *Display ROI*. Click inside a slice view, then drag the round handles. |
| Box clips part of the tooth | Drag that face of the box outwards and click **Apply** again. Use a new output name, or overwrite the bad crop on purpose. |
| First tooth's crop got overwritten | *Output volume* wasn't set to a new volume. Re-crop it with **Create new Volume as…**. |
| Very slow or out of memory | Delete volumes you don't need from the **Data** module. Keep the full scan loaded only while cropping. |
| 3D view shows a tiny blob | Volume Rendering is on the full 28 GB scan. Turn it off, and render only the cropped volumes. |
