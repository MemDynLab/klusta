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


##### mask computation 
Mask computation is not performed on the features vectors as described in Kadir et al. *Neur Comp* 2014, but it is rather
based on thresholding on the raw signals. the  mask will correspond to a connected component containing at least one channel 
exceeding `threshold_strong_std_factor` and containing all connected sites that exceed `threshold_weak_std_factor`. 
If I am correct, this may mean that in cases where the masked EM behavior is not desired (for example, tetrodes), 
one could just set `threshold_weak_std_factor` to zero. In this case, the mask should be assured to cover all sites for 
all events, effectively reverting to standard EM. 


