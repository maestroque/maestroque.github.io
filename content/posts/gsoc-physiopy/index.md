+++
title = 'GSoC Physiopy'
date = 2024-09-03T19:33:41+03:00
draft = false
toc = true
+++

During the summer of 2024, I had the chance to become a Google Summer of Code student, contributing in the [Physiopy](https://physiopy.github.io/) community, under the [INCF](https://www.incf.org/) organization. In this page, which is also serving as my Work Product Submission for finalizing my GSoC participation, the project will be presented, how it progressed during the summer and what future directions it could take. 
Emphasis is going to be given to the development breakdown, the rationale of all the contributions and the main ideas for future development.

## 1. Project Overview
{{< lead >}}
*"Semi-Automated Workflows for Physiological Signals"*
{{< /lead >}}

In general, the project entailed the upgrade and restructuring of existing Physiopy packages, `peakdet` and `phys2denoise`, in order to be used more harmoniously with each other. 
Following the upgrade, the development of semi-automated workflows was planned, along with the accompanying CLI. 

## 2. Pull Request Synopsis
### `peakdet`
- https://github.com/physiopy/peakdet/pull/67
- https://github.com/physiopy/peakdet/pull/62
### `phys2denoise`
- https://github.com/physiopy/phys2denoise/pull/54
- https://github.com/physiopy/phys2denoise/pull/56
- https://github.com/physiopy/phys2denoise/pull/57
### `physutils`
- https://github.com/physiopy/physutils/pull/2
- https://github.com/physiopy/physutils/pull/4
- https://github.com/physiopy/physutils/pull/7

## 3. Development Overview

### 3.1 Infrastructure 

### 3.2 Optimizing peakdet/phys2denoise interoperability

The `peakdet` and `phys2denoise` packages, have been both very critical packages of the physiopy suite, with their operation being frequently sequential in multiple fMRI denoising applications. 
Meaning, that in many common data processing routines, the physiological data first needed to pass through `peakdet`, for filtering and peak/trough detection, and then through `phys2denoise` for the computation of the several denoising metrics.
However, by nature up to this point, `peakdet` was built in an "object-oriented" fashion, being structured around the `Physio` object, while `phys2denoise` was predominantly "function-oriented".
The main idea was to incorporate the `Physio` object into both packages, to exploit some of its powerful features, such as operation history management and history "replaying" and easy interfacing with multiple input physiological data types.
Therefore, the main challenge was to make this migration, incorporating these features to `phys2denoise` and adding more to cater to the additional needs, while continuing to support its "function-only" use.

Initially, prior to the upgrade, a new package was issued for this scope: `physutils`.
This module would host the common physiopy suite tools, to be used throughout the physiopy packages, with one of them being the `Physio` object.
Another honorable mention is that while `phys2denoise` has now fully migrated to use `physutils`, the community decided to proceed with such changes in a fork of the `peakdet` package, re-named as [`prep4phys`](https://github.com/physiopy/prep4phys), which will serve as the active package moving on.



### 3.3 Workflow and CLI for phys2denoise

For the structuring of the `phys2denoise` workflow, the `pydra` python package was used, a dataflow engine commonly used in neuroimaging. 
Since `phys2denoise`'s use is pretty straightforward, the workflow turned out to be linear, with all component `pydra` tasks, being used sequentially. The following three tasks were defined for the workflow:

{{< mermaid >}}
graph LR;
A[transform_to_physio]-->B[compute_metrics];
B-->C[export_metrics]
style A fill:#c197db,stroke:#333
{{< /mermaid >}}

The `transform_to_physio` task is part of `physutils`, as it serves as an interface task to be used in other physiopy workflows (prep4phys, physioqc, etc.), to transform multiple input formats into a `Physio` object, which is the primary I/O object throughout the tasks and functions. 
Its primary function is to take as an input either a "physio" file of any type that contains 1D data, including pickled `Physio` object files, or a BIDS data structure. Within the workflow the input data type can also be deduced automatically. To accomplish that it uses `physutils.load_physio` and `physutils.load_from_bids`.

```python
def transform_to_physio(
    input_file: str, mode="physio", fs=None, 
    bids_parameters=dict(), bids_channel=None 
) -> Physio:
```

The `compute_metrics` task is used to check if all the provided metrics can be computed (i.e. all the required parameters and signal properties are calculated or given), computes them, and then stores them in the `Physio` object for further use.
It utilizes all the functions from `phys2denoise.metrics` for the actual computations, and a `select_input_args` function to verify that the metric can be computed

```python
def compute_metrics(
    phys: Physio, metrics: Union[list, str], 
    args: dict
) -> Physio:
```

Finally, the `export_metrics` task exports the specified metrics, which could be a subset of the computed ones. 
If no export metric parameters are specified, the CLI defaults to exporting all the metrics. 
Exporting includes generating 1D files that contain the metric. If a metric contains lags, then a file will be exported for each lag value, and if a metric is convolved it will export both a file of the convolved signal and a file of the raw signal. Furthermore, all metrics are both exported in the original sample rate and resampled at the fMRI scan TR.
```python
def export_metrics(
    phys: Physio, metrics: Union[list, str], 
    outdir: str, tr: Union[int, float]
) -> int:
```

A current version of a CLI run can be seen as captured below

<script src="https://asciinema.org/a/PHi5AIIpxSH9jsq1AvKELom09.js" id="asciicast-PHi5AIIpxSH9jsq1AvKELom09" async="true"></script>

## 4. Future Directions

### 4.1 Infrastructure

### 4.2 CLI for peakdet

### 4.3 CLI for physiopy suite

---
Lastly, I'd like to express my gratitude towards my mentors Stefano Moia ([@smoia](https://github.com/smoia)), Mary-Eve Picard ([@me-pic](https://github.com/me-pic)) and Mary Miedema ([@m-miedema](https://github.com/m-miedema)) and the physiopy community for providing me with this great experience. 