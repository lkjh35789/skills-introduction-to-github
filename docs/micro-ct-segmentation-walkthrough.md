# Micro-CT in 3D Slicer: from slices to a segmented 3D model

A beginner's walkthrough for the two SkyScan 1172 scans of the BASE specimens
(BASE1/BASE2 at 80 kV, BASE3/BASE4 at 85 kV), reconstructed in NRecon.

> **About Rob's tools (`D:\Extensions`).** There are two tools, and they are
> designed to be used in sequence on **teeth**:
>
> 1. **Tooth Segmenter** (Section 12) uses a deep-learning (nnU-Net) model to
>    make `Tooth`, `Pulp` and `Bone` segments.
> 2. **Enamel Dentin Segmenter** (Section 11) splits `Tooth` into `Enamel`,
>    from which you can derive dentine.
>
> Both were written for clinical **CBCT** (about 0.3 mm voxels, HU-like
> values), not 5 µm micro-CT, so read the caveats in each section. Neither tool
> loads data or segments restorative material, so Sections 1–10 still apply.

---

## 0. Concepts in two minutes

| Term | What it means for your data |
|---|---|
| **Slice** | One reconstructed BMP, a 3360 × 3360 cross-section of the sample. |
| **Volume** | The 2,480 slices stacked into a 3D block. Your `.nrrd` file is this block. |
| **Voxel** | A 3D pixel. Yours are 5.04569 µm cubes (isotropic). |
| **Grey value** | How strongly that voxel attenuated X-rays. It rises with density and atomic number, so radiopaque filler glasses (Ba, Sr, Zr, Yb) are bright, resin matrix is mid-grey, and air or voids are dark. |
| **Segmentation** | Labelling each voxel as a class, for example *material*, *pore* or *air*. |
| **3D model** | A surface mesh wrapped around a segment. It can be viewed, measured and exported (STL/OBJ). |
| **Volume rendering** | A 3D picture made straight from the grey values, without segmenting. Good for looking, not for measuring. |

**What 5 µm voxels can and cannot show.** Most dental-composite filler particles
are sub-micron to a few microns across, so they're **not resolved individually**
at this resolution. You can reliably segment specimen geometry, voids and
porosity, and large filler agglomerates or inhomogeneities. A feature needs to be
about 2–3 voxels across (roughly 10–15 µm) to be measured with any confidence.

---

## 1. Check your computer first (this matters with 28 GB volumes)

Each `.nrrd` holds 3360 × 3360 × 2480 voxels at 1 byte each, which is about
**28 GB**. 3D Slicer keeps the whole volume in RAM. A segmentation of the same
size, undo history and 3D surfaces add more on top. As a rule of thumb, plan for
**4–5 × the volume size in RAM**.

Open *Task Manager → Performance → Memory* and note your installed RAM. Then
choose a working size:

| Working size | Dimensions | Voxel size | Approx. size in RAM | Use for |
|---|---|---|---|---|
| Full | 3360 × 3360 × 2480 | 5.05 µm | 28 GB | Only on a workstation with ≥ 128 GB RAM |
| Half (2× downsample) | 1680 × 1680 × 1240 | 10.09 µm | 3.5 GB | Overview, rough segmentation |
| Quarter (4× downsample) | 840 × 840 × 620 | 20.18 µm | 0.44 GB | Quick look, planning crops, volume rendering |
| **Cropped single specimen, full resolution** | depends on specimen | 5.05 µm | (size in mm ÷ 0.00504569)³ bytes, e.g. a 5 mm cube ≈ 1 GB | **Final segmentation and measurements** |

**Your lab PC** (from the Slicer logs) has **120 GB RAM**, an 8-core CPU and
Slicer 5.12.4. It can open a full 28 GB `.nrrd` for viewing and volume
rendering. Segmenting a full scan is still borderline, because the volume, the
segmentation, undo history and surfaces together exceed 120 GB. Rob's tools
need far less than a full scan (Sections 11.2 and 12.3).

**The strategy that works on most PCs:**

1. Load a downsampled preview of the whole scan.
2. Work out where each specimen sits. Each scan holds two specimens.
3. Load **one specimen at a time at full resolution**, cropped to just that specimen.
4. Segment and measure the cropped volume.

Also:

- Work from a local SSD if you can. Reading 28 GB from a USB hard drive is slow.
- Make sure the drive you save to has tens of GB free.

---

## 2. Install 3D Slicer, helper extensions and Rob's tools

### 2.1 3D Slicer

1. Download the latest **Stable Release** for Windows from <https://download.slicer.org>.
2. Install it with the defaults, then open it.

### 2.2 Extensions from the Extensions Manager

1. Open the Extensions Manager: the puzzle-piece icon on the toolbar, or *View → Extensions Manager*.
2. Install these extensions:
   - **SlicerMorph**. It includes the *ImageStacks* module, which loads the BMP slices with downsampling and cropping. This is the memory-safe way to load your data.
   - **SegmentEditorExtraEffects**. It adds *Local threshold*, *Mask volume* and other effects.
   - **SurfaceWrapSolidify** (optional). It builds a closed outer shell of a specimen.
3. Restart Slicer when prompted.

### 2.3 Rob's tools in `D:\Extensions`

Both of Rob's tools are loose scripted modules, form **(c)** below. Add these
two folders as additional module paths:

```
D:\Extensions\ToothSegmenter
D:\Extensions\EnamelDentinSegmenter
```

> ⚠ On the lab PC the paths are currently set to
> `D:\Extensions\…\__pycache__`. That is Python's compiled-cache folder, not
> the module folder. Remove those two entries, add the two folders above, and
> restart Slicer. Otherwise Slicer can't find `Resources\UI\ToothSegmenter.ui`,
> and edits Rob makes to the `.py` files won't take effect.

Then see Sections 11.1 and 12.1.

For any other tools in that folder, open it in File Explorer and look for a
`README`. Then work out which of these three forms each tool is in:

**(a) A packaged extension file**, such as `…-win-amd64.zip` or `.tar.gz`, with a number in the name

1. In the Extensions Manager, click **Install from file…** and choose the file.
2. Restart Slicer.

The package must have been built for **your exact Slicer version**. If
installation fails, ask Rob which Slicer version it was built for.

**(b) An extension source folder**: a top-level `CMakeLists.txt` and sub-folders that each contain a `.py` file

1. Go to *Modules → Developer Tools → Extension Wizard*.
2. Click **Select Extension** and choose the top folder.
3. Accept the offer to load the modules.
4. Restart Slicer.

**(c) Loose scripted modules**: folders that contain `SomeName.py`, often with a `Resources` folder alongside

1. Go to *Edit → Application Settings → Modules*.
2. Next to **Additional module paths**, click **Add**.
3. Choose the folder that **directly contains** the `.py` file.
4. Repeat for each tool, then restart Slicer.

**Check that it worked.** Click the magnifying glass on the toolbar (module
finder) and type the tool's name. If the tool is missing, open
*View → Python Console* and look for red error text, which often means a
missing Python package.

To find out which form you have, paste the output of this Windows Command
Prompt command into the chat:

```
tree D:\Extensions /f
```

---

## 3. Get a 3D image (no segmentation yet)

### Route A: memory-safe preview from the BMP slices (recommended first step)

1. Open the module finder and go to **ImageStacks** (SlicerMorph).
2. For **Input files**, click *Browse* and select the first slice, for example
   `…__rec0096.bmp`. ImageStacks picks up the whole numbered series.
   - Scan 2: the slices are named `BASE3_BASE3__rec….bmp`. That is expected
     (known mislabel). Pick them anyway.
3. Set **Spacing** to `0.00504569` in all three directions. Slicer works in millimetres.
4. Set **Quality** (downsampling) to *preview* or *half resolution*. The module
   shows the estimated memory size, so check it before loading.
5. Set **Colour** to **greyscale** (single component). Your BMPs are 3-channel,
   but the three channels are identical.
6. Under **Output volume**, create a new volume and name it with the correct
   label, for example `BASE1_BASE2_q4` or `BASE3_BASE4_q4`.
7. Click **Load files**.

> Don't drag the BMP slices into Slicer, and don't open them with
> *Add Data* or `slicer.util.loadVolume`. They are 3-channel BMPs, so every
> slice fails with "Unsupported number of components: 1 != 3". The logs show
> this cost more than an hour on 6 October.

### Route B: open the saved `.nrrd`

Only take this route if you have enough RAM (see Section 1).

1. Drag the `.nrrd` into the Slicer window, or use *File → Add Data*.
2. In the **Volumes** module, check *Volume Information → Image Spacing*. It should read 0.00504569.

### Look at it

- The three 2D views are axial (red), sagittal (yellow) and coronal (green).
  Scroll through slices with the mouse wheel. Hold *Shift* and move the mouse
  to link the cursor across all three views.
- Adjust contrast with **window/level**: left-click and drag in a 2D view, or
  use the **Volumes** module, *Display* section. Watch how the dark pores
  separate from the grey material.

### Make a volume rendering (optional)

Volume rendering is only for looking at the scan. You don't need it to segment.

1. Make a small copy first (Section 13, step 2, `Overview`). Never render the
   full 28 GB scan: graphics cards don't have enough memory, and you'll see
   only a tiny blob in the 3D view.
2. Open the **Volume Rendering** module and select `Overview`. Leave the
   **Inputs** section at its defaults.
3. Click the **eye icon** next to *Visibility*, then pick a CT **Preset**.
4. Drag the **Shift** slider until the tooth appears and the air disappears.
5. When you've finished, click the eye icon again to turn rendering off.

### Plan your crops

Separate the teeth with the **Crop Volume** module, not Volume Rendering's
crop option. The step-by-step instructions are in **Section 13, step 3**. In
scan 1 the two teeth are stacked **crown to crown**, with an air gap between
the occlusal surfaces:

- **Upper tooth:** drag the bottom edge of the box up into the gap.
- **Lower tooth:** drag the top edge of the box down into the gap.

**Confirm which tooth is which** (top or bottom) from your scan notes. Neither
the slice names nor the image tell you this, and the BASE3/BASE3 file names
make it especially important for scan 2.

---

## 4. Load one specimen at full resolution

### Option 1: ImageStacks with the ROI (low RAM)

1. Go back to **ImageStacks** with the same input files.
2. Select your ROI under **Region of interest**.
3. Set **Quality** to *full resolution*.
4. Name the output, for example `BASE1_full`.
5. Click **Load files**. Only the voxels inside the box are read.

### Option 2: Crop Volume (if the full `.nrrd` is already loaded)

1. Open the **Crop Volume** module.
2. Set the input volume and your ROI.
3. Leave **Interpolated cropping** unticked. Voxels are then copied exactly,
   not resampled.
4. Click **Apply**.

### Save the cropped volume

1. Go to *File → Save*.
2. Tick only the new cropped volume.
3. Save it as `.nrrd`, for example `D:\…\BASE1_full.nrrd`.

From now on, reopen this small file instead of the 28 GB one.

---

## 5. Segment the specimen

Open the **Segment Editor** module. Set **Segmentation** to *Create new* and
**Source volume** to the cropped volume, for example `BASE1_full`.

### 5.1 Find a sensible threshold

1. Click **Add** to create a segment and rename it `Material` (double-click the name).
2. Select the **Threshold** effect. The 2D views preview the selected voxels in colour.
3. Drag the lower slider up until the air and pores are excluded and the
   material is fully covered. Alternatively, open *Automatic threshold* and
   choose **Otsu**, which picks the gap between the two peaks of the histogram.
4. Write down the threshold values you used. You'll need them for your methods section.
5. Click **Apply**.

### 5.2 Clean up noise

1. Select the **Islands** effect and choose **Keep largest island**.
2. Click **Apply**. This removes isolated bright specks outside the specimen.

### 5.3 Segment the internal pores

You'll make two helper segments, then get the internal pores by subtracting one
from the other.

1. Make the helper segment `AllDark`:
   1. **Add** a new segment named `AllDark`.
   2. Select **Threshold** and set the range from the minimum up to just
      below your material threshold.
   3. Click **Apply**. `AllDark` now contains the outside air and the internal pores.
2. Make the helper segment `OutsideAir`:
   1. **Add** a segment named `OutsideAir`.
   2. Select **Logical operators**, choose **Copy**, and copy from `AllDark`.
   3. Select **Islands** and choose **Keep largest island**. The surrounding air
      is one big connected region, so this keeps only that.
3. Make the `Pores` segment:
   1. **Add** a segment named `Pores`.
   2. Use **Logical operators** to **Copy** from `AllDark`.
   3. Use **Logical operators** again and **Subtract** `OutsideAir`.
4. Remove noise from `Pores`:
   1. Select **Islands** and choose **Remove small islands**.
   2. Set a minimum size, for example 27 voxels (3 × 3 × 3, about 15 µm).
      Anything smaller can't be told apart from noise. Report the cut-off you used.
5. Delete or hide the two helper segments.

> **Caveat:** a pore that breaks through the specimen surface connects to the
> outside air, so it's removed together with it. Mention this in your methods.
> Alternatively, use **Wrap Solidify** (SurfaceWrapSolidify) on `Material` to
> make a closed `Envelope` segment, then set `Pores = Envelope − Material`.

### 5.4 Useful tools while you segment

| Tool | What it's for |
|---|---|
| **Masking** section at the bottom of Segment Editor | *Editable area: inside Material* limits any effect to inside the specimen. |
| **Scissors** | Cut away the holder, jig or mounting material in 3D. Rotate the 3D view, draw around the part to remove, and choose *Erase inside*. |
| **Paint / Erase** | Small manual fixes. |
| **Smoothing** | Use for display only. It changes measured volumes, so measure **before** you smooth. |

---

## 6. Create the 3D model

1. In Segment Editor, click **Show 3D**. Slicer builds a surface for each segment.
2. In the **Segmentations** module, set the `Material` opacity to about 0.2 so
   the `Pores` show through.
3. For publication-style images, change the background via
   *View → Application Settings → Views*, then capture with the camera icon
   (Screen Capture module).

---

## 7. Measure

1. Open the **Segment Statistics** module.
2. Set the segmentation and the source volume.
3. Tick *Labelmap statistics*, and optionally *Closed surface* for surface area.
4. Click **Apply**. You get the volume (mm³) and voxel count per segment.
5. Export the table: right-click → *Copy table*, or save it as CSV.

Calculate porosity as:

```
porosity (%) = V_Pores / (V_Material + V_Pores) × 100
```

Pore-size distribution: in the **Islands** effect, *Split islands to segments*
on `Pores` gives one segment per pore. Segment Statistics then gives each pore's
volume. This produces many segments, so do it on cropped data only.

---

## 8. Save and export

| What | How |
|---|---|
| **Whole session** | *File → Save*. Save the scene as `.mrb` (single file) or as a folder. **Untick any 28 GB volume** so Slicer doesn't rewrite it. |
| **Segmentation only** | Saved as `.seg.nrrd` in the same dialog. You can reopen it later on top of the cropped volume. |
| **3D surface for other software** | Go to the **Segmentations** module → *Export/import models and labelmaps* → *Export to files*. Choose **STL** or **OBJ**, keep units in mm, and click *Export*. |

---

## 9. Comparing the two scans fairly

Scan 1 was acquired at 80 kV / 124 µA and scan 2 at 85 kV / 118 µA. Grey values
are therefore **not directly comparable** between scans, and the same numeric
threshold may not mean the same material boundary.

- Open both `__rec.log` files in Notepad and compare the NRecon settings, in
  particular the *Minimum/Maximum for CS to Image Conversion* lines (the
  contrast mapping), *Beam Hardening Correction*, *Ring Artifact Correction* and
  *Smoothing*. If these differ, the 8-bit grey scales differ too.
- Use the **same method** (for example Otsu, plus the same minimum pore size)
  rather than the same number on every specimen. Record the resulting threshold
  values per specimen.
- Segment one specimen twice, with thresholds a few grey levels either side,
  to see how sensitive the porosity is to your choice. Report that sensitivity.

---

## 10. Per-specimen checklist

- [ ] Preview loaded; specimen positions identified and labelled (BASE1/2/3/4)
- [ ] Cropped full-resolution volume saved as `.nrrd`
- [ ] `Material` segmented: threshold method and values recorded
- [ ] Holder/jig removed (Scissors)
- [ ] `Pores` segmented: minimum pore size recorded
- [ ] Segment Statistics exported to CSV
- [ ] 3D model shown, screenshot captured, STL exported
- [ ] Scene saved (without re-saving the 28 GB volume)

Keeping these numbers per specimen in one spreadsheet (formulation ID, scan
settings, threshold, porosity, pore count, largest pore) gives you tidy,
ML-ready features later. Porosity sits naturally between handling and placement
(RQ1) and mechanical performance (RQ3).

---

## 11. Rob's Enamel Dentin Segmenter (tooth specimens only)

### 11.0 What it is, and when to use it

The module is `D:\Extensions\EnamelDentinSegmenter\EnamelDentinSegmenter.py`,
version `0.2.0-round1-threshold-methods`. It appears in Slicer under
*Modules → Segmentation → Enamel Dentin Segmenter*.

It **does not load or segment a scan from scratch**. It takes a segmentation
you've already made, containing a tooth, its pulp and "bone", and adds an
**`Enamel`** segment in two steps:

1. **Generate Enamel Seed (Phase 1a).** Peels a 1-voxel outer shell off the
   tooth, then keeps only the parts of that shell that are farther than *X* mm
   from `Bone` (to stay above the CEJ) and farther than *Y* mm from `Pulp`.
2. **Grow Enamel from Seed (Phase 1b).** Picks an intensity threshold from the
   tooth-minus-pulp voxels (Otsu by default). It then grows the seed through
   connected voxels above that threshold. Highly mineralised enamel
   (~96 wt% apatite) is brighter than dentine (~70 wt%), so the growth stops at
   the dentine–enamel junction (DEJ).

It was written for **clinical CBCT of teeth in the jaw**. Use it only if your
BASE specimens are **teeth** (for example, restored extracted teeth). If they
are blocks or discs of restorative material, there is no enamel or dentine to
find, so skip this section.

### 11.1 Installation and Python packages

1. Go to *Edit → Application Settings → Modules → Additional module paths → Add*.
2. Choose `D:\Extensions\EnamelDentinSegmenter` (not its `__pycache__` subfolder) and restart Slicer.
   - Alternatively, drag `EnamelDentinSegmenter.py` onto the Slicer window and
     accept the offer to load it as a module.
   - The README describes copying the folder into `AppData`. That works too,
     but the module-path method is simpler and leaves Rob's folder untouched.
3. Open *View → Python Console* and run these once, then restart Slicer:

   ```python
   slicer.util.pip_install('scikit-image')   # required for "Grow Enamel from Seed"
   slicer.util.pip_install('scikit-learn')   # optional: K-means and GMM threshold methods
   ```

   Without scikit-image, *Grow* fails with "No module named 'skimage'". The
   *GPU Acceleration* section (CuPy) is optional and only helps on an NVIDIA GPU
   with enough GPU memory.

Other files in the folder:

| File | What it is |
|---|---|
| `EnamelDentinSegmenter.py.old` | Rob's previous version. Slicer ignores it. |
| `__pycache__\…pyc` | Python's compiled cache. Ignore it. |
| `README.md` | Describes the older percentile version, so it is partly out of date. The *help* text inside the module is current. |
| `Refresh_Command.txt` | `slicer.util.reloadScriptedModule("EnamelDentinSegmenter")`. Paste it into the Python Console to reload the module after Rob sends a new `.py`, without restarting Slicer. |

### 11.2 Memory: never run it on a full scan

The module turns whole-volume masks into NumPy arrays and runs two Euclidean
distance transforms. Peak memory is roughly **30–40 bytes per voxel** of the
input volume:

| Input | Voxels | Approx. peak RAM |
|---|---|---|
| Full scan (3360 × 3360 × 2480) | 28 billion | > 1 TB: impossible |
| One cropped tooth at full 5 µm resolution (e.g. 8 × 8 × 10 mm) | ~5 billion | ~150–200 GB: impossible on a PC |
| Same tooth at half resolution (10 µm) | ~620 million | ~20–25 GB |
| Same tooth at quarter resolution (20 µm) | ~80 million | ~3 GB |

Enamel is 1–2.5 mm thick, so **10–20 µm voxels are plenty for enamel and
dentine geometry**. Run the module on a **cropped, downsampled tooth** using
ImageStacks with the ROI and *half* or *preview* quality. Keep the full 5 µm
crops for fine features such as voids in the base or restorative material
(Section 5.3).

The segmentation must be made on **the same volume** you give the module (the
same crop and resolution).

### 11.3 Prepare the segments it needs

The module looks up segments **by exact name**. In Segment Editor, on the
cropped and downsampled tooth volume, create these segments:

| Segment name | What it should contain | How |
|---|---|---|
| `Tooth` (any name **not** containing pulp/bone/enamel/dentin) | The whole tooth, **excluding any restoration or base material** | Threshold above air → Islands *Keep largest island*. Then Logical operators *Subtract* `Restoration`. |
| `Pulp` | The pulp chamber and canals | Threshold the dark range, then use *Masking: inside Tooth* and Islands *Keep largest island*. Or use *Grow from seeds*. |
| `Bone` | Something below the CEJ (see note) | See the note below this table. |
| `Restoration` (optional, recommended) | The restorative or base material | Threshold. It is often brighter than enamel. |

**About `Bone`.** Extracted teeth have no bone, but the module refuses to run
without a non-empty `Bone` segment. It uses `Bone` only as "stay at least *X* mm
away from here", to stop the enamel seed running down the root. Two stand-ins work:

- If the teeth are set in a mounting resin or jig up to near the CEJ, segment
  that mount and name it `Bone`.
- Otherwise, use Paint or Draw on a few slices to mark the root surface below
  the CEJ, run *Fill between slices*, and name the result `Bone`. Then set
  *Min Distance from Bone* small, for example 0–0.5 mm.

**Why the restoration must stay out of `Tooth`.** Radiopaque restoratives (Ba,
Sr, Zr or Yb glass fillers) are often as bright as or brighter than enamel. If
the restoration is inside `Tooth`, Otsu is skewed and enamel grows into the
restoration.

### 11.4 Run it

1. Under **Inputs**, select:
   - *Input Volume*: the cropped tooth volume.
   - *Segmentation*: your segmentation.
   - *Tooth Segment*: your tooth segment. **Check this selection.** The list
     shows every segment except Pulp/Bone/Enamel/Dentin, so `Restoration` and
     `Air` appear there too.
2. Under **Phase 1a**, set:
   - *Min Distance from Bone*: default 1.5 mm.
   - *Min Distance from Pulp*: default 0.5 mm.
   - *Min Enamel Voxels*: default 100.
3. Click **Generate Enamel Seed**. A blue `Enamel` shell appears. Check it
   covers the crown surface and not the root. If it doesn't, click **Undo**,
   change the distances and run it again. Undo goes back **one step only**.
4. Under **Phase 1b**, choose a *Threshold Method*. Start with **Otsu** and
   offset 0. Then click **Grow Enamel from Seed**.
5. **Units warning.** The spin boxes are labelled "HU" because the module was
   written for CBCT. Your data are **8-bit grey values (0–255)**, so the boxes
   mean grey levels:
   - *Manual HU* defaults to 1500. That is above 255, so the growth domain
     would be empty. Type a grey value instead, for example 170.
   - *Otsu Offset* steps by 50, which is far too coarse at this scale. Type
     small values such as −5 or +5.
6. Open *View → Python Console*. The module prints the threshold it actually
   used (lines starting `[DIAG]`). **Record that number** for each tooth.
7. If the enamel looks too thick or too thin, click **Undo**, change the method
   or offset, and grow again. *Multi-Otsu* and *GMM (3-component)* can help
   when the histogram has three populations, such as dentine, enamel and a
   bright artefact.

### 11.5 Finish: dentine, checks and export

1. **Make `Dentin` by hand** (the module doesn't do this yet). Add a segment,
   **Copy** `Tooth`, then **Subtract** `Enamel` and **Subtract** `Pulp`.
2. **Check the seed shell at cavity walls.** The seed is kept unconditionally,
   so a 1-voxel shell may remain on the cavity walls next to the restoration
   and on any exposed root surface. Remove it with **Erase**, or with
   **Margin** → shrink by 1 voxel then grow by 1 voxel, if it matters.
3. Then continue with Sections 6–8: Show 3D, Segment Statistics (enamel and
   dentine volumes) and export.

### 11.6 Notes for Rob

These are observations from reading the code. Nothing has been changed.

- The README still describes the older v1 percentile workflow.
- The "HU" labels and the defaults (Manual HU 1500, offset step 50) assume CBCT
  Hounsfield units. Grey-level-aware defaults would help with 8-bit micro-CT.
- `scikit-image` is imported at the top of `growEnamelFromSeed`, outside the
  `try`, and there's no install hint. A check like the scikit-learn one would
  make the failure clearer.
- `boneDistanceMm` is passed to `growEnamelFromSeed` but not used there.
- Memory scales at about 30–40 bytes per voxel (float64 distance transforms,
  full-volume mask copies, and the undo copy). A warning above roughly 300
  million voxels, or computing on a bounding-box crop of the tooth, would
  prevent out-of-memory crashes on micro-CT data.

---

## 12. Rob's Tooth Segmenter (nnU-Net deep learning)

### 12.0 What it is

The module is `D:\Extensions\ToothSegmenter\ToothSegmenter.py`, v1.0 (January 2026).
It appears in Slicer under *Modules → Segmentation → Tooth Segmenter*.

You draw a box (ROI) around one tooth. The module sends the voxels inside the
box to a trained **nnU-Net** model, which returns a segmentation node named
`Seg_<volume>` with three segments:

| Label | Segment | Colour |
|---|---|---|
| 1 | `Tooth` (enamel + dentine together) | ivory |
| 2 | `Pulp` | red |
| 3 | `Bone` | beige |

These are exactly the segment names that **Enamel Dentin Segmenter**
(Section 11) needs. The intended pipeline is therefore:

**Tooth Segmenter → Enamel Dentin Segmenter → Dentin = Tooth − Enamel − Pulp → export (STL/FEA)**

The model was trained on Rob's HPC from a dataset called
`Dataset001_ToothFairy`. ToothFairy is a public **clinical CBCT** dataset,
with about 0.3 mm voxels and teeth set in jaw bone. Your micro-CT is very
different (see 12.3).

### 12.1 What must be in place before it can run

| Requirement | How to check / fix |
|---|---|
| **Module folder** | `D:\Extensions\ToothSegmenter\` contains `ToothSegmenter.py` and `Resources\UI\ToothSegmenter.ui` ✓. Both are confirmed present, and every widget the code uses exists in the `.ui`. The module path must point at this folder, not at `__pycache__` (Section 2.3). |
| **PyTorch + nnU-Net** | Open the module and look at the *Dependencies* status. If it shows ✗, click **Install Dependencies**, wait 5–10 minutes, then restart Slicer. This installs CPU-only PyTorch, which is fine for small inputs (12.3). |
| **Trained model files** | The module path is hard-coded to `D:\nnunet_models\Dataset001_ToothFairy\nnUNetTrainer__nnUNetPlans__3d_fullres\`. That folder needs `dataset.json`, `plans.json` and `fold_0` … `fold_4`, each containing `checkpoint_final.pth` (about 2 GB in total). The model lives on Rob's HPC project (`punim2702`), so ask Rob to copy it if it isn't there. The README's `~/nnunet_models` path is out of date. The code uses `D:\nnunet_models`. |

### 12.2 How to run it (the module's own workflow)

1. **Input Volume**: choose the volume.
2. **Create ROI**: a green box (11 × 11.7 × 27.8 mm by default) appears at the
   centre of the volume. Drag its handles to enclose one tooth.
3. **Validate ROI**: this checks two things:
   - Each side of the box must be 5–50 mm. Under 5 mm is an error; over 50 mm
     is a warning.
   - The voxel spacing should be within 0.05 mm of 0.3 mm. If not, you get a
     **warning only**, and you can click *Proceed anyway*.
4. **Run Segmentation**: a progress bar runs, and then a "Segmentation
   complete!" message appears.

### 12.3 ⚠ Using it on your micro-CT: three things to get right

**1. Never run it on the 5 µm data. Downsample to about 0.3 mm first.**

At 5 µm, even the default box holds about 12 billion voxels. The module
converts them to 32-bit floats (about 50 GB). nnU-Net then resamples them, and
it resamples its output probabilities back to the input grid as 4 classes ×
32-bit. That is hundreds of GB, which is far beyond 120 GB, so Slicer will
freeze or crash.

nnU-Net resamples everything to its training spacing (about 0.3 mm) anyway, so
make a 0.3 mm copy first:

1. Open the **Crop Volume** module.
2. Set *Input volume* to the full `.nrrd`. For *Input ROI*, use a box around one tooth.
3. Tick **Interpolated cropping**.
4. Set *Spacing scale* so the output spacing is 0.3 mm (about **59.5**). Check
   the *Output spacing* readout.
5. Click *Apply*. You get a small volume (for example 40 × 40 × 40 voxels)
   named something like `BASE1_tooth_0.3mm`.
6. Run Tooth Segmenter on that volume. Validation passes the spacing check,
   and CPU inference takes under a minute.

**2. Expect the model to struggle. Check the result against the image.**

- **Intensities.** The model learned CBCT grey values. Yours are 8-bit micro-CT
  values (0–255, and possibly only 0–33; check this first). If the model's
  `plans.json` uses `CTNormalization`, your values will sit far outside what it
  saw in training. Look for `"normalization_schemes"` in `plans.json`.
  `ZScoreNormalization` is more forgiving.
- **Anatomy.** It expects a tooth in **jaw bone**. An extracted tooth in air, or
  in a mounting jig, may come back with an empty `Bone` segment, or with the jig
  labelled as `Bone`.
- **Resolution.** At 0.3 mm a tooth is only about 30 voxels across, so the
  result is a coarse outline. It isn't a 5 µm-accurate boundary.

**3. Watch for the silent "dummy" result.**

If nnU-Net fails for any reason (model files missing, out of memory, an import
error), the module **does not stop**. It writes "Falling back to dummy
segmentation" to the Python Console and puts a **fake sphere** labelled `Tooth`
in the middle of the box. It still shows "Segmentation complete!". Two other
details:

- The intended inner "Pulp" sphere never appears. The `elif` order in the code
  means every voxel inside the sphere is labelled `Tooth`.
- The fallback loops over every voxel in pure Python, so on a large box it can
  run for hours.

**After every run, open *View → Python Console* and confirm there is no
"nnUNet inference failed" line.** A perfect ball-shaped "tooth" means the run failed.

### 12.4 Is it worth it for micro-CT? A suggested approach

An extracted tooth scanned at 5 µm sits in air or mounting material, so the
contrast is very high. A threshold (Section 5.1) gives `Tooth`, and the dark
cavity inside gives `Pulp`. Both are far more accurate than a 0.3 mm model
output. Rob's nnU-Net earns its keep on clinical CBCT, where teeth touch bone of
similar density.

Suggested approach for each tooth:

1. **Try Tooth Segmenter** on a 0.3 mm copy (12.3). It's quick, and it shows
   whether the model generalises.
2. **Make the real segments by thresholding** on a 10–20 µm crop of the same
   tooth:
   - `Tooth`: threshold above air, then Islands *Keep largest island*.
   - `Pulp`: dark range with *Masking: inside Tooth*, then Islands.
   - `Restoration`: a high threshold, then subtract it from `Tooth` (Section 11.3).
   - `Bone`: the mount, or a painted region below the CEJ (Section 11.3).

   If the nnU-Net output looked sensible, you can use it as a guide. Grow it
   with **Margin**, then use it as the *Masking → Editable area* for these thresholds.
3. **Run Enamel Dentin Segmenter** (Section 11) on that crop. Then make
   `Dentin = Tooth − Enamel − Pulp`.
4. Use **full 5 µm crops** only for fine features, such as voids in the base
   or restorative material (Section 5.3).

Recording which route produced each segment (nnU-Net-guided or threshold-only)
keeps the method reproducible for your thesis.

### 12.5 Notes for Rob

These are observations from reading the code. Nothing has been changed.

- **Silent fallback.** A failed inference produces a sphere and a success
  dialog. Consider raising an error instead. The fallback's `elif` also never
  assigns label 2 (Pulp), and the voxel-by-voxel Python loop is very slow on
  large ROIs.
- **No guard on input size.** Micro-CT voxels (5 µm) make even a small ROI
  enormous. A voxel-count check, or automatic resampling to the model spacing
  before prediction, would prevent crashes. A spacing mismatch is currently
  only a warning.
- **README is out of date.** The model path (`~/nnunet_models`) and the
  `nnUNet_results` example (`modelPath.parent` vs `.parent.parent`) differ from
  the code.
- **Segment naming.** Names use the `LabelValue` tag if present, otherwise
  sequential order. If a label is absent, for example no pulp, later segments
  could be misnamed.
- **Validation tooltip.** The *Validate ROI* tooltip mentions "content" checks, but only size and spacing are checked.
- **CPU-only install.** *Install Dependencies* installs CPU-only PyTorch even
  on GPU machines.

---

## 13. Step-by-step protocol: two teeth → separate teeth → enamel, dentine, pulp

This protocol needs only built-in Slicer tools. It doesn't need Rob's nnU-Net
model. Do it for **one tooth first**, then repeat for the others. Scan 1 holds
BASE1 and BASE2, and scan 2 holds BASE3 and BASE4.

The plan:
1. Load the scan.
2. Crop each tooth into its own smaller volume. **This is what separates the
   two teeth.**
3. In each cropped volume, segment `Tooth`, then `Pulp`, then `Enamel`.
4. Derive `Dentin` from those, then check, measure and save.

Keep a lab-notebook table as you go (template in step 9).

### Step 0: Before you start (once)
1. Restart Slicer so it starts empty.
2. Open *View → Python Console*. You'll paste a few short commands into it.
3. Change the view layout with the layout button on the toolbar (the grid
   icon). Choose **Four-Up**: three slice views plus a 3D view.

### Step 1: Load the scan
1. Go to *File → Add Data → Choose File(s) to Add*. Select
   `D:\BASE1_BASE2_JIGNITE.nrrd`, or whichever scan-1 `.nrrd` you have, and click **OK**.
2. Wait for it to load. It is 28 GB; your PC has 120 GB RAM, so this is fine.
3. Improve the contrast. In a slice view, **left-click and drag**: up/down
   changes brightness, left/right changes contrast. Or open the **Volumes**
   module and pick a *Window/Level preset*.
4. Learn the **Data Probe** at the bottom-left of the window. It shows the grey
   value under your mouse. Hover over air, dentine, enamel, the pulp and any
   restoration or base material, and **write down typical values for each**.

### Step 2: Get an overview and find both teeth (optional but helpful)
1. Open the **Crop Volume** module.
2. Set *Input volume* to your scan.
3. Set *Input ROI* to *Create new ROI*, then click **Fit to Volume**.
4. In *Advanced*, tick **Interpolated cropping** and set *Spacing scale* to **8**.
   The output spacing is about 0.04 mm and the copy is only about 55 MB.
5. Set *Output volume* to *Create new volume* and rename it `Overview`. Click **Apply**.
6. Open **Volume Rendering**, select `Overview`, click the eye icon and choose
   a CT preset. Drag *Shift* until you see both teeth clearly.
7. Work out how the teeth sit: stacked top and bottom, or side by side. Find
   any mounting jig. Then **decide which tooth is BASE1 and which is BASE2**
   from your scan notes. The image can't tell you this.

### Step 3: Crop each tooth into its own volume (this separates them)
1. In **Crop Volume**, set *Input volume* to the **full scan**, not `Overview`.
2. Set *Input ROI* to *Create new ROI* and rename it `ROI_BASE1`. Click **Fit to Volume**.
3. In the slice views, drag the ROI box handles until the box tightly encloses
   **only the first tooth**, with about 0.5 mm margin. Check it in all three views.
4. In *Advanced*, tick **Interpolated cropping** and set *Spacing scale* to **2**.
   - The output spacing is about 0.0101 mm (10 µm): fine enough for enamel,
     dentine and pulp, and a manageable size (about 1 GB).
   - If later steps feel slow, use **4** (20 µm) instead.
5. Set *Output volume* to a new volume named `BASE1_10um`. Click **Apply**.
6. Repeat steps 1–5 for the second tooth with a new ROI (`ROI_BASE2`, output `BASE2_10um`).
7. **Save the cropped volumes.** Go to *File → Save*. Untick everything except
   `BASE1_10um` and `BASE2_10um`. Set the folder, for example
   `D:\Segmentation\Scan1\`, and click **Save**.
8. **Free memory.** In the **Data** module, right-click the full 28 GB scan and
   choose *Delete*. This removes it from Slicer only; the file stays on disk.

If a box can't avoid catching part of the other tooth or the jig, that's fine.
You'll remove it in step 4.3.

### Step 4: `Tooth`, the hard tissue (enamel + dentine)
1. Open **Segment Editor**:
   - *Segmentation*: *Create new segmentation*, renamed `Seg_BASE1`.
   - *Source volume*: `BASE1_10um`.
   - Click **Add** and rename the segment `Tooth` (double-click its name).
2. Select **Threshold**:
   - Open *Automatic threshold* and choose **Otsu**. This separates air from tissue.
   - Check the slice views. All dentine and enamel should be coloured, and air
     shouldn't be. Nudge the **lower** value if needed, using your Data-Probe
     values from step 1.4. **Write down the final range.**
   - Click **Apply**.
3. Select **Islands** → **Keep largest island** → **Apply**. This removes
   specks, and any bits of the other tooth or the jig that aren't touching this tooth.
   - If the jig or mounting material **touches** the tooth and is still attached,
     use **Scissors**. Rotate the 3D view, choose *Erase inside*, and draw
     around the unwanted part. Repeat until only the tooth remains.
4. **If the tooth contains a restoration or base material:**
   1. Add a segment named `Restoration`.
   2. Select **Threshold** and set the range to the material's grey values
      (it is often brighter than enamel). Click **Apply**.
   3. Clean it with **Islands → Keep largest island**.
   4. Select `Tooth`, then use **Logical operators** → **Subtract** →
      modifier segment `Restoration` → **Apply**.

   If the material's grey values overlap enamel, outline it with **Paint** or
   **Draw** on a few slices, then run *Fill between slices*.

### Step 5: `Pulp`, using Grow from seeds
The pulp is as dark as air and connects to the outside at the root tip, so a
plain threshold can't separate the two. Instead, you mark a few examples of
"pulp" and "outside", and Slicer fills in the rest.

1. Click the **eye icon** next to `Tooth` to hide it. Hidden segments don't take
   part in Grow from seeds, so `Tooth` won't change.
2. Add two segments, `Pulp` and `Outside`.
3. Select **Paint** and make the brush smaller than the pulp chamber.
   - With `Pulp` selected, paint short strokes **inside** the pulp chamber and
     along each root canal, on 5–10 slices spread through the tooth. Use the
     axial, sagittal and coronal views.
   - With `Outside` selected, paint strokes in the **air around** the tooth on
     the same slices. Add extra strokes right at each **root tip**, where the
     canal opens.
4. At the bottom of Segment Editor, open **Masking**:
   - Tick **Editable intensity range**.
   - Set it from the minimum up to **just below your `Tooth` lower threshold**.
     Now only dark voxels can become `Pulp` or `Outside`.
5. Select **Grow from seeds** and click **Initialize**. After a short wait a
   preview appears.
   - If the pulp leaks out of the root tip, add more `Outside` strokes there.
   - If parts of the canal are missing, add `Pulp` strokes there.
   - The preview updates as you add strokes. When it looks right, click **Apply**.
6. Select `Pulp`, then **Islands → Keep largest island → Apply**.
7. **Untick** *Editable intensity range* in Masking. Delete the `Outside`
   segment and make `Tooth` visible again.

### Step 6: `Enamel`
1. **Get an objective enamel/dentine threshold.** Paste this into the Python
   Console. It calculates Otsu's threshold **inside the tooth only**:
   ```python
   from skimage.filters import threshold_otsu
   vol = slicer.util.getNode("BASE1_10um"); seg = slicer.util.getNode("Seg_BASE1")
   a = slicer.util.arrayFromVolume(vol)
   t = slicer.util.arrayFromSegmentBinaryLabelmap(seg, seg.GetSegmentation().GetSegmentIdBySegmentName("Tooth"), vol)
   print("Enamel/dentine Otsu threshold:", threshold_otsu(a[t > 0]))
   ```
   Write the number down. Check it is between your typical dentine and enamel
   values from step 1.4.
2. Add a segment named `Enamel`.
3. In **Masking**, set:
   - **Editable area**: `Tooth`.
   - **Modify other segments**: **Allow overlap**. This keeps `Tooth` complete.
4. Select **Threshold**. Set the lower value to the Otsu number and the upper
   value to the maximum. Check the crown: the enamel cap should be coloured and
   the dentine underneath shouldn't be. Click **Apply**.
5. Select **Islands** → **Remove small islands**, with *Minimum size* about
   10 000 voxels → **Apply**. This removes bright specks inside the dentine.
   Use *Remove small islands* rather than *Keep largest*, because a cavity can
   split the enamel into separate pieces.
6. Set **Editable area** back to **Everywhere**.

*Optional cross-check:* run Rob's **Enamel Dentin Segmenter** on the same
segmentation (Section 11). It needs a `Bone` stand-in segment (Section 11.3).
It grows enamel inward from the crown surface, which can clean up bright spots
that aren't enamel.

### Step 7: `Dentin`
1. Add a segment named `Dentin`.
2. Use **Logical operators**:
   1. **Copy**, with modifier `Tooth`. Click **Apply**.
   2. **Subtract**, with modifier `Enamel`. Click **Apply**.
   3. **Subtract**, with modifier `Pulp`. Click **Apply**. This should change
      nothing, but it is a safe check.

You now have `Tooth` (= Enamel + Dentin), `Enamel`, `Dentin`, `Pulp`, and
`Restoration` if there is one.

### Step 8: Check, view in 3D, measure, save
1. **Check.** Scroll through all three views. In particular, check:
   - the dentine–enamel junction;
   - where the enamel ends at the cemento-enamel junction (CEJ);
   - the root tips.

   Fix small errors with **Paint** or **Erase**, working on the correct segment.
2. **View in 3D.** Click **Show 3D**. In the **Segmentations** module, set the
   `Enamel` opacity to about 0.4 to see the dentine and pulp inside.
3. **Measure.** Open **Segment Statistics**. Set *Segmentation* to `Seg_BASE1`
   and *Scalar volume* to `BASE1_10um`, then click **Apply**. You get the
   volume in mm³ of each segment. Right-click the table to copy it, or export it as CSV.
4. **Save.**
   - *File → Save* saves `Seg_BASE1.seg.nrrd` next to `BASE1_10um.nrrd`. You
     can also save the whole scene as `.mrb`.
   - For STL meshes, go to the **Segmentations** module → *Export to files* →
     STL.
5. Repeat steps 4–8 for `BASE2_10um`. Then repeat the whole protocol for scan 2
   (BASE3 and BASE4) with the other `.nrrd`. **Grey values differ between the
   scans** (80 kV vs 85 kV), so don't reuse scan 1's numbers. Repeat the
   measurements in steps 1.4, 4.2 and 6.1.

### Step 9: Record for reproducibility
Fill in one row per tooth:

| Tooth | Scan | Crop spacing | Tooth threshold | Otsu enamel threshold (final if adjusted) | Pulp method | Islands min size | V_enamel | V_dentin | V_pulp | Notes |
|---|---|---|---|---|---|---|---|---|---|---|
| BASE1 | 1 (80 kV) | 0.0101 mm | | | Grow from seeds | 10 000 vox | | | | |

**Tips**
- **Undo:** Ctrl+Z works in Segment Editor. Save after each step.
- **Speed:** if Islands or Grow from seeds take minutes, that's expected at
  10 µm. For a faster practice run, re-crop at spacing scale 4.
- **Selected segment:** every effect acts on the segment that is **highlighted**
  in the list. Before clicking Apply, check that the right one is selected.
