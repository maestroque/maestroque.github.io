+++
title = 'GSoC Physiopy'
date = 2024-09-03T19:33:41+03:00
draft = false
toc = true
+++

{{< lead >}}
*"Semi-Automated Workflows for Physiological Signals"*
{{< /lead >}}

During the summer of 2024, I had the chance to become a Google Summer of Code student, contributing in the [Physiopy](https://physiopy.github.io/) community, under the [INCF](https://www.incf.org/) organization. In this page, which is also serving as my Work Product Submission for finalizing my GSoC participation, the project will be presented, how it progressed during the summer and what future directions it could take. 
Emphasis is going to be given to the development breakdown, the rationale of all the contributions and the main ideas for future development.

## 1. Project Overview

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

Some of my first contributions to physiopy included infrastructure upgrades, notably pre-commit hook configurations for style checks and the configuration of the [`loguru`](https://loguru.readthedocs.io/en/stable/#readme) package as the primary logger in `peakdet`.

The main benefit of `loguru` that was exploited is that other than the fully customizable stylized logs, the module can also catch exceptions and log them in the same manner, improving uniformity when used in a CLI for example. Furthermore, `loguru` can provide insightful diagnostics if exceptions happen, making the overall user experience better. A set of utility functions was implemented to wrap around the `loguru` logger, in order to make the developer use easier as well.

### 3.2 Optimizing peakdet/phys2denoise interoperability

The `peakdet` and `phys2denoise` packages, have been both very critical packages of the physiopy suite, with their operation being frequently sequential in multiple fMRI denoising applications. 
Meaning, that in many common data processing routines, the physiological data first needed to pass through `peakdet`, for filtering and peak/trough detection, and then through `phys2denoise` for the computation of the several denoising metrics.
However, by nature up to this point, `peakdet` was built in an "object-oriented" fashion, being structured around the `Physio` object, while `phys2denoise` was predominantly "function-oriented".
The main idea was to incorporate the `Physio` object into both packages, to exploit some of its powerful features, such as operation history management and history "replaying" and easy interfacing with multiple input physiological data types.
Therefore, the main challenge was to make this migration, incorporating these features to `phys2denoise` and adding more to cater to the additional needs, while continuing to support its "function-only" use.

At the beggining of my GSoC project, prior to the upgrade, a new package was issued for this scope: `physutils`.
This module would host the common physiopy suite tools, to be used throughout the physiopy packages, with one of them being the `Physio` object.
Another honorable mention is that while `phys2denoise` has now fully migrated to use `physutils`, the community decided to proceed with such changes in a fork of the `peakdet` package, re-named as [`prep4phys`](https://github.com/physiopy/prep4phys), which will serve as the active package moving on.

Within `physutils` the `Physio` object was upgraded to contain more information about the contained physiological signal, aiming for most information used throughout the processing packages `prep4phys` and `phys2denoise` to be contained within it.
Another major addition, used in conjunction with the previous one, was the ability to generate `Physio` objects from [BIDS structures](https://bids-specification.readthedocs.io/en/stable/) using the `physutils.io.load_from_bids` function, which are most commonly used in research. Another advantage of this feature is the out-of-the-box compatibility with the widely used [`phys2bids`](https://phys2bids.readthedocs.io/en/latest/) module, developed by the Physiopy community, which transforms many data types commonly found in physiological recording setups (e.g. BIOPAC, ADInstruments) into the more standardized BIDS format. This was aided by the use of the [`pybids`](https://bids-standard.github.io/pybids/) module, and allowed for a BIDS structure to be transformed into an array of Physio objects, extracting all information relevant to the physiological signals.

This was a well needed feature, since BIDS support would greatly motivate the users to make use of `phys2denoise` and `prep4phys`, since this `physutils` function would be used as an entry point to both. Thus, a step was made towards a unified `physiopy` suite of tools, decreasing user involvement in custom scripts using the libraries, as all would communicate through standardized objects. Below a flowgraph can be seen describing the interfaces between the packages.

{{< mermaid >}}
graph LR;
Z[Recording Data]-->A[phys2bids]-->|BIDS Data| B[physutils.io.load_from_bids];
classDef redArrow stroke:#ff0000,stroke-width:2px;
B-- Physio --> DataProcessing
subgraph DataProcessing[Physiological Data Processing]
direction TB;
    C[phys2denoise]
    D[prep4phys/peakdet]
    E[physioqc]
    C<-.->|Physio|D
    D<-.->|Physio|E
end
style A fill:#c197db,stroke:#333
{{< /mermaid >}}

---

Moving on, regarding the integration of the `Physio` object into the `phys2denoise` module, a careful approach had to be considered, in order to maintain the ability of the package to be used both with and without `Physio`. In order to do this a wrapper was created to handle function I/O differently in case the input was a `Physio` object or a `numpy.array`-like object for example.
- In case a `Physio` is used, all the necessary information is already embedded for the metric computation (sampling frequency, peak/trough data) and the metric is saved in the `Physio.computed_metrics` dictionary (with members of type `Metric` containing the metric numerical data along with metadata). Then all the computed metrics can be referenced by name as `physio.computed_metrics["<Metric_Name>"]`. Note that all the computations along with the arguments passed are saved in the `Physio` object's history and can be reproduced given the `history.json` file

### 3.3 Workflow and CLI for phys2denoise

The next step, would be to create a workflow model for one of the packages, `phys2denoise`.
For its structuring, the `pydra` python package was used, a dataflow engine commonly used in neuroimaging, that allows for distributed execution of data processing routines, while also ensuring the reproducibility of these routines. 
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

Lastly, this flowgraph is templated so that it can be built given the parameters passed to the CLI by the user. A current version of a CLI run can be seen as captured below

<script src="https://asciinema.org/a/PHi5AIIpxSH9jsq1AvKELom09.js" id="asciicast-PHi5AIIpxSH9jsq1AvKELom09" async="true"></script>


## 4. Future Directions
Furthermore, I will list some of my thoughts for improvements and optimizations on the packages which were out of the scope of my summer in GSoC.

**Infrastructure**
- One change that would improve uniformity throughout the packages (and respective CLIs in the future) would be to migrate the `loguru` utility functions from `peakdet` to `physutils` in order for them to be used in all modules. 
- Furthermore, the issue observed regarding the inclusion of `loguru` logger calls inside `pydra` tasks, shall be resolved. As of now, it is not certain if the issue stems from `pydra`/`loguru` or the Physiopy packages yet.

**Physio Object**
- One of the directions towards further upgrading the `Physio` object, would be the integration of a `MultimodalPhysio` class, that would encapsulate all the physiological signals, per recording, into one object. Furthermore, it could also contain an `MRIConfig` object, hosting all the common MRI parameters among the physiological data (TR, Slice Timings, Number of Scans, etc.) which are also used in metrics computations. The inclusion of such a class would be greatly beneficial as it would be able to contain all the information from a BIDS structure (per subject/session/recording). Lastly, given the trigger channel commonly included in BIDS files, several `MRIConfig` parameters could also be computed using simple signal processing. [[Issue]](https://github.com/physiopy/physutils/issues/5)

**CLI Development**
- One of the main advantages of `pydra` as mentioned above is the ability to parallelize tasks. In the case of `phys2denoise` this wouldn't be the case at a first glance due to its linearity. However, an upgrade of the CLI could be considered to allow for the processing of multi-subject multi-session physiological data concurrently.

---
Lastly, I'd like to express my gratitude towards my mentors Stefano Moia ([@smoia](https://github.com/smoia)), Mary-Eve Picard ([@me-pic](https://github.com/me-pic)) and Mary Miedema ([@m-miedema](https://github.com/m-miedema)) and the physiopy community for providing me with this great experience. 