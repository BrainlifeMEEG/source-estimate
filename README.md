# Source Estimate

[![Run on Brainlife.io](https://img.shields.io/badge/Brainlife-bl.app.906-blue.svg)](https://doi.org/10.25663/brainlife.app.906)

## Description

Applies a pre-computed inverse operator to evoked MEG/EEG data to produce source estimates
(STCs), using `mne.minimum_norm.apply_inverse`. The evoked input may contain multiple conditions
(e.g. produced by an evoked app with `by_event_type=True`); in that case the app loops over every
condition and saves one STC per condition, named after the `comment` field of each evoked object
(e.g. `"13"`, `"Dir-Ext"`). If no evoked file is given, epochs are averaged to a single grand-mean
evoked first. STCs can optionally be morphed to the fsaverage template for group comparison.

The app generates:
- one source estimate (STC) per condition
- the same STC(s) morphed to fsaverage space, if `morph_to_fsaverage` is set
- QC figures (evoked butterfly, source time course, brain surface filmstrip)
- an HTML report with all condition STCs
- a `product.json` with status/diagnostic messages and figure thumbnails

## Inputs

- **`inverse`** (`neuro/meeg/mne/forward-inverse`): inverse operator (`inv.fif`) from
  app-inverse-v2 (required)
- **`evoked`** (`neuro/meeg/mne/evoked`): evoked data — single or multi-condition (one of
  `evoked`/`epo` required; `evoked` takes priority if both are given)
- **`epo`** (`neuro/meeg/mne/epochs`): epoched data, averaged to a single grand-mean evoked if no
  `evoked` file is given (also accepted under the legacy config key `epochs` as a fallback alias
  for the same path) (one of `evoked`/`epo` required)
- **`freesurfer`** (`neuro/freesurfer`): FreeSurfer subject directory, used for brain-surface
  plots and for morphing to fsaverage (also accepted under the config key `output` as a fallback
  alias for the same path) (optional — falls back to the built-in fsaverage template for plots if
  absent)

## Outputs

- `out_dir/source_estimate-lh.stc` / `source_estimate-rh.stc` (`neuro/meeg/stc`): source estimate
  (single-condition input)
- `out_dir/source_estimate_<label>-lh.stc` / `-rh.stc`: one STC pair per condition
  (multi-condition evoked input)
- `out_dir/source_estimate_fsaverage[-<label>]-lh.stc` / `-rh.stc`: STC(s) morphed to fsaverage
  space (only if `morph_to_fsaverage=true` and the FreeSurfer subject isn't already `fsaverage`)
- `out_figs/evoked_butterfly.png`: evoked butterfly plot (grand average across conditions)
- `out_figs/source_time_course_<label>.png` (per condition) and
  `out_figs/source_time_course_grandavg.png` (grand average, only when there are multiple
  conditions): mean source amplitude over time, left/right hemisphere side by side
- `out_figs/brain_lh.png` / `out_figs/brain_rh.png`: brain surface filmstrip at several timepoints
  around the peak, grand-average STC
- `out_report/report.html` (`report/html`): interactive HTML report with the evoked butterfly,
  source time course figures, and per-condition STC viewers
- `product.json`: Brainlife.io metadata — status/diagnostic messages and figure thumbnails

## Configuration Parameters

| key | type | default | description |
|---|---|---|---|
| `method` | string (enum) | `dSPM` | Inverse method passed to `mne.minimum_norm.apply_inverse`: `dSPM`, `sLORETA`, `MNE`, or `eLORETA`. |
| `snr` | number | `3.0` | Assumed signal-to-noise ratio; sets the regularization parameter λ² = 1/SNR². Use `3.0` for averaged evoked data, `1.0`–`2.0` for single trials/noisy data. |
| `pick_ori` | string | `""` (free orientation) | Source orientation constraint: empty/`None` for free orientation, `normal` for surface-normal only (one dipole per vertex instead of three). |
| `tmin` | number \| null | `null` | If set, crops each evoked to `[tmin, tmax]` seconds before applying the inverse. |
| `tmax` | number \| null | `null` | If set, crops each evoked to `[tmin, tmax]` seconds before applying the inverse. |
| `morph_to_fsaverage` | boolean | `false` | If `true`, morphs each STC to the fsaverage template (skipped automatically if the FreeSurfer subject is already `fsaverage`). |
| `subject` | string | `""` (auto) | FreeSurfer subject name; auto-detected from the `freesurfer`/`output` input directory if not set, or `fsaverage` if no FreeSurfer input is given. |
| `subjects_dir` | string | `""` (auto) | FreeSurfer `SUBJECTS_DIR`; auto-detected from the `freesurfer`/`output` input directory if not set. |

## Usage

### Running on Brainlife.io

1. Select the inverse operator (`inverse`) and either `evoked` or `epo` sensor data as inputs
   (`evoked` takes priority if both are given).
2. Optionally select a FreeSurfer subject directory (`freesurfer`) for brain-surface plots and
   morphing.
3. Set `method`, `snr`, `pick_ori`, `tmin`/`tmax`, and `morph_to_fsaverage` if the defaults are
   not appropriate.
4. Submit the process.
5. Review the QC figures and the interactive HTML report in the output viewer.

### Local Testing

```bash
# Edit config.json to point "inverse" and "evoked" (or "epo") at real files, and "freesurfer"
# at a FreeSurfer subject directory, then:
python main.py
```

## Technical Details

The app:

1. Loads the inverse operator from app-inverse-v2.
2. Loads sensor data — `evoked` (prioritized) or `epo` (averaged to one grand-mean evoked).
3. Adds an EEG average reference projection (if EEG channels are present), applies baseline
   correction (always `(None, 0)`), and crops to `tmin`/`tmax` if set.
4. For each condition, runs `apply_inverse(evoked, inverse_operator, λ², method, pick_ori)` and
   saves one STC.
5. If `morph_to_fsaverage` is set, computes the morph once (from the first condition) and reuses
   it for all other conditions; skipped automatically if the FreeSurfer subject is `fsaverage`.
6. Generates QC figures from the grand-average STC (evoked butterfly, source time course, brain
   surface filmstrip).
7. Saves an HTML report with all condition STCs.

Notes:
- Baseline correction is always applied (`(None, 0)`) — epochs should cover at least 200 ms
  pre-stimulus.
- Brain surface plots require a FreeSurfer directory via the `freesurfer`/`output` input; they
  fall back to the built-in fsaverage template if absent.
- STC files are saved with MNE's default naming convention (`-lh.stc` / `-rh.stc` for surface
  STCs).
- Container image: `docker://brainlifemeeg/mne-freesurfer:1.12.1-7.4.1`.

## Pipeline Position

This app is the last step of the source reconstruction pipeline:

```
app-coreg-v2 ──→ app-bem-v2 ──→ app-forward-v2 ──→ app-inverse-v2 ──→ [app-source-estimate-v2]
                                                          ↑
                                             app-noise-covariance-v2
```

## Authors

- Guiomar Niso (https://github.com/guiomar)
- Antonio Caulín (https://github.com/AntonioCauAt)
- Maximilien Chaumon (https://github.com/dnacombo)
- obVdo (https://github.com/obVdo)

Based on: https://github.com/guiomar/app-source-estimate

## Citations

- Hayashi, S., Caron, B.A., Heinsfeld, A.S. et al. brainlife.io: a decentralized and open-source cloud platform to support neuroscience research. Nat Methods 21, 809–813 (2024). https://doi.org/10.1038/s41592-024-02237-2
- Gramfort, A., Luessi, M., Larson, E., et al. MEG and EEG data analysis with MNE-Python. Front. Neurosci. 7, 267 (2013). https://doi.org/10.3389/fnins.2013.00267

## Funding Acknowledgement

brainlife.io is publicly funded and for the sustainability of the project it is helpful to acknowledge the use of the platform. We kindly ask that you acknowledge the funding below in your code and publications.

[![NSF-BCS-1734853](https://img.shields.io/badge/NSF_BCS-1734853-blue.svg)](https://nsf.gov/awardsearch/showAward?AWD_ID=1734853)
[![NSF-BCS-1636893](https://img.shields.io/badge/NSF_BCS-1636893-blue.svg)](https://nsf.gov/awardsearch/showAward?AWD_ID=1636893)
[![NSF-ACI-1916518](https://img.shields.io/badge/NSF_ACI-1916518-blue.svg)](https://nsf.gov/awardsearch/showAward?AWD_ID=1916518)
[![NSF-IIS-1912270](https://img.shields.io/badge/NSF_IIS-1912270-blue.svg)](https://nsf.gov/awardsearch/showAward?AWD_ID=1912270)
[![NIH-NIBIB-R01EB029272](https://img.shields.io/badge/NIH_NIBIB-R01EB029272-green.svg)](https://grantome.com/grant/NIH/R01-EB029272-01)
[![NIH-NIBIB-R01EB030896](https://img.shields.io/badge/NIH_NIBIB-R01EB030896-green.svg)](https://grantome.com/grant/NIH/R01-EB030896-01)

## License

Copyright (c) 2026 MEEG Brainlife team. Licensed under AGPL-3.0, see [license.txt](license.txt).
