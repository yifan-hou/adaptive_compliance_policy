# adaptive_compliance_policy
[[Project Page]](https://adaptive-compliance.github.io/)[[Paper]](https://arxiv.org/abs/2410.09309)

Code and guide for training and evaluation of the Adaptive Compliance Policy in real world. This readme also describes how to setup our data collection and robot compliance controller pipeline.

This repo is built on top of [universal_manipulation_interface](https://github.com/real-stanford/universal_manipulation_interface). 

## How to use this repo
This document describes how to setup our whole software system, including python code for training, modeling and c++ code for control. You may not need all of them. Below lists what is needed for a few different use cases. 

### *Just checkout how ACP is implemented*
You can start with reading the code below:
* **Compute virtual target & stiffness label:** `adaptive_compliance_policy/PyriteEnvSuites/scripts/postprocess_add_virtual_target_label.py`
* **Reconstruct stiffness matrix from policy outputs:** `adaptive_compliance_policy/PyriteEnvSuites/env_runners/virtual_target_real_env_runner.py`
* **Policy implementation:** `/home/yifanhou/temp/adaptive_compliance_policy/PyriteML/diffusion_policy/policy/diffusion_unet_timm_mod1_policy.py`
### *Train an ACP*
You can [download our dataset](#downloads), then setup this package following the [Install](#install) section, launch training following the [Training](#training-acp) section.
### *Try our compliance controlled data collection pipeline*
Follows the section on [setting up our c++ code base](#setup-robot-controllers), and [data collection](#data-collection). You might also need the [data postprocessing](#data-postprocessing) if you want to train with your data.
### *Try ACP on your own robot*
Hardware information is abstracted away in this package. You may fork our [hardware_interfaces](https://github.com/yifan-hou/hardware_interfaces) package, add in your new devices. The interface to hardware is the ManipServer class defined in hardware_interfaces. 


## Downloads
``` sh
wget https://real.stanford.edu/adaptive-compliance/checkpoints.zip # Download all checkpoints
wget https://real.stanford.edu/adaptive-compliance/data/flip_up_230.zip # Download processed dataset for the item flip up task
wget https://real.stanford.edu/adaptive-compliance/data/vase_wiping_200.zip # Download processed dataset for the vase wiping task
```

## Install
The following is tested on Ubuntu 22.04.

1. Install [mamba](https://mamba.readthedocs.io/en/latest/installation/mamba-installation.html)
2. Clone this repo along with its submodule.
``` sh
git clone --recursive git@github.com:yifan-hou/adaptive_compliance_policy.git
```
3. Create a virtual env called `pyrite`:
``` sh
cd adaptive_compliance_policy
mamba env create -f conda_environment.yaml
# after finish, activate it using
mamba activate pyrite
# a few pip installs
pip install v4l2py
pip install toppra
pip install atomics
pip install vit-pytorch # Need at least 1.7.12, which was not available in conda
pip install imagecodecs # Need at least 2023.9.18, which caused lots of conflicts in conda
```
4. Setup environment variables: add the following to your .bashrc or .zshrc, edit according to your local path.
``` sh
# where the collected raw data folders are
export PYRITE_RAW_DATASET_FOLDERS=$HOME/data/real
# where the post-processed data folders are
export PYRITE_DATASET_FOLDERS=$HOME/data/real_processed
# Each training session will create a folder here.
export PYRITE_CHECKPOINT_FOLDERS=$HOME/training_outputs
# Hardware configs.
export PYRITE_HARDWARE_CONFIG_FOLDERS=$HOME/git/RobotTestBench/applications/ur_test_bench/config
# Logging folder.
export PYRITE_CONTROL_LOG_FOLDERS=$HOME/data/control_log
```

## Setup robot controllers
If you want to run our data collection or testing pipeline, you need to setup our c++ robot controllers.
We provide an admittance controller implementation based on our [force_control](https://github.com/yifan-hou/force_control) package.

### Setup conda packages
Make sure the conda packages are visible to c++ linkers. Create a .sh file with the following content:
``` sh
# clib_path_activate.sh
export LD_LIBRARY_PATH=/home/yifanhou/miniforge3/envs/pyrite/lib/:$LD_LIBRARY_PATH
```
at `${CONDA_PREFIX}/etc/conda/activate.d/`, e.g. `$HOME/miniforge3/envs/pyrite/etc/conda/activate.d` if you install miniforge at the default location.

### Installation
Pull the following packages:
``` sh
# https://github.com/yifan-hou/cpplibrary
git clone git@github.com:yifan-hou/cpplibrary.git
# https://github.com/yifan-hou/force_control
git clone git@github.com:yifan-hou/force_control.git
# https://github.com/yifan-hou/hardware_interfaces
git clone git@github.com:yifan-hou/hardware_interfaces.git
```
Then build & install following their readme.

### (Optional) Install to a local path
I recommend to install to a local path for easy maintainence, also you don't need sudo access. To do so, replace the line
``` sh
cmake ..
```
with
``` sh
cmake -DCMAKE_INSTALL_PREFIX=$HOME/.local  ..
```
when building packages above. Here `$HOME/.local` can be replaced with any local path.
Then you need to tell gcc to look for binaries/headers from your local path by adding the following to your .bashrc or .zshrc:
``` sh
export PATH=$HOME/.local/bin:$PATH
export C_INCLUDE_PATH=$HOME/.local/include/:$C_INCLUDE_PATH
export CPLUS_INCLUDE_PATH=$HOME/.local/include/:$CPLUS_INCLUDE_PATH
export LD_LIBRARY_PATH=$HOME/.local/lib/:$LD_LIBRARY_PATH
```
You need to run `source .bashrc` or reopen a terminal for those to take effect.

## Data collection
(Requires robot controller setup)
The data collection pipeline is wrapped in `hardware_interfaces/applications/manipulation_data_collection".
1. Check the correct config file is selected in `hardware_interfaces/applications/manipulation_data_collection/src/main.cc`.
2. Build `hardware_interfaces` follow its readme.
3. On the UR teach pendant, make sure you calibrated the TCP mass. 
4. Edit the config file specified in step 1, make sure you have the correct hardware IP/ID, data saving path, etc.
5. Launch the manipulation_data_collection binary:
``` sh
cd hardware_interfaces/build
./applications/manipulation_data_collection/manipulation_data_collection
```
Then follow the on screen instructions.

Our data collection pipeline saves data episode by episode. The saved data folder looks like this:
```
current_dataset/
    episode_1727294514
    episode_1727294689
    episode_1727308394/
        rgb_0
        rgb_1/
            img_count_timestamp.jpg
            ...
        robot_data_0.json
        wrench_data_0.json
    ...
```
Within an episode, each file/folder corresponds to a device. Every frame of data is saved with a timestamp that was calibrated across all devices. For rgb images, its timestamp is saved in its file name, e.g.
```
img_000695_29345.186724_ms
```
means that this image is the 695th frame saved in this episode, and it is saved at 29345.186724ms since the program launched. 

## Data postprocessing
We postprocess the data to match the data format for training and compute the virtual target labels. We do so in two steps:

1. Convert to zarr format using `adaptive_compliance_policy/PyriteUtility/data_pipeline/real_data_processing.py`.
Specify `id_list`, `input_dir`, `output_dir` then run the script. This script will compress the images into a [zarr](https://zarr.dev/) database, then generate meta data for the whole dataset. This step creates a new folder at `output_dir`.

2. Generate virtual target labels using `adaptive_compliance_policy/PyriteEnvSuites/scripts/postprocess_add_virtual_target_label.py`
Specify `id_list` and `dataset_path`, then run the script. You can set `flag_plot=True` to check 3D visualizations of the reference/virtual target trajectories, episode by episode.
This step does not create a new folder, instead, it only adds new fields to the input dataset.

After finishing both steps, the output folder can be used for training.

## Training ACP
Before launching training, setup accelerator if you haven't done so:
``` sh
accelerate config
```

Then launch training with:
``` sh
# Train ACP with FFT spectrum wrench encoding if you have high frequency wrench data (>5k Hz)
HYDRA_FULL_ERROR=1 accelerate launch train.py --config-name=train_spec_workspace
# Or, train ACP with temporal convolution wrench encoding
HYDRA_FULL_ERROR=1 accelerate launch train.py --config-name=train_conv_workspace
```
Or, train with multiple GPU like this
``` sh
HYDRA_FULL_ERROR=1 accelerate launch --gpu_ids 0,1,2,3 --num_processes=4 train.py --config-name=train_spec_workspace
```

## Evaluation on real robots
After building the `hardware_interfaces` package, a pybind library is generated under `hardware_interfaces/workcell/table_top_manip/python/`. This library contains a c++ multi-thread server that maintains low-latency communication and data/command buffers with all involved hardware. It also maintains an admittance controller. We will launch a python script that communicates with the hardware server, while the python script itself does not need multi-processing.

Before testing, check the following:
1. `pyrite` virtual environment is activated.
2. Env variables `PYRITE_CHECKPOINT_FOLDERS`, `PYRITE_HARDWARE_CONFIG_FOLDERS`, `PYRITE_CONTROL_LOG_FOLDERS` are properly set.
3. You have specified name of the checkpoint folder and the hardware config file in `adaptive_compliance_policy/PyriteEnvSuites/env_runners/virtual_target_real_env_runner.py`.
Then start execution by running `adaptive_compliance_policy/PyriteEnvSuites/env_runners/virtual_target_real_env_runner.py`.

The script will first launch the manpulation server, which initialize all the hardware specified in the hardware config file. A video streaming window will pop up. When the video stream looks good (actual video is streaming, no black screen), press `q` to leave the window and continue the test.

The full stiffness matrix is reconstructed from policy outputs in `virtual_target_real_env_runner.py`.

## Citation
If you find this codebase useful, feel free to cite our paper:
```bibtex
@inproceedings{hou2025adaptive,
  title={Adaptive compliance policy: Learning approximate compliance for diffusion guided control},
  author={Hou, Yifan and Liu, Zeyi and Chi, Cheng and Cousineau, Eric and Kuppuswamy, Naveen and Feng, Siyuan and Burchfiel, Benjamin and Song, Shuran},
  booktitle={2025 IEEE International Conference on Robotics and Automation (ICRA)},
  pages={4829--4836},
  year={2025},
  organization={IEEE}
}
```

## License
This repository is released under the MIT license. 

## Acknowledgement
We would like to thank Mengda Xu, Huy Ha, Zhenjia Xu, for their advice on the development of this codebase.

## Code References
* This repo is built on top of [universal_manipulation_interface](https://github.com/real-stanford/universal_manipulation_interface).
* We use [multimodal_representation](https://github.com/stanford-iprl-lab/multimodal_representation) for the temporal convolution encoding for forces.
* The FFT spectrum force encoding is adapted from [maniwav](https://github.com/real-stanford/maniwav).