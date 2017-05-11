# Parallel KCF Tracking
_A 15-618 Final Project by Ilaï Deutel and Denis Merigoux_

## Summary

We are implementing an optimized object tracker on NVIDIA GPUs using the [KCF algorithm](http://home.isr.uc.pt/~pedromartins/Publications/henriques_eccv2012.pdf). The goal is to perform real-time object tracking with a simple OpenCV interface (using the [OpenCV Tracking API](http://docs.opencv.org/trunk/d9/df8/group__tracking.html)).

## The challenge

The main dependency is temporal: you need to examine the frames in order, each one after the other. However, the algorithm present some points of synchronization that prevent total parallelization. The size of the working data is not that big (we only work one image at a time) but the complexity of the computations in the algorithm is relatively low, so the dominant factor between data access and computation is unclear.

The challenge is then to optimize the parallelization rate of the algorithm, find out the bottlenecks and try to overcome them The goal is to perform the computation in real time with the best framerate possible (with a decent resolution).

## Correctness and performance analysis

We have built a correctness and performance analysis engine, which shows performance per task (averaged over the number of frames) updated in real time, and stop if the parallel solution is not correct. We determine if the solution is correct by comparing the bounding boxes returned by the parallel implementation with those returned by the sequential implementation. They must be *exactly* the same (no approximation).

Example of output for an incorrect implementation:
```
=== Sequential ===
// Performance information for the 189 frames of the video, not shown here
=== Parallel ===
// Performance information for the first 4 frames of the video, not shown here
Correctness failed at frame 3
Bounding box mismatch:
* Sequential: [286 x 720 from (635, 219)]
* Parallel: [286 x 720 from (635, 221)]
```

## Analyzing performance of the sequential implementation

The table below gives the time, averaged over the number of frames after the first frame (which serves as initialization), taken by the baseline sequential algorithm for each subtask:

| Phase | Subtask | Time taken |
|--------------------------|-------------------------------------|-----------------|
| Detection | Extract and pre-process the patch | 1.028 ms |
|  | Non-compressed custom descriptors | 0.204 ms |
|  | Compressed descriptors | 7.796 ms |
|  | Compressed custom descritors | 2.821 ms |
|  | Compress features and KRSL | 9.986 ms |
|  | Merge all features | 1.595 ms |
|  | **Compute the gaussian kernel** | **21.149 ms** |
|  | Compute the FFT | 1.874 ms |
|  | Calculate filter response | 3.171 ms |
|  | Extract maximum response | 0.248 ms |
|  | *Total* | *49.873 ms* |
| Extracting patches | Update bounding box | 0.000 ms |
|  | Non-compressed descriptors | 1.023 ms |
|  | Non-compressed custom descriptors | 0.199 ms |
|  | Compressed descriptors | 7.397 ms |
|  | Compressed custom descriptors | 3.009 ms |
|  | Update training data | 3.153 ms |
|  | *Total* | *14.666 ms* |
| Feature compression | **Update projection matrix** | **20.677 ms** |
|  | Compress | 4.404 ms |
|  | Merge all features | 0.717 ms |
|  | *Total* | *25.623 ms* |
| Least Squares Regression | Initialization | 0.000 ms |
|  | **Calculate alphas** | **19.000 ms** |
|  | Compute FFT | 1.847 ms |
|  | Add a small value | 0.380 ms |
|  | New Alphaf | 1.135 ms |
|  | Update RLS model | 0.923 ms |
|  | *Total* | *23.115 ms* |
|  | ***Total time for a frame*** | ***114.811 ms*** |

## Preliminary results

Given the timing results on the sequential algorithm, we have decided to start improving the performance of [`denseGaussKernel`](https://github.com/denismerigoux/GPU-tracking/blob/master/src/trackerKCF.cpp), which is called in both the _Compute the gaussian kernel_ and _Calculate alphas_ phases.

This function uses calls to [Fast Fourier Transform](https://en.wikipedia.org/wiki/Fast_Fourier_transform) and inverse FFT, therefore we have decided to use [the NVIDIA CUDA Fast Fourier Transform library (cuFFT)](https://developer.nvidia.com/cufft).

We have also implemented a parallel version of `updateProjectionMatrix`, which involves a very large matrix multiplication.

We have written custom kernels for function `extractCN`, which extracts color names for patches.

Our parallel implementation yield the following results:

| Phase | Subtask | Time taken |
|--------------------------|-------------------------------------|-----------------|
| Detection | Extract and pre-process the patch | 1.049 ms |
|  | Non-compressed custom descriptors | 0.207 ms |
|  | **Compressed descriptors** | **3.129 ms** |
|  | Compressed custom descritors | 2.887 ms |
|  | Compress features and KRSL | 9.741 ms |
|  | Merge all features | 1.618 ms |
|  | **Compute the gaussian kernel** | **11.072 ms** |
|  | Compute the FFT | 1.934 ms |
|  | Calculate filter response | 3.248 ms |
|  | Extract maximum response | 0.252 ms |
|  | *Total* | *35.135 ms* |
| Extracting patches | Update bounding box | 0.000 ms |
|  | Non-compressed descriptors | 1.001 ms |
|  | Non-compressed custom descriptors | 0.198 ms |
|  | **Compressed descriptors** | **3.125 ms** |
|  | Compressed custom descriptors | 3.578 ms |
|  | Update training data | 3.102 ms |
|  | *Total* | *10.909 ms* |
| Feature compression | **Update projection matrix** | **14.010 ms** |
|  | Compress | 4.465 ms |
|  | Merge all features | 0.719 ms |
|  | *Total* | *17.932 ms* |
| Least Squares Regression | Initialization | 0.000 ms |
|  | **Calculate alphas** | **12.068 ms** |
|  | Compute FFT | 1.924 ms |
|  | Add a small value | 0.400 ms |
|  | New Alphaf | 1.168 ms |
|  | Update RLS model | 0.864 ms |
|  | *Total* | *15.034 ms* |
| Total time for a frame | ****** | ***80.548 ms*** |

Speedup: **x1.42**.

## Ressources

We have used the [sequential KCF OpenCV implementation](http://docs.opencv.org/trunk/d2/dff/classcv_1_1TrackerKCF.html) as a starting point. We are using the GPUs of the GHC machines to run our program.


## Deliverables
* [Project proposal](https://github.com/denismerigoux/GPU-tracking/raw/master/proposal/proposal.pdf)
* [Checkpoint Writeup](https://github.com/denismerigoux/GPU-tracking/raw/master/checkpoint/checkpoint.pdf)