# Signal Processing Reference

## Filtering

All filter functions work on Tsd, TsdFrame, and TsdTensor. They handle discontinuous
epochs automatically (filter applied per epoch).

### Bandpass Filter

```python
# Extract theta oscillation (6-12 Hz)
theta_band = nap.apply_bandpass_filter(lfp, cutoff=(6, 12), fs=1250)

# Extract gamma (30-80 Hz)
gamma_band = nap.apply_bandpass_filter(lfp, cutoff=(30, 80), fs=1250)
```

### Lowpass / Highpass / Bandstop

```python
# Lowpass below 100 Hz
filtered = nap.apply_lowpass_filter(signal, cutoff=100, fs=1250)

# Highpass above 1 Hz
filtered = nap.apply_highpass_filter(signal, cutoff=1, fs=1250)

# Notch filter (remove 60 Hz line noise)
filtered = nap.apply_bandstop_filter(signal, cutoff=(59, 61), fs=1250)
```

### Filter Parameters

```python
nap.apply_bandpass_filter(
    sig,                            # Tsd/TsdFrame/TsdTensor
    cutoff,                         # float or tuple of (low, high)
    fs=None,                        # sampling rate (auto from sig.rate if None)
    order=4,                        # filter order
    filter_type='butterworth',      # 'butterworth' or 'windowed-sinc'
    transition_bandwidth=None,      # for windowed-sinc only
)
```

### Analyzing Filter Response

```python
freqs, response = nap.get_filter_frequency_response(
    cutoff=(6, 12),
    filter_type='butterworth',
    fs=1250,
    order=4
)
plt.plot(freqs, response)
plt.xlabel("Frequency (Hz)")
plt.ylabel("Magnitude")
```

## Wavelet Transform

Morlet wavelet decomposition for time-frequency analysis.

```python
# Define frequency range
freqs = np.geomspace(5, 200, 100)  # 100 log-spaced frequencies, 5-200 Hz

# Compute wavelet transform
cwt = nap.compute_wavelet_transform(lfp, fs=1250, freq=freqs)
# Returns TsdFrame: (n_times, n_freqs), complex-valued

# Extract amplitude
amplitude = np.abs(cwt.values)

# Visualize spectrogram
plt.pcolormesh(cwt.t, freqs, amplitude.T)
plt.yscale('log')
plt.ylabel("Frequency (Hz)")
plt.xlabel("Time (s)")
plt.colorbar(label="Amplitude")
```

### Alternative: Specify Frequency Range

```python
# Auto-generate frequencies
cwt = nap.compute_wavelet_transform(
    lfp,
    fs=1250,
    freq=(0.5, 100),    # (min_freq, max_freq)
    nb_freqs=100         # number of frequencies
)
```

## Spectral Analysis

### FFT

```python
fft_result = nap.compute_fft(signal, fs=1250)
# Returns: pandas Series with frequency index

plt.plot(fft_result.index, np.abs(fft_result.values))
plt.xlabel("Frequency (Hz)")
plt.ylabel("Amplitude")
```

### Power Spectral Density

```python
psd = nap.compute_power_spectral_density(signal, fs=1250)

# Average PSD across a group
mean_psd = nap.compute_mean_power_spectral_density(tsgroup, fs=1250)
```

## Phase Extraction (Hilbert Transform)

Extract instantaneous phase from a bandpass-filtered signal using scipy:

```python
import scipy as sp

# 1. Restrict LFP to epochs of interest
lfp_restricted = lfp.restrict(forward_ep)

# 2. Bandpass filter for theta
theta_band = nap.apply_bandpass_filter(lfp_restricted, (6, 12), fs=1250)

# 3. Hilbert transform for instantaneous phase
phase = np.angle(sp.signal.hilbert(theta_band.values))
phase %= 2 * np.pi  # wrap to [0, 2pi]

# 4. Store as Tsd
theta_phase = nap.Tsd(
    t=theta_band.t,
    d=phase,
    time_support=theta_band.time_support
)
```

## Oscillatory Event Detection

Detect oscillatory bursts (e.g., sharp-wave ripples, spindles). Also reachable as
`nap.process.signal.detect_oscillatory_events` -- `nap.detect_oscillatory_events` is the
same function. Internally: bandpass filter -> Hilbert envelope -> smooth -> z-score ->
threshold -> duration filtering -> merge close events -> extract per-event peak.

```python
events = nap.detect_oscillatory_events(
    data,                       # Tsd -- single-channel raw signal (not multi-channel)
    epochs,                      # IntervalSet -- restrict detection to these epochs
    frequency_band,                # (low, high) tuple, Hz, e.g. (100, 250) for ripples
    threshold_band,                  # (low, high) tuple -- z-scored envelope thresholds
    duration_band,                     # (min, max) tuple -- event duration, seconds
    min_interval,                        # merge events closer than this apart, seconds
    fs=None,                                # sampling rate; inferred from data.rate if omitted
    sliding_window_size=51,                    # smoothing window size, in samples
)
```

Returns a single `IntervalSet` (not a tuple) -- event start/end as the interval bounds, with
per-event **metadata already attached**: `power` (dB), `amplitude`, `peak_time`.

```python
ripples = nap.detect_oscillatory_events(
    lfp, sleep_ep, frequency_band=(100, 250), threshold_band=(1, 10),
    duration_band=(0.02, 0.2), min_interval=0.02,
)
peak_times = nap.Ts(ripples.peak_time.values)   # metadata column -> a Ts of each event's peak
strongest = ripples[ripples.amplitude > ripples.amplitude.quantile(0.9)]
```

**`data` must be single-channel.** For a multi-channel region/shank, reduce to one
representative trace first (e.g. average across non-noise channels) -- the function raises
`TypeError` on a `TsdFrame`.

**Metadata columns aren't touched by `as_units()`.** `some_interval_set.as_units("ms")` only
converts the `start`/`end` index, not attached metadata like `peak_time` -- convert those by
hand (`* 1000`) if you need everything in the same unit, e.g. writing a peak/start/stop event
file.

## Peak Detection

Find discrete peak events in a `Tsd` (e.g. turn-onset detection from an angular-velocity
trace). Wraps `scipy.signal.find_peaks` -- same keyword arguments (`height`, `distance`,
`prominence`, etc.) -- but takes a `Tsd` and returns pynapple objects with real timestamps
instead of integer sample indices.

```python
peaks = tsd.find_peaks(height=thr)
# Returns: Tsd of peak times -> values (same units as tsd)

peaks_with_props = tsd.find_peaks(height=thr, return_prop=True)
# Returns: TsdFrame, one row per peak -- "peak_value" (the signal's value at the peak)
# plus "peak_heights" (and any other scipy peak property requested, e.g. "prominences")

# find troughs (negative peaks) by flipping the sign first
troughs = (tsd * -1).find_peaks(height=thr)

# restrict peak search to specific epochs
peaks = tsd.find_peaks(height=thr, epochs=wake_ep)
```

## Perievent Analysis

Align data to reference events (e.g., stimulus onsets, spike times). One function,
`compute_perievent`, dispatches automatically on `data`'s type -- there is no separate
"continuous" variant to call (an older API had `timestamps=`/`tref=`/`minmax=` and a
distinct `compute_perievent_continuous`; neither exists anymore -- check installed
version if code you're reading uses those names, it will raise `TypeError`).

```python
perievent = nap.compute_perievent(
    data,                    # Ts, Tsd, TsdFrame, TsdTensor, or TsGroup -- what to align
    events,                   # Ts, Tsd, TsdFrame, or TsdTensor -- events to align to
    window,                    # float (symmetric, e.g. 1.0 -> +/-1s) or (before, after)
                                # tuple, e.g. (-1, 2)
    time_unit='s',               # units of `window` ('s', 'ms', 'us')
    epochs=None,                   # restrict to these epochs; default: data.time_support
)
```

`events` accepts any existing pynapple time series directly (only its timestamps are
used) -- no need to wrap it in `nap.Ts(t=...)` first:

```python
# turns is already a Tsd (e.g. from find_peaks) -- pass it as-is
perievent = nap.compute_perievent(pop_rate, turns, window=1.0)
```

Dispatch by `data`'s type:
- **discrete**, a single `Ts`: returns a `TsGroup`, one element per event
- **discrete**, a `TsGroup`: returns a dict of `TsGroup`, one per unit
- **continuous**, `Tsd`/`TsdFrame`/`TsdTensor` (must be regularly sampled): returns one
  `TsdFrame`/`TsdTensor` with one column/slice per event and a shared relative-time index

**For continuous data, the output's relative-time axis is not a fixed function of
`window` and the data's bin size -- read it fresh from the result each time, don't
precompute an expected length.** The normal case is per-event NaN-padding: an event
whose window partially runs past available data gets NaN for the missing part, and every
other event keeps its full window. But if this can't be resolved for the batch as a
whole, the entire *shared* axis for that call can come back shrunk (fewer points,
narrower range) instead -- observed in practice when an event in a batch sits closer than
`window` to the edge of its epoch's `time_support`. This can vary from one call to the
next (e.g. one call per session, or per brain state) even with identical `window`/bin
size, because it depends on how close events happen to sit to an epoch edge in that
particular batch.

```python
# don't assume this always holds -- it can legitimately fail for some calls:
n_expected = int(round(2 * window / bin_size)) + 1
assert perievent.t.shape[0] == n_expected
```

### Event-Triggered Average

```python
# Average LFP waveform around events
eta = nap.compute_event_trigger_average(
    timeseries=lfp,                 # Tsd/TsdFrame/TsdTensor
    event=spike_times,              # Ts/Tsd
    minmax=(-0.01, 0.01),           # 10ms before/after
)
# Returns: TsdFrame with averaged waveform
```

## Correlograms

### Autocorrelogram

```python
ac = nap.compute_autocorrelogram(
    tsgroup,
    binsize=0.001,          # 1ms bins
    windowsize=0.1,         # +/- 100ms window
    ep=wake_ep,             # restrict to epoch
    norm=True               # normalize
)
```

### Cross-Correlogram

```python
cc = nap.compute_crosscorrelogram(
    tsgroup,
    binsize=0.001,
    windowsize=0.1,
    ep=wake_ep
)

# Between two specific groups
cc = nap.compute_crosscorrelogram(
    (group1, group2),
    binsize=0.001,
    windowsize=0.5
)
```

### Event Correlogram

```python
ec = nap.compute_eventcorrelogram(
    tsgroup,
    event=stimulus_ts,
    binsize=0.01,
    windowsize=1.0
)
```

### Interspike Interval Distribution

```python
isi = nap.compute_isi_distribution(
    tsgroup,
    binsize=0.001,
    windowsize=0.1,
    ep=wake_ep
)
```

## Randomization / Bootstrapping

```python
# Jitter spike times
jittered = nap.jitter_timestamps(tsgroup, max_jitter=0.01)

# Circular shift within epochs
shifted = nap.shift_timestamps(tsgroup, min_shift=0.0, max_shift=10.0)

# Poisson resampling
resampled = nap.resample_timestamps(tsgroup)

# Shuffle within intervals
shuffled = nap.shuffle_ts_intervals(tsgroup)
```
