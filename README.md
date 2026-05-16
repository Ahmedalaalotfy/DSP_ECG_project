# ECG Signal Processing — MATLAB GUI App + ADC Bit Analysis

This repository contains two MATLAB tools built as Part II of a two-part project on ECG signal acquisition and processing. Part I covers the analog electronics and signal generation side (Cadence simulation). This part handles everything on the digital signal processing end — both a GUI app for ECG analysis and a standalone script for ADC output analysis.

---

## Repository Structure

```
📦 ecg-dsp-app
 ┣ 📜 app1.mlapp          ← MATLAB App Designer GUI (ECG signal analysis)
 ┣ 📜 adc_analysis.m      ← Standalone script (ADC bit reconstruction + FFT)
 ┣ 📜 README.md           ← You're reading this
 ┗ 📁 sample_data/        ← Put your test CSV files here (optional)
```

---

## Tool 1 — ECG Signal Analysis GUI (`app1.mlapp`)

A MATLAB App Designer application for loading, visualizing, filtering, and analyzing ECG signals from CSV files.

### What It Does

- Plots the raw ECG signal in the **time domain**
- Computes and plots the **FFT power spectrum** (frequency domain)
- Applies a **Butterworth low-pass filter** to clean up the noise
- Shows the **filtered signal** and its **power spectral density (PSD)**
- Computes and displays **mean, median, and variance** of the filtered signal

### Requirements

- MATLAB R2019b or later
- Signal Processing Toolbox (`butter`, `filtfilt`, `periodogram`, `hamming`)

To verify the toolbox is installed:
```matlab
ver('signal')
```

### How to Run

Open MATLAB, navigate to the project folder, and run:
```matlab
app1
```

### Using the App — Step by Step

**Step 1 — Upload your ECG file**
- Click **"Choose File"** and select your `.csv` file
- The file should contain a single column of ECG amplitude values sampled at **1000 Hz**

> The app currently assumes `fs = 1000 Hz`. If your signal has a different sampling rate, update the `fs` variable in the `PlotthedataButtonPushed` callback.

**Step 2 — Plot the raw signal**
- Click **"Plot the data"**
- The centre panel shows the raw ECG in time domain (top) and FFT power spectrum 0–100 Hz (bottom)

**Step 3 — Process and analyze**
- Click **"Compute the characteristics of the data"**
- The right panel updates with the filtered signal (bottom) and PSD in dB (top)
- The bottom-left panel displays mean, median, and variance

### CSV Format (ECG App)

A single column of numeric samples, no header:
```
0.0023
0.0041
-0.0012
0.0187
...
```

If your file has headers, modify the `readmatrix` call:
```matlab
data = readmatrix(fullfile(filepath, filename), 'NumHeaderLines', 1);
```

### What's Happening Under the Hood

| Step | Code | What it does |
|---|---|---|
| Time vector | `t = (0:N-1)/fs` | Maps sample indices to seconds |
| FFT | `Y = fft(data)` | Converts signal to frequency domain |
| Power spectrum | `P = abs(Y).^2 / N` | Energy per frequency bin |
| Butterworth filter | `butter(4, 50/(fs/2))` | 4th-order LPF, 50 Hz cutoff |
| Zero-phase filter | `filtfilt(b, a, data)` | Filters without shifting peaks |
| PSD | `periodogram(..., hamming(N), ...)` | Windowed spectral density estimate |
| Statistics | `mean`, `median`, `var` | DC offset, robust center, AC power |

### Known Limitations

- `fs = 1000 Hz` is hardcoded — update it manually if needed
- Global variables (`data`, `fs`, `t`) are used between callbacks — works fine but not ideal for large-scale extension
- No guard against pressing buttons before loading a file — always load the CSV first
- The PSD panel plots the **original** signal's spectrum. If you want the filtered version there instead, change `Pxx_original` to `Pxx_filtered` in the compute callback

---

## Tool 2 — ADC Bit Analysis Script (`adc_analysis.m`)

A standalone MATLAB script that reads raw ADC bit output from a CSV file, reconstructs the analog voltage from the 8-bit binary columns, and analyzes the signal in both the time and frequency domains.

### What It Does

- Reads 8 binary bit columns (b7 to b0) from a CSV file
- Reconstructs the decimal code and converts it to a voltage using a reference voltage
- Plots the **reconstructed analog signal** in the time domain
- Removes the DC component and computes the **FFT spectrum** with zero-padding
- Plots the **normalized FFT magnitude in dB** from 0 to 1000 Hz

### Requirements

- MATLAB (any reasonably recent version)
- Signal Processing Toolbox (for `nextpow2` — though this is a base MATLAB function)

### How to Run

1. Place your `adc_bits.CSV` file in the same folder as the script
2. Open MATLAB, navigate to the folder, and run:
```matlab
adc_analysis
```

### CSV Format (ADC Script)

The file must have **exactly 8 columns**, one per bit from MSB to LSB:

```
b7, b6, b5, b4, b3, b2, b1, b0
1,  0,  1,  1,  0,  0,  1,  0
0,  1,  0,  0,  1,  1,  0,  1
...
```

The script reconstructs the integer code as:
```
code = b7×128 + b6×64 + b5×32 + b4×16 + b3×8 + b2×4 + b1×2 + b0
```
Then scales it to voltage:
```
voltage = code × Vref / 255
```

### Parameters You Can Change

At the top of `adc_analysis.m`:

| Parameter | Default | Description |
|---|---|---|
| `filename` | `'adc_bits.CSV'` | Input CSV file name |
| `Vref` | `10e-6` | ADC reference voltage (V) |
| `fs` | `20480` | Sampling frequency (Hz) |
| `fin` | `100` | Input sine frequency (Hz) — used for THD reference |
| `numHarm` | `5` | Number of harmonics for THD calculation |

### What's Happening Under the Hood

**Bit-to-voltage conversion**
The 8 binary columns are combined into a decimal integer using standard binary weighting (b7 is the MSB with weight 128, b0 is the LSB with weight 1). The result is then scaled linearly to a voltage range using the reference voltage and the full-scale value of 255 (= 2⁸ − 1).

**DC removal**
Before computing the FFT, the mean value is subtracted from the signal. This removes the DC component (the zero-frequency spike) so the spectrum only shows the AC content, which is what you care about for evaluating ADC performance.

**Rectangular window**
A rectangular window (all ones) is used, which gives the narrowest main lobe in the frequency domain. This is intentional — it gives the sharpest frequency resolution, which is useful when you want to clearly identify harmonic peaks for THD analysis.

**Zero-padding**
The FFT is computed with `Nfft = 8 × 2^nextpow2(N)` points — much larger than the signal length. This doesn't add any real frequency resolution, but it interpolates the spectrum to give a smoother, denser plot that's easier to read.

**Normalized dB plot**
The spectrum is normalized to its own peak (`P1 / max(P1)`) before converting to dB. This means the fundamental tone always sits at 0 dB, and everything else is shown relative to it. A small epsilon (`1e-12`) is added before taking the log to avoid `log(0)` errors.

---

## Part I — Electronics (Cadence)

Both scripts in this repo handle the digital processing side only. The analog front-end — the circuit that generates, amplifies, and conditions the ECG/ADC signal before digitization — is documented separately as Part I of the project (Cadence simulation).

---

## License

Open for academic and educational use. Feel free to use, modify, or build on it — just give credit if you use it in your own work.

---

## Author

Built as part of a Biomedical / Mechatronics engineering project.
Feel free to open an issue or pull request if you spot a bug or want to suggest an improvement.
