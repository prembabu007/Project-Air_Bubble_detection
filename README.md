# Automatic Detection of Air Bubbles in Industrial CT Data

My Master's project at the University of Koblenz.
The goal is simple: **find the air bubbles in a CT scan automatically**, count them and measure how much air they hold.

## What is this project about?

I have CT scans of a sample made of many small spheres packed together (like marbles in a jar).
Inside the sample there are small air bubbles. In the CT pictures, air is **dark** and material is **bright**.

Finding the bubbles by eye is not realistic, because one scan has 800 slices and the bubbles are small.
The problem is that bubbles are not the only dark things. The gaps between the spheres and the edge of the sample are dark too.
So the program has to tell real bubbles apart from these other dark parts.

## How it works

The whole pipeline is in one Jupyter notebook (`XRT_Image_Processing.ipynb`), made to run in Google Colab with a T4 GPU.

1. **Reconstruction**: the 2001 projections of a scan are turned into a 3D volume (800 x 800 x 800 voxels, 0.4 mm each) with the ASTRA toolbox (FDK_CUDA).
2. **Denoising**: non-local means removes noise but keeps the edges.
3. **Smoothing**: a Gaussian filter with sigma 0.7.
4. **Global threshold (Otsu)**: finds the sphere pack.
5. **Sphere pack and envelope**: the envelope is the outline of the pack.
6. **Local threshold**: finds the dark air inside the pack (the threshold comes from the data, I don't type it in by hand).
7. **Air pockets**: air inside the envelope, kept a bit away from the surface because the surface is blurred.
8. **Cleaning**: tiny regions are removed, DBSCAN groups close air voxels into bubbles, and very big branching air networks are treated as pore network (not bubbles).
9. **Adaptive edge filter**: the blur at the sample edge looks like thin dark clusters. The filter measures the blur width and removes clusters near the surface that are too small, too thin, too elongated or too big. Each cluster is either kept or removed as a whole.
10. **Result**: number of bubbles and their volume.

There are two modes in the notebook:

| | Normal data | Test scans |
|---|---|---|
| Mode | original | adaptive |
| Local threshold | Yen + 0.015 | Otsu |
| Surface margin | 3 voxels | 0.6 mm |
| Smallest bubble | 2 voxels | 0.5 mm³ |
| Edge filter | off | on |

## Results

### Normal data
52 bubbles found, the largest has 916 mm³, the smallest 0.26 mm³, total void volume is 1002 mm³.

### How I checked it (validation)
I used two separate test scans, **A** and **B**. For each of them I marked the bubbles by hand (in red) on **20 slices**, some of them without any bubble.
Then I compared my marks with what the program found.

| | Scan A | Scan B |
|---|---|---|
| Pixel precision | 0.970 | 1.000 |
| Pixel recall | 0.418 | 0.067 |
| IoU | 0.413 | 0.067 |
| Dice | 0.584 | 0.125 |
| Marked bubbles found | 28 of 31 | 10 of 12 |
| Detections that were marked | 30 of 31 | 12 of 12 |

What does this mean?

- **Precision** is very high: what the program marks as bubble is almost always a bubble I also marked.
- **Recall on pixels** is low: the program covers only a part of each marked area.
- **Bubble level** is much better: most of the marked bubbles are found (28 of 31 and 10 of 12).
- In the empty slices the program found nothing, so no false alarms there.

Edge filter: on scan A it removed 50 clusters and kept 35 bubbles, on scan B it removed 67 and kept 7.

### Threshold sweep (scan A)
I also tried every grey level from 0 to 255 as threshold:

- precision and recall are equal at grey 152 (precision 0.08, recall 0.10), the threshold used is grey 168
- AUC of the ROC curve is 0.808
- the full algorithm reaches precision 0.970 at recall 0.418, so the cleaning steps help a lot compared to a threshold alone

## Limitations

- small test set: two scans with 20 marked slices each
- pixel recall is low (0.418 on A, 0.067 on B), even though most bubbles are found
- on scan B the edge filter removed 67 of 74 clusters and kept only 7 bubbles

## How to run it

1. Open the notebook in Google Colab and choose a **T4 GPU** runtime.
2. Mount Google Drive. The data is not in this repository because it is too large.
3. Put the scan zip files (projections) into the project folder on Drive and set the names in the notebook.
4. Run the cells from top to bottom. Section 5 is the detection on the normal data, section 8 is the validation with the test scans, section 9 is the threshold sweep.

Python libraries used: `astra-toolbox`, `numpy`, `scipy`, `scikit-image`, `scikit-learn`, `pandas`, `matplotlib`, `plotly`, `Pillow`.

## Author

Prembabu Chandrababu, University of Koblenz
