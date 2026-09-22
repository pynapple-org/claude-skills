# Data Structures Reference

## Ts (Timestamps)

Container for event timestamps with no associated data (e.g., spike times).

```python
# Constructor
ts = nap.Ts(t=[1.0, 2.5, 3.7], time_units="s")
ts = nap.Ts(t=spike_times_ms, time_units="ms")  # auto-converts to seconds
ts = nap.Ts(t=times, time_support=epoch)         # restrict to epoch

# Properties
ts.times()         # timestamps as numpy array
ts.time_support    # IntervalSet of valid time range
ts.rate            # frequency (events per second)
ts.shape           # (n_events,)
ts.start_time()    # first timestamp
ts.end_time()      # last timestamp
```

## Tsd (Time Series Data - 1D)

1D time series with timestamps and data values (e.g., LFP, position, single neuron rate).

```python
# Constructor
tsd = nap.Tsd(t=timestamps, d=values)
tsd = nap.Tsd(t=timestamps, d=values, time_support=epoch)

# Properties
tsd.values         # data as numpy array (same as tsd.d)
tsd.times()        # timestamps as numpy array
tsd.rate           # sampling rate (Hz)
tsd.time_support   # IntervalSet

# Indexing
tsd[0]             # first time point (returns Tsd)
tsd[10:20]         # slice by index
tsd.get(50.0)      # closest value to time 50.0
tsd.get(50, 60)    # values between time 50 and 60

# NumPy compatibility
np.mean(tsd)       # works directly
np.abs(tsd)        # returns Tsd
tsd + 1            # arithmetic returns Tsd
tsd[tsd > 0]       # boolean indexing
```

**Don't manually rebuild the `Tsd`/`TsdFrame` a pynapple operation already gives you back.**
A reduction like `.mean(axis)` on a `TsdFrame` already returns a properly time-stamped `Tsd`
(right `t`, `time_support`, `rate`) -- there's a recurring habit of pulling `.values` back out
and reconstructing one by hand instead of just using the result directly:

```python
# unnecessary -- rebuilds a Tsd that .mean(1) already returned
lfp = nap.Tsd(t=eeg.t, d=eeg[:, channels].values.mean(1), time_support=eeg.time_support)

# eeg[:, channels] is a TsdFrame; .mean(1) is a pynapple method, not bare numpy --
# it already returns a Tsd with the right t/time_support
lfp = eeg[:, channels].mean(1)
```
Same applies to numpy ufuncs called on a single pynapple object (`np.cos(tsd)`, `np.abs(tsd)`,
etc. -- see below) and to methods like `.smooth()`/`.bin_average()`/`.restrict()`: if a
pynapple operation already produced the object you want, use its result directly instead of
extracting `.values`/`.t` and passing them back into a fresh `nap.Tsd(...)` constructor.

**NumPy dispatch works for a pynapple object combined with a plain scalar/array, but not for
two pynapple objects together** -- even a `Tsd`/`TsdFrame` with another `Tsd`/`TsdFrame` that
has the exact same timestamps (e.g. two reductions computed from the same perievent call).
`__array_ufunc__` returns `NotImplemented` for that case rather than aligning them, so it
raises instead of silently producing a wrong result -- but it's easy to hit by accident when
chaining reductions of the same pynapple object:

```python
cos_m = np.nanmean(np.cos(perievent), axis=1)   # Tsd (numpy dispatch collapses the column axis)
sin_m = np.nanmean(np.sin(perievent), axis=1)   # Tsd, same timestamps as cos_m

cos_m ** 2 + sin_m ** 2
# TypeError: operand type(s) all returned NotImplemented from __array_ufunc__(<ufunc 'add'>, ...): 'Tsd', 'Tsd'

# fix: drop to .values for the combining step (still fine to wrap the result back in a
# Tsd afterwards using either side's timestamps, since they're the same)
R = np.sqrt(cos_m.values ** 2 + sin_m.values ** 2)
```

## TsdFrame (Time Series Data - 2D)

2D time series with labeled columns (e.g., multi-neuron calcium imaging, multi-channel LFP).

```python
# Constructor
tsdframe = nap.TsdFrame(
    t=timestamps,
    d=data_2d,                          # shape: (n_times, n_columns)
    columns=["neuron_0", "neuron_1"],   # optional column names
    time_support=epoch,                  # optional
    metadata={"region": ["M1", "V1"]}   # optional per-column metadata
)

# From numpy arrays
tsdframe = nap.TsdFrame(
    t=position.t,
    d=np.stack([position, speed]).T,
    time_support=position.time_support,
    columns=["position", "speed"],
)

# Column access
tsdframe[:, 0]                 # first column -> Tsd
tsdframe[:, [0, 2]]           # columns 0 and 2 -> TsdFrame
tsdframe["neuron_0"]           # by column name -> Tsd (deprecated, use loc)
tsdframe[:, 0:3]              # slice columns -> TsdFrame

# Metadata-based column selection
tsdframe.loc["neuron_0"]       # by column name
tsdframe.loc[["neuron_0", "neuron_1"]]  # multiple columns
tsdframe[:, tsdframe.region == "M1"]    # filter by metadata
tsdframe.depth = [10, 20, 10]           # attach a new column via direct attribute
                                         # assignment -- same as tsdframe.set_info(depth=...)

# Properties
tsdframe.columns       # column labels
tsdframe.shape         # (n_times, n_columns)
tsdframe.metadata      # DataFrame of per-column metadata
```

## TsdTensor (Time Series Data - 3D+)

Multi-dimensional time series (e.g., video frames, spatial maps over time).

```python
# Constructor
tensor = nap.TsdTensor(
    t=timestamps,
    d=data_3d,       # shape: (n_times, height, width) or higher
    time_support=epoch
)

# Properties
tensor.shape    # (n_times, dim1, dim2, ...)
tensor.ndim     # >= 3

# Indexing
tensor[0]       # first frame
tensor[:, 0]    # first row across all frames
```

## TsGroup (Group of Spike Trains)

Dictionary-like container for multiple Ts/Tsd objects, each with potentially different
timestamps (e.g., spike trains from multiple neurons).

```python
# Constructor from dict
tsgroup = nap.TsGroup(
    {0: nap.Ts(spikes_0), 1: nap.Ts(spikes_1), 2: nap.Ts(spikes_2)},
    time_support=epoch,
    metadata={"location": ["adn", "adn", "v1"]}
)

# Constructor from arrays (auto-wrapped in Ts)
tsgroup = nap.TsGroup({0: times_0, 1: times_1})

# Accessing individual neurons
tsgroup[0]              # Ts for neuron 0
tsgroup[[0, 2]]         # TsGroup with neurons 0 and 2

# Properties
tsgroup.index           # neuron IDs
tsgroup.keys()          # same as index
len(tsgroup)            # number of neurons
tsgroup.rates           # firing rates array
tsgroup.time_support    # shared time support

# Iteration
for key in tsgroup:
    print(key, tsgroup[key])

# Convert to single Tsd (for raster plots)
raster = tsgroup.to_tsd([-1])  # all spikes with value -1
plt.plot(raster, "|")

# Metadata access
tsgroup.location          # metadata column as array
tsgroup["location"]       # same
tsgroup.set_info(pref_ang=preferred_angles)

# direct attribute assignment works too, and is equivalent to set_info() -- no need to
# build a filter mask by hand (e.g. looping per-row against a dict of per-category
# thresholds) when you can attach the computed column and then boolean-index on it directly
tsgroup.SI = spatial_info               # same effect as tsgroup.set_info(SI=spatial_info)
tsgroup = tsgroup[tsgroup.SI > 0.2]     # filter using the newly attached column
```

## IntervalSet (Time Intervals / Epochs)

Set of non-overlapping time intervals. Overlapping intervals are automatically merged.

```python
# Constructor
ep = nap.IntervalSet(start=[0, 100], end=[50, 150])
ep = nap.IntervalSet(start=10, end=20)              # single interval
ep = nap.IntervalSet(start=starts_ms, end=ends_ms, time_units="ms")

# With metadata
ep = nap.IntervalSet(
    start=[0, 100], end=[50, 150],
    metadata={"trial_type": ["go", "nogo"]}
)

# From sorted array (alternating start/end)
ep = nap.IntervalSet(np.sort(np.random.uniform(0, 100, 20)))

# Properties
ep.start           # start times array
ep.end             # end times array
ep.shape           # (n_intervals, 2)
ep.tot_length("s") # total duration of all intervals

# Indexing
ep[0]              # first interval as IntervalSet
ep[[0, 2]]         # intervals 0 and 2
ep.start[0]        # start time of first interval
ep.end[0]          # end time of first interval

# Set operations
ep1.intersect(ep2)     # intersection
ep1.union(ep2)         # union
ep1.set_diff(ep2)      # difference (ep1 minus ep2)

# Filtering
ep.drop_short_intervals(threshold=0.5)    # remove intervals < 0.5s
ep.drop_long_intervals(threshold=10.0)    # remove intervals > 10s
ep.merge_close_intervals(threshold=0.1)   # merge gaps < 0.1s

# Splitting
ep.split(interval_size=1.0)   # split into 1-second intervals

# Metadata
ep.tags                        # metadata column "tags" if exists
ep[ep.tags == "wake"]          # filter by metadata
ep.set_info(condition=["A", "B"])
ep.condition = ["A", "B"]      # same as set_info() -- direct attribute assignment works
                                # here too, same as on TsGroup/TsdFrame
```

## Converting Between Types

```python
# TsdFrame column -> Tsd
single_channel = tsdframe[:, 0]

# Multiple Tsd -> TsdFrame
combined = nap.TsdFrame(
    t=tsd1.t,
    d=np.column_stack([tsd1.values, tsd2.values]),
    columns=["ch1", "ch2"]
)

# Tsd -> numpy
arr = tsd.values        # data only
times = tsd.times()     # timestamps only

# TsGroup -> count matrix (TsdFrame)
count_matrix = tsgroup.count(bin_size=0.01)
```
