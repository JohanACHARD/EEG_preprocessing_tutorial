# EEG_preprocessing_tutorial

Python tutorial for EEG preprocessing and analysis from BioSemi BDF recordings, including event alignment, bad-channel interpolation, re-referencing, filtering, ICA-based ocular artifact correction, residual artifact rejection, current source density (CSD) transformation, event-based segmentation, event-related potentials, and time-frequency analysis of motor-preparatory activity.

## Overview

This repository provides a tutorial-oriented version of the EEG preprocessing and analysis workflow developed for delayed decision.

The complete empirical dataset and analysis code associated at this tutorial are currently available on the Open Science Framework (OSF):
[https://osf.io/86cbw/overview?view_only=1f47b76612334dc082318a501fda5a4c](https://osf.io/86cbw/overview?view_only=1f47b76612334dc082318a501fda5a4c)

This GitHub repository offers a series of explanatory Jupyter notebooks adapted to illustrate and facilitate the monitoring, visualization, and modification of the main steps of EEG processing.

The example is based on EEG acquired during a delayed perceptual decision-making task. Participants judged the direction of random-dot motion, withheld the corresponding manual response during a delay, and executed the response when a response signal was presented. This paradigm is useful for illustrating EEG preprocessing because the same trial contains several experimentally meaningful events—stimulus onset, stimulus offset, response-signal onset, and the behavioral response-and because motor-preparatory activity can be examined both in the time and time-frequency domains.

The tutorial follows the data from the continuous recording to cleaned single-trial signals and then to event-related and spectral measures of lateralized motor preparation.

## Workflow

### 1. EEG preprocessing and conversion to epochs

`1-EEG_tutorial_preprocessing_to_epochs.ipynb`

The first notebook establishes the correspondence between the continuous electrophysiological recording and the experimental events and performs the preprocessing steps that are best applied before trial segmentation.

The example recordings originate from a BioSemi ActiveTwo system. EEG was recorded from 64 active scalp electrodes, together with electrodes used to derive vertical and horizontal electro-oculographic (EOG) signals. Because subsequent analyses depend on precise temporal alignment between neural activity and task events, the event structure is first reconstructed and checked against the experimental information before the continuous EEG is segmented.

The continuous signal is then inspected for channels exhibiting excessive variance, large drifts, or unstable activity. Channels considered defective are reconstructed from the remaining electrodes using spherical-spline interpolation. This procedure is preferable to simply discarding a channel when later analyses require a common electrode configuration across participants. After interpolation, EEG data are re-referenced to the average of the scalp electrodes.

Filtering is performed on the continuous signal rather than independently within each trial. This limits the repetition of filter edge effects at epoch boundaries. For the data used in the final analyses, the EEG and EOG signals are filtered between 0.1 and 40 Hz using finite-impulse-response (FIR) filters with a Hamming window. The 0.1-Hz high-pass filter attenuates very slow drifts while preserving low-frequency event-related activity, whereas the 40-Hz low-pass filter attenuates higher-frequency activity outside the range considered in the study.

Once the continuous data and events have been prepared, the signal is segmented into long stimulus-locked epochs encompassing the complete trial. In the original study, these epochs extended from -0.6 to 9 s relative to stimulus onset. Keeping relatively long epochs at this stage preserves the different task periods within a common trial structure and allows the cleaned data to be realigned to other events later in the workflow.

This notebook therefore produces the basic epoched datasets on which the subsequent ICA and artifact-correction procedures operate.

---

### 2. Ocular artifact correction using independent component analysis

`2-EEG_tutorial_ICA.ipynb`

The second notebook addresses ocular artifacts. Blinks and eye movements often generate voltages much larger than the cortical signals of interest, particularly at frontal electrodes. Simply rejecting every trial containing such activity can lead to substantial data loss. Independent component analysis (ICA) provides an alternative by decomposing the multichannel EEG into components with relatively independent temporal dynamics, some of which can be identified as predominantly reflecting ocular activity and removed before reconstructing the EEG signal.

ICA decomposition is sensitive to slow drifts. For this reason, the decomposition is not estimated directly from the 0.1-Hz high-pass-filtered data used in the final analyses. Instead, a separate copy of the continuous EEG is high-pass filtered at 1 Hz and segmented into the same long trial epochs. This more aggressive high-pass filter is used only to obtain a more stable ICA decomposition; it is not imposed on the EEG signals subsequently analyzed.

The tutorial then illustrates how candidate ocular components can be identified by comparing independent-component activity with the vertical and horizontal EOG signals. In the original analysis, MNE-Python's `find_bads_eog()` procedure was used to calculate correlations between ICA components and EOG signals filtered between 1 and 10 Hz. Candidate components were identified using an iterative z-score procedure with a threshold of 3.

Automatic component selection is not treated as sufficient on its own. Candidate components are also inspected from their scalp topography and temporal activity because a high EOG correlation does not necessarily imply that a component should be removed. This visual validation helps avoid suppressing components containing meaningful neural activity.

After the artifactual components have been selected, the ICA solution estimated from the 1-Hz data is applied to the EEG epochs prepared for the final analyses, which retain the 0.1–40 Hz frequency range. The important distinction illustrated in this notebook is therefore between the data used to *estimate* ICA and the data to which the resulting correction is ultimately *applied*.

---

### 3. Baseline correction and residual artifact detection

`3-EEG_tutorial_artifact_detection.ipynb`

ICA primarily targets structured artifacts such as blinks and eye movements, but it does not guarantee that every EEG epoch is suitable for analysis. Large transient artifacts can remain because of movements, electrode instability, muscle activity, or other sources of noise. The third notebook therefore evaluates the reconstructed EEG at the single-trial level and identifies epochs that should be excluded from subsequent analyses.

Before evaluating residual artifacts, the EEG signal is baseline corrected. In the example corresponding to the original study, the mean signal during the 100 ms preceding stimulus onset is subtracted from the entire epoch. This operation expresses subsequent voltage changes relative to a common prestimulus period and removes constant trial-wise offsets.

Residual artifacts are then detected using an amplitude criterion applied to the scalp electrodes. In the study from which this tutorial is derived, an epoch was rejected when the absolute EEG amplitude exceeded 150 µV on at least one scalp electrode between the beginning of the analyzed trial and the behavioral response.

The notebook is intended not only to apply this threshold but also to visualize what it detects. Examining rejected and retained epochs is useful for understanding the trade-off involved in any artifact-rejection criterion: a very permissive threshold may retain strongly contaminated trials, whereas an excessively conservative threshold can remove valid neural data.

The output of this step is therefore a set of cleaned epochs together with a reproducible trial-selection mask. Maintaining this trial correspondence is particularly important when EEG data are subsequently combined with behavioral or EMG measurements from the same trials.

---

### 4. Current source density transformation

`4-EEG_tutorial_CSD.ipynb`

The fourth notebook introduces the current source density (CSD), or surface-Laplacian, transformation. EEG potentials measured at the scalp are spatially blurred because electrical fields propagate through the tissues between neural generators and the recording electrodes. In addition, conventional voltage measurements necessarily depend on the chosen reference electrode or reference scheme.

CSD provides a spatial transformation of the scalp potential distribution based on its second spatial derivative. In practical terms, activity that changes locally across neighboring electrodes is emphasized, whereas broadly distributed activity is attenuated. The resulting signal is therefore less dependent on the EEG reference and less spatially diffuse than the original scalp potentials.

In this workflow, the transformation is applied after ICA correction and residual artifact rejection so that large non-neural artifacts are not unnecessarily accentuated by the spatial derivative. The implementation uses the spherical-spline CSD procedure available in MNE-Python. In the original analysis, the default MNE parameters were used, corresponding to a spline stiffness of 4 and a regularization parameter λ of 10⁻⁵.

The notebook compares scalp distributions before and after the transformation to make its effect visually explicit. The important point is that CSD improves the spatial specificity of scalp-recorded activity but should not be interpreted as a cortical source-localization procedure. It remains a transformation of measurements obtained at the scalp.

The CSD-transformed epochs constitute the main input for the lateralized motor-preparation analyses used later in the tutorial.

---

### 5. Segmentation relative to different trial events

`5-EEG_tutorial_periods_segmentation.ipynb`

The EEG data are initially stored as long epochs aligned to stimulus onset, but many questions cannot be addressed optimally from a single temporal reference point. In a delayed-response task, stimulus onset, stimulus offset, response-signal onset, and the behavioral response correspond to different processing transitions, and their absolute timing can vary across trials or conditions.

The fifth notebook therefore shows how the already preprocessed single-trial data can be reorganized relative to these different events without repeating the complete EEG preprocessing pipeline. Event times are expressed within each original trial, and appropriate portions of the cleaned signal are extracted to construct stimulus-locked, stimulus-offset-locked, response-signal-locked, and response-locked representations.

This approach is particularly useful for distinguishing processes associated with different stages of the trial. Stimulus-locked activity can be used to examine the development of neural signals while sensory evidence is available. Stimulus-offset alignment provides a common reference for the delay during which the stimulus is absent. Response-signal alignment isolates activity associated with the transition from maintained preparation to permitted action, whereas response alignment reveals the activity immediately preceding movement execution.

In the associated study, the same four temporal reference points were also used for the statistical analyses of EEG and EMG measures. This common organization facilitates comparisons between cortical activity, partial muscle activations, and behavior across the decision-to-action sequence.

---

### 6. Time-domain motor preparation: the lateralized readiness potential

The cleaned and CSD-transformed EEG can first be examined directly in the time domain. The associated study focused on the lateralized readiness potential (LRP), a classic measure of effector-selective preparation for unilateral manual responses.

For each trial, the responding hand determines which central electrode is contralateral and which is ipsilateral to the selected action. In the present example, analyses are centered on C3 and C4. Lateralized activity is obtained by subtracting the signal recorded ipsilateral to the responding hand from that recorded contralaterally. With this convention, increasingly negative values indicate stronger preparation of the selected response.

C3 and C4 were selected because the response-locked scalp distribution, collapsed across experimental conditions, showed the expected lateralized motor pattern around these sites. In the original study, topographies were evaluated over the 200 ms preceding the response before these electrodes were used for the main analyses.

Because the LRP is computed directly from the time-domain signal, it retains the high temporal resolution of the EEG. It is therefore particularly useful for examining how late-stage effector-selective preparation develops as the response signal approaches and after the response becomes permitted.

The same event-locking procedures introduced in the previous notebook can be used to examine LRP dynamics during stimulus processing, the delay, the response-signal period, and the final interval preceding the response.

---

### 7. Time-frequency analysis and mu–beta lateralization

`6-EEG_tutorial_TFA.ipynb`

The final notebook moves from voltage changes in the time domain to the spectral content of the EEG signal. Neural oscillations are not stationary over an experimental trial: their amplitude can vary when a stimulus appears, while a response is prepared, and around movement execution. Time-frequency analysis is therefore required to estimate how spectral activity evolves over time.

The tutorial uses the short-time Fourier transform (STFT). The principle is to extract a short portion of the EEG signal, multiply it by a temporal window, compute its Fourier spectrum, and then repeat the operation after moving the window forward in time. Repeating this procedure across the epoch produces an estimate of spectral amplitude as a joint function of frequency and time.

A central part of the tutorial illustrates the trade-off between temporal and frequency resolution. Short windows provide more precise temporal localization but coarser frequency sampling, whereas longer windows distinguish nearby frequencies more accurately at the cost of averaging activity over a longer period. In the associated study, a 300-ms Hamming window moved in 100-ms steps provided the compromise used for the final analyses.

Spectral amplitudes were then averaged across the 8–30 Hz range to quantify activity in the combined mu–beta band. Manual action preparation is typically accompanied by a reduction of mu–beta activity over motor cortical regions, with a stronger decrease over the hemisphere contralateral to the prepared hand.

Mu–beta lateralization (MBL) is therefore calculated using the same response-relative C3/C4 organization as the LRP: spectral amplitude ipsilateral to the responding hand is subtracted from contralateral amplitude. Negative values consequently indicate a stronger contralateral reduction in mu–beta activity and thus stronger effector-selective motor preparation.

The LRP and MBL provide complementary views of motor preparation. The LRP describes lateralized activity directly in the time domain, whereas MBL quantifies lateralized changes in oscillatory amplitude. In the associated study, these two measures exhibited partly distinct temporal dynamics, illustrating why time-domain and time-frequency analyses should not automatically be treated as interchangeable measures of the same preparatory process.


## Example dataset

A reduced example dataset accompanies the tutorial so that the complete workflow can be followed without downloading the full experimental dataset.

The example is intended to demonstrate how the processing steps operate, how intermediate results can be inspected, and how the code can be adapted to another experiment. It is not intended to reproduce the group-level statistical results of the associated study.

## Main dependencies

The notebooks are written in Python and rely primarily on MNE-Python for electrophysiological data structures and signal processing, together with NumPy, SciPy, Pandas, and Matplotlib for numerical operations, data handling, and visualization. Jupyter Notebook or JupyterLab is required to run the tutorial interactively.

## Related EMG tutorial

A companion repository describes the processing of the surface electromyographic recordings used in the same research program, including bipolar EMG construction, filtering, automatic muscle-activation detection, integrated-profile onset/offset estimation, and manual validation:

[https://github.com/JohanACHARD/EMG_preprocessing_onset_detection](https://github.com/JohanACHARD/EMG_preprocessing_onset_detection)

## References

Achard, J. A., Gajdos Preuss, T., & Servant, M. *Under review*

Allen, J. B., & Rabiner, L. R. (1977). A unified approach to short-time Fourier analysis and synthesis. *Proceedings of the IEEE, 65*(11), 1558–1564. https://doi.org/10.1109/PROC.1977.10770

Carvalhaes, C., & de Barros, J. A. (2015). The surface Laplacian technique in EEG: Theory and methods. *International Journal of Psychophysiology, 97*(3), 174–188. https://doi.org/10.1016/j.ijpsycho.2015.04.023

Chaumon, M., Bishop, D. V. M., & Busch, N. A. (2015). A practical guide to the selection of independent components of the electroencephalogram for artifact correction. *Journal of Neuroscience Methods, 250*, 47–63. https://doi.org/10.1016/j.jneumeth.2015.02.025

Dimigen, O. (2020). Optimizing the ICA-based removal of ocular EEG artifacts from free viewing experiments. *NeuroImage, 207*, 116117. https://doi.org/10.1016/j.neuroimage.2019.116117

Gramfort, A., Luessi, M., Larson, E., Engemann, D. A., Strohmeier, D., Brodbeck, C., Goj, R., Jas, M., Brooks, T., Parkkonen, L., & Hämäläinen, M. (2013). MEG and EEG data analysis with MNE-Python. *Frontiers in Neuroscience, 7*, 267. https://doi.org/10.3389/fnins.2013.00267

Hyvarinen, A. (1999). Fast and robust fixed-point algorithms for independent component analysis. *IEEE Transactions on Neural Networks, 10*(3), 626–634. https://doi.org/10.1109/72.761722

Jung, T.-P., Makeig, S., Humphries, C., Lee, T.-W., McKeown, M. J., Iragui, V., & Sejnowski, T. J. (2000). Removing electroencephalographic artifacts by blind source separation. *Psychophysiology, 37*(2), 163–178. https://doi.org/10.1111/1469-8986.3720163

Kayser, J., & Tenke, C. E. (2015). Issues and considerations for using the scalp surface Laplacian in EEG/ERP research: A tutorial review. *International Journal of Psychophysiology, 97*(3), 189–209. https://doi.org/10.1016/j.ijpsycho.2015.04.012

Kelly, S. P., & O’Connell, R. G. (2013). Internal and external influences on the rate of sensory evidence accumulation in the human brain. *Journal of Neuroscience, 33*(50), 19434–19441. https://doi.org/10.1523/JNEUROSCI.3355-13.2013

Klug, M., & Gramann, K. (2021). Identifying key factors for improving ICA-based decomposition of EEG data in mobile and stationary experiments. *European Journal of Neuroscience, 54*(12), 8406–8420. https://doi.org/10.1111/ejn.14992

Luck, S. J. (2014). *An Introduction to the Event-Related Potential Technique* (2nd ed.). MIT Press.

Perrin, F., Pernier, J., Bertrand, O., & Echallier, J. F. (1989). Spherical splines for scalp potential and current density mapping. *Electroencephalography and Clinical Neurophysiology, 72*(2), 184–187. https://doi.org/10.1016/0013-4694(89)90180-6

Pfurtscheller, G., & Lopes da Silva, F. H. (1999). Event-related EEG/MEG synchronization and desynchronization: Basic principles. *Clinical Neurophysiology, 110*(11), 1842–1857. https://doi.org/10.1016/S1388-2457(99)00141-8

Rogge, J., Jocham, G., & Ullsperger, M. (2022). Motor cortical signals reflecting decision making and action preparation. *NeuroImage, 263*, 119667. https://doi.org/10.1016/j.neuroimage.2022.119667

Twomey, D. M., Kelly, S. P., & O’Connell, R. G. (2016). Abstract and effector-selective decision signals exhibit qualitatively distinct dynamics before delayed perceptual reports. *The Journal of Neuroscience, 36*(28), 7346–7352. https://doi.org/10.1523/JNEUROSCI.4162-15.2016

Winkler, I., Debener, S., Müller, K.-R., & Tangermann, M. (2015). On the influence of high-pass filtering on ICA-based artifact reduction in EEG-ERP. *2015 37th Annual International Conference of the IEEE Engineering in Medicine and Biology Society (EMBC)*, 4101–4105. https://doi.org/10.1109/EMBC.2015.7319296
