### Default spikedetekt procedure 

The main function with the workflow is SpikeDetekt.run_serial() 
(file spikedetekt.py:671)

parameters for spike detection are in the .prm file, defaults are 

```python
spikedetekt = {
    'filter_low': 500.,
    'filter_high_factor': 0.95 * .5,  # will be multiplied by the sample rate
    'filter_butter_order': 3,

    # Data chunks.
    'chunk_size_seconds': 1.,
    'chunk_overlap_seconds': .015,

    # Threshold.
    'n_excerpts': 50,
    'excerpt_size_seconds': 1.,
    'use_single_threshold': True,
    'threshold_strong_std_factor': 4.5,
    'threshold_weak_std_factor': 2.,
    'detect_spikes': 'negative',

    # Connected components.
    'connected_component_join_size': 1,

    # Spike extractions.
    'extract_s_before': 10,
    'extract_s_after': 10,
    'weight_power': 2,

    # Features.
    'n_features_per_channel': 3,
    'pca_n_waveforms_max': 10000,

}

```

1. data is chunked (TODO by default it seems not to be used)
1. A SpikeDetektStore is created to store the results (providing functionality for chunking)
1. a threshold is calculated, see below for details 



##### threshold calculation
The main function is `compute_threshold()` in detect.py:18.
The calculation of thresholds is based on computing the standard deviation from the filtered signal. For computation of 
the standard deviation, signal is rectified, and then the median is computed. STD is taken to be median * 0.6745. The 
coefficient stems from the relationship between standard deviations and interquartile range for  normal distribution. 
This amounts to say that we expect the signal to be distributed by a normal distribution plus outliers (using the 
interquartile range helps discounting big outliers which would affect the std calculation). Actual thresholds are taken 
to be a multiple of that. Perhaps it is more correct to say that thresholds are defined in terms of the interquartile 
range. 

A slightly worrying point is that the threshold is computed on the rectified signal, which to say, based on fluctuations
both negative and positive going. median values are likely to be take from periods that do not correspond to a sortable 
spike. Therefore, they will reflect noise which will be of biological origin, and made up of spikes. Therefore, the 
median will depend on the distribution of the noise signal and it may be affected by its asymmetries. If the noise 
distribution changes (for example from one brain location to another) this will affect the value of thresholds in a 
possibly undesired way. Also it is to be noticed that a DC shift, while unlikely, could still survive filtering, and 
affect interquartile range calculation. 

Another worrying point is that the threshold is only computed from a few samples of the signal: `n_excerpts` blocks of 
`excerpt_size_seconds`. By default parameters, this means that only 50 1 second periods are considered. This is probably OK 
 for head fixed, low noise/artifact conditions, but can be problematic for e.g. freely moving situations. Increasing the 
  number/duration of excerpts seems advisable in those cases (note that this will affect running time a bit as filtering 
  is done on the excerpts for threshold finding once, and then for the actual spike detection a second time, but this will
  probably be negligible). 

if `use_single_threshold` is `True`, the threshold is computed from all the channels at the same time. If `False` the 
threshold is computed separately for different channels. This may be useful if channels have very different impedance 
(sometimes the case for tetrodes) of for e.g. different brain locations with different noise levels (for example 
different shanks). 
 
##### mask computation 
Mask computation is  performed as described in Kadir et al. *Neur Comp* 2014, but it is 
based on thresholding on the raw signals, not the features. The  mask will correspond to a connected component containing at least one channel 
exceeding `threshold_strong_std_factor` and containing all connected sites that exceed `threshold_weak_std_factor`. 
If I am correct, this may mean that in cases where the masked EM behavior is not desired (for example, tetrodes), 
one could just set `threshold_weak_std_factor` to zero. In this case, the mask should be assured to cover all sites for 
all events, effectively reverting to standard EM. However, the mask will in this case fall in the "transition zone" between 
zero and one and be effectively proportional to peak amplitude.

##### spike detection 
the workflow is in `SpikeDetekt.stepdetect` in spikedetekt.py:554. 
Detection is performed on overlapping chunks, whose length/overlap is controlled by
```python
# Data chunks.
'chunk_size_seconds': 1.,
'chunk_overlap_seconds': .015,
``` 

The detection also takes care of calculating connected components for each spike. Connected components are computed 
by a spatio-temporal floodfill algorithm, looking at timepoints/channel at which the weak threshold is computed. A temporal jitter of 
`connected_component_join_size` is allowed between neighboring channel. 
if the weak threshold is zero, then connection takes place regardless, therefore `join_size` can be safely set to zero 
TODO make sure this is true.

##### spike extraction 
The spike extraction is performed by the `WaveformExtractor` class at waveform.py:43. 
This acts on the filtered and transformed (e.g. sign is changed if thresholding is negative going) data
How many samples are extracted is determined by the two parameters `extract_s_before` and `extract_s_after`. These refer to the 
number of samples extracted for each waveform before and after the peak time. Peak time is computed as a weighted average of the
peak times on all channels overcoming threshold. The weights computed from the mask (in `WaveformExtractor.spike_sample_aligned`, 
waveform.py:130), raised to the power `weight_power`. This means that all channels overcoming strong threshold are weighted 
equally (probably a wise choice). 

##### PCA computation 
Key computations taking place in _compute_pcs in pca.py:16
For each channel, pca is computed from all the unmasked spikes (mask > 0, regardless of mask value) for that specific channel. 

As (especially 
for big probes) it could be that some channels had very few spikes unmasked, a "regularization" 
covariance matrix , which is computed from all spikes from all channels is added to the channel
based covariance matrix with weight equal to 1/N_spikes (see pca.py:64). Therefore its weight will be 
quite negligible unless there are very few unmasked spikes on that channel. 

##### feature extractions 
on each channel, projections on the principal components are computed 

##### saving data
if legacy output is requested, spikes are saved in numpy txt format.

##### TODOs
- can you concatenate files directly in spikedetekt?
spikedetekt.py:558
```python

# Use the recording offsets when dealing with multiple recordings.
        self.recording_offsets = getattr(traces, 'offsets', [0, len(traces)])
```

- get rid of transition zone for binary thresholding. this would have to be done in the WaveformExtractor or slightly downstream
 of that 
 
 
 
#### parameters for spikedetekt for tetrodes v1
```python
spikedetekt = {
    'filter_low': 500.,
    'filter_high_factor': 0.95 * .5,  # will be multiplied by the sample rate
    'filter_butter_order': 3,

    # Data chunks.
    'chunk_size_seconds': 1.,
    'chunk_overlap_seconds': .015,

    # Threshold.
    'n_excerpts': 50,
    'excerpt_size_seconds': 20., # allow for more averaging in finding thresholds
    'use_single_threshold': True,
    'threshold_strong_std_factor': 4.5,
    'threshold_weak_std_factor': 0., # to include all channels in the mask
    'detect_spikes': 'negative',
    'binary_masks': True, # ADDED parameter, make masks all 0 or 1s. The joint effect of these parameter values should to 
                          # have all masks values at 1, which is likely desirable for tetrodes. 

    # Connected components.
    'connected_component_join_size': 1,

    # Spike extractions.
    'extract_s_before': 10,
    'extract_s_after': 10,
    'weight_power': 2,

    # Features.
    'n_features_per_channel': 3,
    'pca_n_waveforms_max': 10000,

}
```