# Deep SORT with openvino

## Introduction

This repository contains code for *Simple Online and Realtime Tracking with a Deep Association Metric* (Deep SORT).
We extend the original [SORT](https://github.com/abewley/sort) algorithm to
integrate appearance information based on a deep appearance descriptor.
See the [arXiv preprint](https://arxiv.org/abs/1703.07402) for more information.

## Dependencies

The code is compatible with Python 2.7 and 3. The following dependencies are
needed to run the tracker:

* NumPy
* sklearn
* OpenCV

Additionally, feature generation requires TensorFlow (>= 1.0).

## Installation

First, clone the repository:
```
git clone https://github.com/r-or/deep_sort.git
```
Then, download pre-generated detections and the CNN checkpoint file from
[here](https://drive.google.com/open?id=18fKzfqnqhqW3s9zwsCbnVJ5XF2JFeqMp).

*NOTE:* The candidate object locations of our pre-generated detections are
taken from the following paper:
```
F. Yu, W. Li, Q. Li, Y. Liu, X. Shi, J. Yan. POI: Multiple Object Tracking with
High Performance Detection and Appearance Feature. In BMTT, SenseTime Group
Limited, 2016.
```
We have replaced the appearance descriptor with a custom deep convolutional
neural network (see below).

## Running the tracker

The following example starts the tracker on one of the
[MOT16 benchmark](https://motchallenge.net/data/MOT16/)
sequences.
We assume resources have been extracted to the repository root directory and
the MOT16 benchmark data is in `./MOT16`:
```
python deep_sort_app.py \
    --sequence_dir=./MOT16/test/MOT16-06 \
    --detection_file=./resources/detections/MOT16_POI_test/MOT16-06.npy \
    --min_confidence=0.3 \
    --nn_budget=100 \
    --display=True
```
Check `python deep_sort_app.py -h` for an overview of available options.
There are also scripts in the repository to visualize results, generate videos,
and evaluate the MOT challenge benchmark.

## Generating detections

### Procedure without Openvino support
Beside the main tracking application, this repository contains a script to
generate features for person re-identification, suitable to compare the visual
appearance of pedestrian bounding boxes using cosine similarity.
The following example generates these features from standard MOT challenge
detections. Again, we assume resources have been extracted to the repository
root directory and MOT16 data is in `./MOT16`:
```
python tools/generate_detections.py \
    --model=resources/networks/mars-small128.pb \
    --mot_dir=./MOT16/train \
    --output_dir=./resources/detections/MOT16_train
```
The model has been generated with TensorFlow 1.5. If you run into
incompatibility, re-export the frozen inference graph to obtain a new
`mars-small128.pb` that is compatible with your version:
```
python tools/freeze_model.py
```
The ``generate_detections.py`` stores for each sequence of the MOT16 dataset
a separate binary file in NumPy native format. Each file contains an array of
shape `Nx138`, where N is the number of detections in the corresponding MOT
sequence. The first 10 columns of this array contain the raw MOT detection
copied over from the input file. The remaining 128 columns store the appearance
descriptor. The files generated by this command can be used as input for the
`deep_sort_app.py`.

**NOTE**: If ``python tools/generate_detections.py`` raises a TensorFlow error,
try passing an absolute path to the ``--model`` argument. This might help in
some cases.

### Notes on Openvino
This fork adds support for Openvino. Obviously this is more useful for online
feature extraction instead of generating detections into a text file for MOT16.

**Openvino installation:** if you are using an offcially not supported (newer) 
version of Ubuntu, e.g. 19.04, the easiest way of installing openvino is just to 
selectively the rpms inside ```<extracted framework>/rpm/``` to ```/``` after you ran through
the normal installation procedure detailed at [openvino installation guide](https://docs.openvinotoolkit.org/latest/_docs_install_guides_installing_openvino_linux.html).
Even if you run the semi-officially supported Ubuntu 18.04 you'll likely run into
issues caused by missing libraries due to bugs in the installation script.
    
A shortcut to get things going: backup your original ```/etc/lsb-release``` and modify
it so it looks like this:
```
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=18.04
DISTRIB_CODENAME=bionic
DISTRIB_DESCRIPTION="Ubuntu 18.04"
```
After performing the installation the original file can be restored.

**Performance estimation:** during the [AI hackathon](http://www.ai-hackathon.com/)
we used this to generate embeddings from a video stream on a NCS2. It was able
to sustain around 5fps while tracking more than 15 targets. Note that no batch
processing is available on this device, so for each target inference must be
called sequentially.

On CPU it runs roughly at 
[three times the speed as vanilla tensorflow](https://gist.github.com/r-or/e1b85c47e1906763b6e0e7a209812dda).
(Intel) GPU unfortunately doesn't work currently.

You can run ```python test_ov.py``` to test the installation and make a 
performance comparison.

For the feature extraction to use Openvino, a few additional steps have to be
taken:

#### 1) Freeze model for Openvino
This is necessary as the default model includes elements which are incompatible
with Openvino:
```
python tools/freeze_model.py --no_preprocess
```

#### 2) Convert model with Model Optimizer
```
cd model_data/networks
mo_tf.py --input_model mars-small128.pb -b 1 --data_type <data_type>
```
As data type you need to use a type which is supported for the device you want
to use. The Movidius NCS2 compute stick for instance needs "FP16", the CPU only
supports the default "FP32".

#### 3) Generate detections
To generate the MOT16 detections in addition you have to supply the Openvino
device (e.g. "CPU" or "MYRIAD" for the NCS2):
```
python tools/generate_detections.py \
    --model=resources/networks/mars-small128.pb \
    --mot_dir=./MOT16/train \
    --output_dir=./resources/detections/MOT16_train \
    --use_openvino=MYRIAD
```


## Training the model

To train the deep association metric model we used a novel [cosine metric learning](https://github.com/nwojke/cosine_metric_learning) approach which is provided as a separate repository.

## Highlevel overview of source files

In the top-level directory are executable scripts to execute, evaluate, and
visualize the tracker. The main entry point is in `deep_sort_app.py`.
This file runs the tracker on a MOTChallenge sequence.

In package `deep_sort` is the main tracking code:

* `detection.py`: Detection base class.
* `kalman_filter.py`: A Kalman filter implementation and concrete
   parametrization for image space filtering.
* `linear_assignment.py`: This module contains code for min cost matching and
   the matching cascade.
* `iou_matching.py`: This module contains the IOU matching metric.
* `nn_matching.py`: A module for a nearest neighbor matching metric.
* `track.py`: The track class contains single-target track data such as Kalman
  state, number of hits, misses, hit streak, associated feature vectors, etc.
* `tracker.py`: This is the multi-target tracker class.

The `deep_sort_app.py` expects detections in a custom format, stored in .npy
files. These can be computed from MOTChallenge detections using
`generate_detections.py`. We also provide
[pre-generated detections](https://drive.google.com/open?id=1VVqtL0klSUvLnmBKS89il1EKC3IxUBVK).

## Citing DeepSORT

If you find this repo useful in your research, please consider citing the following papers:

    @inproceedings{Wojke2017simple,
      title={Simple Online and Realtime Tracking with a Deep Association Metric},
      author={Wojke, Nicolai and Bewley, Alex and Paulus, Dietrich},
      booktitle={2017 IEEE International Conference on Image Processing (ICIP)},
      year={2017},
      pages={3645--3649},
      organization={IEEE},
      doi={10.1109/ICIP.2017.8296962}
    }

    @inproceedings{Wojke2018deep,
      title={Deep Cosine Metric Learning for Person Re-identification},
      author={Wojke, Nicolai and Bewley, Alex},
      booktitle={2018 IEEE Winter Conference on Applications of Computer Vision (WACV)},
      year={2018},
      pages={748--756},
      organization={IEEE},
      doi={10.1109/WACV.2018.00087}
    }
