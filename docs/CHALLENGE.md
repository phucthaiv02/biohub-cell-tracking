# Overview
Your goal is to develop algorithms to detect, track and link cells across time in 3D microscopy data, including accurate identification of cell divisions and lineage reconstruction. You will work with real microscopy datasets to build robust methods that can handle dense cell populations, noise and complex biological structures.

Your work will eliminate a massive manual bottleneck in biological research and help scientists quantify the building blocks of life.

# Description
Tracking cells across time in 3D microscopy is a fundamental challenge in biological research. Scientists rely on time-lapse 3D imaging to study how cells grow, interact, and evolve, but analyzing this data remains a massive bottleneck. Currently, researchers spend countless hours manually tracking cells—especially in complex datasets where thousands of visually similar cells move, deform, and divide.

While automated tools exist, they often fail under real-world conditions. High cell density, imaging noise, and irregular cell shapes cause critical errors in lineage reconstruction, limiting the scalability of these studies.

This competition provides a shared benchmark to solve this problem. Your task is to detect cells, associate them across frames, and identify division events to reconstruct accurate cell lineages. By developing robust, generalizable algorithms for 3D+time cell tracking, you will help eliminate manual effort, improve scientific reproducibility, and accelerate new discoveries in developmental biology, immunology, and disease research.

# Data
## Dataset Description
Each sample in this dataset is a short 3D+time video of fluorescently labeled zebrafish embryo cells, stored as a Zarr v3 volume. Your task is to detect cells in each timepoint and link them across time, producing a tracking graph of nodes (cell detections) and edges (temporal links between cells).

## Data Format
Image volumes are stored as `.zarr` directories. Each contains a single array at path `0/` with shape `(T, Z, Y, X)` — typically `(100, 64, 256, 256)` in `uint16` format. Chunks are one timepoint each : `(1, 64, 256, 256)`, compressed with blosc/zstd. The chunk for timepoint `t` is located at `0/c/{t}/0/0/0`. Array metadata `(shape, dtype, codecs)` is in `0/zarr.json`.

The physical voxel scale is z=1.625, y=0.40625, x=0.40625 µm/voxel.

## Ground Truth (Training Only)
Ground-truth annotations are provided as `.geff` directories (a graph exchange format also built on Zarr v3). Each `.geff` contains:
 - `nodes/ids` — node ID array
 - `nodes/props/{t,z,y,x}/values` — integer centroid coordinates per node `(in voxels)`
 - `edges/ids` — edge array of shape `(N, 2)` with columns `(source_id, target_id)`

Annotations are sparse — not every cell in every frame is labeled. The `estimated_number_of_nodes` field in the `.geff` metadata (`zarr.json`) provides an estimate of the true total cell count per sample.

All arrays within `.geff` use zstd compression.

## Embryo Identity
Folder names follow the pattern `{embryo_id}_{field_of_view}` (e.g., `44b6_0049_0438_1330_1273`). The first segment identifies which embryo the sample comes from. Multiple samples may share the same embryo. __Train and test sets are embryo-disjoint__ — no embryo appears in both.

## Files
- __train/__ - Training samples. Each sample has a paired `.zarr` (image volume) and `.geff` (ground-truth tracking graph).
- __test/__ - Example test samples (copies from train). Contains `.zarr` image volumes only — no ground truth is provided. When a notebook is submitted for rerun, a new hidden test set is swapped in. The size of the hidden test set is approximately the same size as the training dataset.
- __sample_submission.csv__ - A valid submission file demonstrating the correct format.


# Evaluation
Submissions are evaluated using a combined tracking metric that measures both edge accuracy (how well cells are linked across time) and division detection (how well cell mitosis events are identified). The combined score is:

$$\text{score} = \text{adjusted\_edge\_jaccard} + 0.1 \times \text{division\_jaccard}$$


Edge Jaccard: Predicted nodes are matched to ground-truth nodes per timepoint via optimal bipartite assignment on scaled centroid distance (max 7.0 µm, physical scale z=1.625, y=x=0.40625 µm/voxel). A predicted edge is a true positive when both endpoints match ground-truth nodes connected by a ground-truth edge. The edge Jaccard is TP / (TP + FP + FN), adjusted by a penalty on over-predicting the total number of nodes.

Division Jaccard: A cell division is a node with two or more outgoing edges. For each ground-truth division, the predicted graph is checked for a connected component that covers the pre-split stage and touches both daughter lineages. Division TP/FP/FN are computed and combined into a micro-averaged Jaccard.

Per-sample adjusted edge Jaccards are weight-averaged by (TP + FP + FN); division Jaccards are micro-averaged across all samples.

Additional details about the metric can be found here. Note that cells are sparsely labeled in the ground truth, which the metric accounts for. Due to the nature of the metric, it is possible for scores to exceed 1.0.

