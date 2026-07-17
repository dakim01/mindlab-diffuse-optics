# MINDLAB VOT Case Study — HDOS Diffuse Optics Data Analysis

A single-subject case study analyzing a **Vascular Occlusion Test (VOT)** measurement acquired with the **HDOS** (Hybrid Diffuse Optical System) device, combining **Time-Domain Near-Infrared Spectroscopy (TD-NIRS)** and **Diffuse Correlation Spectroscopy (DCS)**. This notebook was developed during the hands-on lab sessions of the **MINDLAB Frontiers Research School** (ICFO, Barcelona, July 2026).

## Overview

This repository documents the full processing pipeline used to go from **raw optical/sensor data** to a **quantitative table of vascular-occlusion physiology parameters** for one subject, using a proprietary lab-developed device (HDOS) and its companion Python analysis toolkit.

The measurement combines two complementary diffuse-optics technologies applied simultaneously to a forearm muscle during a standard vascular occlusion (blood-pressure-cuff) protocol:

- **TD-NIRS** → absolute **tissue oxygen saturation (StO₂)**, via time-of-flight fitting of picosecond laser pulses and Beer–Lambert chromophore unmixing.
- **DCS** → **microvascular blood flow (Blood Flow Index, BFI)**, via analysis of laser speckle intensity autocorrelation decay.

Together, these allow the extraction of both **oxygen consumption (metabolic)** and **vascular reactivity (endothelial function)** parameters from a single, non-invasive bedside measurement.

## Background: What the VOT Protocol Measures

The Vascular Occlusion Test is a standard technique (used clinically in ICU/critical-care settings) in which a pneumatic cuff is inflated on the upper arm to well above systolic pressure, fully occluding arterial blood flow to the forearm for several minutes, then rapidly released.

- **During occlusion**: no new oxygenated blood arrives, but the muscle continues consuming the oxygen already present → tissue oxygenation (StO₂) **declines**, and the *rate* of decline reflects local **oxygen consumption / metabolic rate**.
- **Upon release**: blood rushes back into vasculature that has reflexively dilated during ischemia, producing a transient **hyperemic overshoot** in both flow and oxygenation before returning to baseline. The *magnitude and speed* of this overshoot reflects **endothelial function / microvascular reactivity**.

Because flow (DCS) and oxygenation (TD-NIRS) are acquired **simultaneously and co-located**, this setup allows oxygen delivery to be disentangled from oxygen consumption — something neither technology can do alone.

## Device & Data Source

- **Device:** HDOS (custom-built hybrid TD-NIRS + DCS platform), data format **v2**.
- **Session:** ICFO MINDLAB hands-on lab, vascular occlusion / endothelial function test (`VASC_ENDO`).
- **Raw data folder structure expected:**
  ```
  ./tests/HDOS-icfo_Mindlab_4_VASC_ENDO_<timestamp>/
      measurement_info/        # IRF and instrument calibration files
      ...                       # raw + fitted binary/log data files (sensor, DCS, TRS, PPG)
  ```
- Subject data is anonymized (numeric subject ID only); no personally identifiable information is included in the processed outputs.

> **Note:** Raw instrument data files are lab-internal (proprietary HDOS binary/log formats) and are **not included in this repository**. This repo documents the analysis code and pipeline; raw data should be supplied locally in a `./tests/` folder matching the structure above.

## Repository Structure (expected)

```
.
├── MINDLAB_VOT_v2.ipynb        # Main analysis notebook (this case study)
├── myFunctions_TRS.py          # TD-NIRS fitting utilities (external/lab-provided)
├── myFunctions_DCS.py          # DCS fitting utilities (external/lab-provided)
├── HDOS2_DATAREAD_v2.py        # HDOS v2 data-format readers (external/lab-provided)
├── tests/
│   └── HDOS-icfo_Mindlab_4_VASC_ENDO_<timestamp>/   # raw measurement folder
└── README.md
```

## Dependencies

Standard scientific Python stack:

```
numpy
pandas
scipy
matplotlib
tqdm
ipympl        # for %matplotlib widget interactive plots
```

Plus lab-internal modules (not distributed publicly — obtained from the MINDLAB/HDOS lab codebase):

- `myFunctions_TRS.py` — `getFitResult2_TRS`, `getFitResult1_TRS`, `Reflectance_Contini`
- `myFunctions_DCS.py` — `getFitResult2_DCS`
- `HDOS2_DATAREAD_v2.py` — folder-read functions for sensor, DCS (raw + fitted), TD-NIRS/TRS (raw + fitted), and PPG data streams

## Notebook Workflow

The notebook is organized into two main parts: **(1) reading all data streams**, and **(2) analyzing, plotting, and quantifying the VOT response**.

### 1. Data Reading

| Section | What it loads |
|---|---|
| Sensor data | Cuff pressure, event markers (inflate/deflate/general/quality-phase), pulse oximeter SpO₂ and heart rate, accelerometer, touch/load sensor, laser status, battery/system diagnostics |
| Raw DCS data | Per-channel (4 fibers) photon count rate and raw intensity autocorrelation curves |
| Fitted DCS data | Fitted Blood Flow Index (BFI), correlation amplitude (β), decay time constants, fit quality (R²) |
| Raw TD-NIRS data | Distribution-of-Time-of-Flight (DTOF) curves and photon count rates at both wavelengths (685 nm, 828 nm) |
| Fitted TD-NIRS data | μₐ and μₛ′ at 685 nm and 828 nm, interpolated μₐ/μₛ′ at the DCS wavelength (785 nm), and derived chromophore concentrations: HbO, HbR, HbT, StO₂ |
| PPG data | Photoplethysmography waveform (not used further in this notebook) |

All timestamps are re-referenced to a common **t = 0** anchored to the start of the "quality phase" marker (the automatic data-quality check performed before the real acquisition begins — see Instrumentation notes below), so that sensor, DCS, and TRS streams — despite being sampled at different native rates — can be compared on a shared time axis.

### 2. Analysis, Plotting & Parameter Extraction

**a) Sensor-level sanity plots**
- Cuff pressure vs. time (confirms clean inflate → hold → deflate profile, with event markers overlaid)
- SpO₂ vs. time
- Heart rate vs. time

**b) Tissue oxygenation (StO₂) analysis**
- Plot StO₂ vs. time with event markers overlaid.
- Manually define, by visual inspection of the trace:
  - **Baseline window** (pre-occlusion, stable resting StO₂)
  - **Deoxygenation window** (first minute of occlusion — the region where the StO₂ decline is most reliably linear)
  - **Reoxygenation window** (short window immediately after cuff release, capturing the fast recovery rise)
  - **Recovery baseline window** (after the hyperemic overshoot has resolved, confirming return to steady state)
  - **AUC window** (from end of reoxygenation phase through resolution of the overshoot, used to integrate the hyperemic response)
- Compute:
  - Baseline StO₂ (mean, %)
  - **Deoxygenation slope** (linear fit slope over the occlusion window, in %/min)
  - **Reoxygenation slope** (linear fit slope over the release window, in %/s)
  - Recovery baseline StO₂ (mean, %)
  - **Area under the curve (AUC)** of the StO₂ hyperemic overshoot above the recovery baseline (trapezoidal integration, in %·min)

**c) Blood flow (BFI) analysis**
- Same structure as the StO₂ analysis, applied to the DCS-derived BFI trace:
  - Baseline BFI (pre-occlusion, cm²/s)
  - Recovery baseline BFI (post-overshoot, cm²/s)
  - **AUC** of the BFI hyperemic overshoot above the recovery baseline (trapezoidal integration, in cm²/s·min)
- Plot y-axis is capped using the **99th percentile** of the BFI trace (rather than the raw max) to prevent a single motion-artifact spike from compressing the rest of the trace visually.

**d) Data export**
- Sensor data (time, cuff pressure, heart rate, SpO₂, markers) exported to a CSV file named after the source data folder.
- Subject metadata entered manually: subject ID, sex, and arm fat-layer thickness (cm) — the latter recorded because overlying fat/skin thickness affects the optical signal (see notes on anthropometric confounds below).
- All extracted VOT parameters compiled into a single-row results table and exported as `RESULTS_VOT_<data_folder_name>.csv`.

## Mathematical & Algorithmic Details

This section documents the underlying physics/statistics behind each processing step, matching the functions called in the notebook (`myFunctions_TRS`, `myFunctions_DCS`, and the manual analysis cells).

### 1. TD-NIRS: Photon Diffusion Model & DTOF Fitting

**Physical model.** In the diffusive (highly-scattering) regime, light transport through tissue is described by the **photon diffusion equation**:

$$\frac{1}{v}\frac{\partial \Phi(\mathbf{r},t)}{\partial t} = D\nabla^2 \Phi(\mathbf{r},t) - \mu_a \, \Phi(\mathbf{r},t) + S(\mathbf{r},t)$$

where $\Phi$ is the photon fluence rate, $v$ is the speed of light in the medium, $S$ is the source term (the injected laser pulse), and

$$D = \frac{1}{3(\mu_a + \mu_s')}$$

is the **photon diffusion coefficient**, with $\mu_a$ the absorption coefficient and $\mu_s'$ the reduced scattering coefficient — the two parameters the fit ultimately recovers.

**Analytical solution (semi-infinite reflectance geometry).** For a semi-infinite homogeneous medium with an extrapolated-boundary condition (the standard solid-tissue approximation used at a source–detector separation $\rho$), the time-resolved reflectance $R(\rho,t)$ has a closed-form solution — this is the model implemented in `Reflectance_Contini` (Contini, Martelli & Zaccanti, *Applied Optics*, 1997), of the general form:

$$R(\rho,t) \propto t^{-5/2}\exp(-\mu_a v t)\left[z_0\exp\!\left(-\frac{z_0^2+\rho^2}{4Dvt}\right) - z_b\exp\!\left(-\frac{z_b^2+\rho^2}{4Dvt}\right)\right]$$

where $z_0 = 1/\mu_s'$ is the depth of the effective isotropic source and $z_b$ is the (refractive-index-mismatch-dependent) extrapolated boundary distance. This is the theoretical DTOF (Distribution of Time-of-Flight) curve shape.

**Instrument response.** Because the real system has its own temporal blurring (fiber/lens reflections, detector timing jitter — the measured IRF), the theoretical curve is **convolved with the measured IRF** before comparison to data:

$$R_{\text{model}}(t) = \big(R(\rho,t) \ast \mathrm{IRF}(t)\big)$$

**Fitting procedure.**
1. Background-subtract and peak-normalize both the measured DTOF and the IRF (§ *TD-NIRS Fitting Pipeline* discussed in the companion lab notes).
2. Restrict the fit range to roughly 90% of the peak (falling edge) through 2–3 decades into the tail, excluding the early-photon rising edge where diffusion theory is not valid.
3. Minimize a chi-squared cost function between the convolved model and the measured (normalized) DTOF:

$$\chi^2 = \sum_{i \in \text{fit window}} \frac{\big(R_{\text{meas}}(t_i) - R_{\text{model}}(t_i;\ \mu_a,\mu_s')\big)^2}{\sigma_i^2}$$

   using a nonlinear least-squares solver (Levenberg–Marquardt-type), varying $\mu_a$ and $\mu_s'$ as free parameters. This is what `getFitResult1_TRS` / `getFitResult2_TRS` perform per timepoint, per wavelength — output includes the fitted $\mu_a$, $\mu_s'$, peak position, FWHM, and the resulting $\chi^2$ (exposed in the notebook as `trs_chi2_685`, `trs_chi2_828`) as a goodness-of-fit diagnostic.

**Wavelength interpolation to the DCS wavelength.** Since $\mu_a$/$\mu_s'$ are only measured at the two TD-NIRS wavelengths (685 nm, 828 nm) but DCS operates at 785 nm, the fitted optical properties are interpolated (linear or low-order polynomial interpolation between the two measured wavelengths) to obtain $\mu_a(785\text{nm})$ and $\mu_s'(785\text{nm})$ — exposed directly as `trs_mua_785`, `trs_mus_785` in the fitted-data output, ready to be fed into the DCS model (see below).

### 2. Chromophore Concentration Unmixing (Modified Beer–Lambert / Spectral Decomposition)

At each wavelength $\lambda$, the fitted absorption coefficient is modeled as a linear combination of the known molar extinction coefficients of the tissue chromophores:

$$\mu_a(\lambda) = \varepsilon_{HbO}(\lambda)\,[HbO] + \varepsilon_{HbR}(\lambda)\,[HbR]$$

With measurements at the two wavelengths (685 nm, 828 nm), this gives a **2×2 linear system**:

$$\begin{bmatrix}\mu_a(\lambda_1)\\ \mu_a(\lambda_2)\end{bmatrix} = \begin{bmatrix}\varepsilon_{HbO}(\lambda_1) & \varepsilon_{HbR}(\lambda_1)\\ \varepsilon_{HbO}(\lambda_2) & \varepsilon_{HbR}(\lambda_2)\end{bmatrix}\begin{bmatrix}[HbO]\\ [HbR]\end{bmatrix}$$

solved directly by matrix inversion:

$$\begin{bmatrix}[HbO]\\ [HbR]\end{bmatrix} = \mathbf{E}^{-1}\begin{bmatrix}\mu_a(\lambda_1)\\ \mu_a(\lambda_2)\end{bmatrix}$$

From which:

$$[HbT] = [HbO] + [HbR], \qquad StO_2 = \frac{[HbO]}{[HbT]}\times 100\%$$

These are the `trs_hbo`, `trs_hbr`, `trs_hbt`, `trs_sto2` outputs used throughout the parameter-extraction cells. (`trs_toe`, `trs_mro2` — tissue oxygen extraction and an estimated metabolic-rate-of-oxygen-consumption proxy — are also computed by the fit routine but not used further in this notebook.)

### 3. DCS: Correlation Diffusion Model & Blood Flow Index Extraction

**Physical model.** DCS relates the decay of the light-field **autocorrelation function** $g_1(\tau)$ to scatterer (red blood cell) motion via the **correlation diffusion equation** (Boas & Yodh formalism):

$$\nabla^2 G_1(\mathbf{r},\tau) - \left[v\mu_a + \frac{1}{3}v\mu_s' k_0^2\, \alpha\langle\Delta r^2(\tau)\rangle\right] G_1(\mathbf{r},\tau) = -vS(\mathbf{r})$$

where $k_0 = 2\pi n/\lambda$ is the light wavevector in the medium, $\alpha$ is the fraction of scattering events occurring on *moving* particles (red blood cells) versus static scatterers, and $\langle\Delta r^2(\tau)\rangle$ is the mean-squared displacement of the moving scatterers over lag time $\tau$. Under the (empirically justified, not first-principles-derived) **effective Brownian-motion approximation**:

$$\langle\Delta r^2(\tau)\rangle = 6 D_b \tau$$

where $D_b$ is an *effective* Brownian diffusion coefficient standing in for blood flow.

**Semi-infinite analytical solution.** As with the TD-NIRS reflectance model, $G_1$ has a closed-form solution for a semi-infinite geometry, of the general form:

$$G_1(\rho,\tau) \propto \frac{\exp(-K(\tau)r_1)}{r_1} - \frac{\exp(-K(\tau)r_2)}{r_2}, \qquad K(\tau)=\sqrt{3\mu_a\mu_s' + \mu_s'^2 k_0^2\alpha\langle\Delta r^2(\tau)\rangle}$$

with $r_1$, $r_2$ the source-to-detector and source-to-image-detector distances (method of images, same extrapolated-boundary construction as the TD-NIRS model). **Crucially, $K(\tau)$ depends explicitly on $\mu_a$ and $\mu_s'$** — this is the formal justification for why the TD-NIRS-derived optical properties at 785 nm (§1 above) must be supplied as fixed inputs to the DCS fit: without them, a change in decay rate is ambiguous between a flow change and an optical-property change.

**From field to intensity autocorrelation.** What is actually measured by the photon-counting hardware is the **intensity** autocorrelation $g_2(\tau)$, related to the normalized field autocorrelation $g_1(\tau) = G_1(\tau)/G_1(0)$ via the **Siegert relation**:

$$g_2(\tau) = 1 + \beta\,|g_1(\tau)|^2$$

where $\beta$ is a coherence/collection-efficiency factor (fitted output `corr_beta`), close to 1 for ideal single-mode, single-speckle detection.

**Fitting procedure.** For each timepoint, a nonlinear least-squares fit of the modeled $g_2(\tau)$ (using the interpolated $\mu_a(785)$, $\mu_s'(785)$ as fixed inputs) to the measured autocorrelation curve is performed, varying the combined parameter $\alpha D_b$ (the **Blood Flow Index, BFI**, units cm²/s — exposed as `corr_bfi`) and $\beta$. Additional outputs `corr_tau_c` and `corr_tau_tail` characterize the correlation decay time constant, and `corr_r2` is the fit's coefficient of determination (goodness-of-fit diagnostic, analogous to $\chi^2$ for the TD-NIRS fit). This is what `getFitResult2_DCS` performs.

**Multi-channel handling.** With 4 parallel detection fibers, each channel's raw intensity/count-rate (`dcs_int_1`…`dcs_int_4`) can be inspected (as in the notebook's raw-DCS plotting cell) to confirm comparable photon statistics across channels before/instead of combining them in the fit.

### 4. Time-Axis Synchronization Algorithm

Because the sensor board, TD-NIRS subsystem, and DCS subsystem each run on independent internal clocks/sampling rates, a **nearest-timestamp matching** algorithm is used to align all streams to a common $t=0$ (the start of the automatic pre-acquisition quality-check phase, marked by `mark_quality_phase`):

```python
bb = np.squeeze(np.where(mark_quality_phase == 0))
differences = np.abs(system_time_fit_dcs - system_time_sensor[bb[0]])
closest_index_qf = differences.argmin()
dcs_system_time = (system_time_fit_dcs - system_time_fit_dcs[closest_index_qf]) / 1000
```

i.e., for each subsystem's raw timestamp vector, find the **index of minimum absolute time difference** to the sensor board's quality-phase-start timestamp, then re-zero that subsystem's clock relative to that matched index (converting from milliseconds to seconds). The identical procedure is applied independently to the TD-NIRS (`trs_system_time`) and DCS (`dcs_system_time`) streams.

### 5. VOT Parameter-Extraction Algorithms

**a) Baseline / recovery baseline values** — simple arithmetic mean over a manually-selected, visually-verified stable time window $[t_1,t_2]$:

$$\overline{X}_{\text{baseline}} = \frac{1}{N}\sum_{t_i \in [t_1,t_2]} X(t_i)$$

applied identically to `trs_sto2` (→ `StO2 bas`, `StO2 rec`) and `corr_bfi` (→ `BF bas.`, `BF rec`).

**b) Deoxygenation / reoxygenation slopes** — ordinary least-squares linear regression (`numpy.polyfit`, degree 1) over the selected window, minimizing:

$$\underset{m,\,b}{\arg\min}\sum_{i}\big(X(t_i) - (m\,t_i + b)\big)^2$$

- **Deoxygenation slope** ($DeOx$): fit over the first ~60 s of occlusion, slope $m$ (in %/s) rescaled to **%/min** by $\times 60$.
- **Reoxygenation slope** ($ReOx$): fit over a short (~10 s) window immediately following cuff release, reported directly in **%/s**.

**c) Area under the curve (hyperemic overshoot magnitude)** — numerical (trapezoidal) integration of the signal's deviation from its recovery baseline, over a manually-selected window spanning the overshoot:

$$AUC = \int_{t_{\text{start}}}^{t_{\text{end}}} \big(X(t) - \overline{X}_{\text{rec.\,baseline}}\big)\,dt \;\approx\; \sum_{i} \frac{\big(f_i + f_{i+1}\big)}{2}\,(t_{i+1}-t_i)$$

implemented via `scipy.integrate.trapezoid`, computed separately for `AUC StO2` and `AUC BF`.

> **Unit-conversion note (worth double-checking):** in the notebook, the StO₂ AUC is converted to %·min by **dividing** the raw trapezoidal result (in %·s) by 60, while the BFI AUC is converted by **multiplying** the raw result by 60. Since both time axes (`trs_system_time`, `dcs_system_time`) are in seconds, this asymmetry looks like it may be a units inconsistency in the current version of the script — worth verifying against the intended target units (`cm²/s · min`) before relying on the absolute `AUC BF` value across subjects.

**d) Plot y-axis robust scaling.** For the BFI trace, the upper plot limit uses the **99th percentile** rather than the raw maximum, to avoid a single motion-artifact spike compressing the visible dynamic range:

$$y_{\text{upper}} = 1.2 \times P_{99}\big(\text{corr\_bfi}\big)$$

This is a visualization convenience only — it does not affect the numerical baseline/slope/AUC calculations, which use the manually-selected windows directly on the unclipped data.

## Extracted Parameters — Definitions

| Parameter | Units | Meaning |
|---|---|---|
| `StO2 bas` | % | Baseline tissue oxygen saturation before occlusion |
| `DeOx` | %/min | Deoxygenation slope during occlusion — rate of local oxygen consumption (steeper = higher metabolic demand) |
| `ReOx` | %/s | Reoxygenation slope immediately after cuff release — rate of oxygen resupply |
| `StO2 rec` | % | Recovery baseline StO₂ after the hyperemic overshoot has resolved |
| `AUC StO2` | %·min | Integrated hyperemic overshoot in oxygenation above recovery baseline — proxy for magnitude of the vascular reactivity response |
| `BF bas.` | cm²/s | Baseline blood flow index (BFI) before occlusion |
| `BF rec` | cm²/s | Recovery baseline BFI after the hyperemic overshoot has resolved |
| `AUC BF` | cm²/s·min | Integrated hyperemic overshoot in blood flow above recovery baseline — proxy for endothelial/microvascular reactivity |

**Interpretation summary:**
- `DeOx` isolates **oxygen consumption** (delivery is deliberately zero during occlusion, so the StO₂ decline reflects metabolism alone).
- `ReOx`, `AUC StO2`, and `AUC BF` together characterize **vascular reactivity / endothelial function** — how strongly and how quickly the vasculature responds once flow is restored. A larger AUC/faster ReOx indicates more vasodilatory capacity; a blunted response can indicate impaired endothelial function.

## Example Results (this case study)

| Subject | Sex | Arm fat thick. (cm) | StO2 bas (%) | DeOx (%/min) | ReOx (%/s) | StO2 rec (%) | AUC StO2 (%·min) | BF bas. (cm²/s) | BF rec (cm²/s) | AUC BF (cm²/s·min) |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 1 | 0.8 | 71.97 | −7.64 | 1.78 | 70.66 | 11.12 | 1.10×10⁻⁸ | 1.17×10⁻⁸ | 6.6×10⁻⁵ |

*(Single-subject illustrative result from the hands-on lab session — not a validated clinical reference range.)*

## How to Run

1. Place raw HDOS measurement data under `./tests/HDOS-icfo_Mindlab_4_VASC_ENDO_<timestamp>/` (folder name must match the pattern used in the notebook's `dataFolder` variable).
2. Ensure `myFunctions_TRS.py`, `myFunctions_DCS.py`, and `HDOS2_DATAREAD_v2.py` are present alongside the notebook (or on the Python path).
3. Install dependencies: `pip install numpy pandas scipy matplotlib tqdm ipympl`
4. Open `MINDLAB_VOT_v2.ipynb` and run cells top to bottom.
5. **Manually adjust the baseline/deoxygenation/reoxygenation/AUC time windows** (given in seconds, defined by inspecting the plotted traces against the inflate/deflate markers) for each new subject/session — these are not automatically detected and will differ between measurements.
6. Update the `subject`, `sex`, and `arm_fat` variables before exporting the final results table.
7. Outputs: `sen_<data_folder_name>.csv` (raw sensor trace) and `RESULTS_VOT_<data_folder_name>.csv` (one-row summary table of extracted parameters).

## Limitations & Practical Notes

- **n = 1**: this is a single-subject case study from a teaching lab session, not a statistically powered dataset. Cross-subject comparison requires pooling many `RESULTS_VOT_*.csv` rows (see companion box-plot/summary scripts referenced in the broader MINDLAB session materials).
- **Manual window selection**: baseline/slope/AUC windows are chosen by visual inspection per subject, introducing some operator-dependent variability.
- **Motion artifacts**: BFI in particular is highly sensitive to probe/limb motion; the 99th-percentile y-axis capping is a plotting convenience, not an artifact-rejection step — accelerometer/touch-sensor channels (read but not analyzed further in this notebook) are available for more rigorous artifact flagging.
- **Contact-pressure sensitivity**: absolute StO₂/BFI values are sensitive to probe contact pressure and positioning; repeat placements on the same subject can show meaningful variability in absolute (though less so in relative/slope-based) parameters.
- **Anthropometric confound**: overlying skin/fat thickness at the probe site affects the optical signal and is recorded (`arm_fat`) to allow later correlation/adjustment across subjects.

## Acknowledgments

Data and analysis pipeline developed as part of the **MINDLAB Frontiers Research School**, ICFO — Institut de Ciències Fotòniques, Barcelona (July 2026), during the diffuse optics hands-on laboratory sessions.
