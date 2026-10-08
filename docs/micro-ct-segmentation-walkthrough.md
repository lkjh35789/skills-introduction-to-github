# Micro-CT in 3D Slicer: from slices to a segmented 3D model

A beginner's walkthrough for the two SkyScan 1172 scans of the BASE specimens
(BASE1/BASE2 at 80 kV, BASE3/BASE4 at 85 kV), reconstructed in NRecon.

> **About Rob's tools (`D:\Extensions`).** Those files live on your computer, so
> this guide doesn't know what they do. Section 2 covers how to install them,
> whatever form they're in. Once you know what each tool does, use it in place
> of the matching manual step below. The manual steps are still worth reading:
> they explain what the tools are doing for you.

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

**The strategy that works on a normal PC:**

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

Open the folder in File Explorer and look for a `README`. Then work out which
of these three forms the tools are in:

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

### Make a volume rendering (the quickest 3D image)

1. Open the **Volume Rendering** module and choose your volume.
2. Click the eye icon to turn it on.
3. Pick a **Preset**. Any CT preset is a starting point.
4. Drag the **Shift** slider until the specimen appears and the air disappears.
5. Rendering mode *GPU Ray Casting* is fastest. Use the preview (downsampled)
   volume here, because GPUs rarely have 28 GB of memory.

### Plan your crops

Each scan contains **two specimens**. For each one:

1. In Volume Rendering, tick **Enable cropping** → *Display ROI*.
2. Drag the ROI box handles until the box tightly encloses one specimen, with a small margin.
3. Rename the ROI in the **Data** module, for example `ROI_BASE1`.

**Confirm which physical specimen is which**, using your scan notes or how the
specimens were stacked in the holder. Neither the slice names nor the image
tell you this, and the BASE3/BASE3 file names make it especially important for
scan 2.

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
