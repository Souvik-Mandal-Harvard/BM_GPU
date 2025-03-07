# ES_GPU
### Version 1.0.0

---
# Overview
**Ethoscope is a "free for academic use" software that extracts behaviors from positional bodypoints, derived from pose estimator software (like [DeepLabCut](http://www.mackenziemathislab.org/deeplabcut), [DeepPoseKit](https://github.com/jgraving/DeepPoseKit), [SLEAP](https://sleap.ai/) etc.), using unsupervised models.** 

Once Ethoscope is installed on the computer (please see the **Getting Started** section for the process) and working directories/paths are properly set, the next workflow is largely divided into 5 consecutive steps:

### STEP 1: Editing config.yaml
Each animal system and experimental setup can be unique and defined by some parameters. This file contains such parameters necessary to be defined for the proper functioning of the rest of the workflow; users can change the parameters according to the need. For details of the parameters, please [click here](docs/config.md), or open the config.yaml file and follow the instructions.

<!-- ### STEP 2: data_formatting.py
* config parameter: 
   * DLC
   * SLEAP
* helper function for converting different data format to npy
 Please note, users do not need to edit this file if the primary source of pose data is DLC or SLEAP. If the user uses some other software for body point estimation, then format the data into npy. -->

### STEP 2: Preprocessing
Animals can show same behavior while they have different body orientations. Also, different animals can have different body size, and their body size can appear different to the camera due to their distance from the camera. Due to these two factors, same behavior can be flagged as different by an automated system. This preprocessing step accounts for these variations by centering, rotating and scaling the body poses by transforming the raw data for the next steps.

*Please note that different body orientations can be taken into account **ONLY** for videos taken from top view, and **NOT** for videos taken from side view.*

### STEP 3: Feature extraction
Behavior is defined by combinations of diffrent unique body movements called behavioral syllables. Imagine different body postures and movements as letters - different combinations of which produce different words, or in the context of behavior science, different behavioral syllables. Ethoscope uses both body postures as well as the kinematics of the animal to define behavioral syllables. Postural features are calculated using the eucledian distance and angle between different body points. Next, Ethoscope performs a Morlet-wavelet transformation on the postural data to coupute the kinematics features. This step generates 4 .npy files in total for each video sample - one contaning data of the Eucledian distances between the labeled body points (limbs.npy), one for the angular data (angles.npy), and two for the kinematics power spectrogram (limb_power.npy, angle_power.npy).

### STEP 4: Embedding
**THIS STEP REQUIRES A GPU on a local computer. In future, we will come up with pipeline that can use cloud-GPU computing and/or CPU (much slower).**

Then, Ethoscope uses these multi-dimensional postural and kinematic feature dataset and reduces it to two-dimension first using PCA (to reduce the dimention of kinamatics data) and then UMAP (to reduce the dimension of the kinematic principal components and the postural data). Running this step generates three different files - all_embeddings.npy, all_kinematic_embeddings.npy, and all_postural_embeddings.npy.

By default, these outcome files of this step will be a two-dimentional behavioral space. However, users can change the number of final dimention by editing the parameter "n_components" under UMAP heading in the config.yaml file.


### Step 5: Getting clusters
**This step may require a GPU depending on the clustering model users select**

Taking the low-dimensional embedding of the behavioral space, this step labels each frame as a behavioral syllable using one of the clustering methods (i.e. Watershed, HDBSCAN). These cluster labels serve as the primary syllables, which can then be utilized to create higher order ethograms.

***Although this version (1.0.0) of Ethoscope requires a GPU-enabled computer, we are in the process of integrating the pipeline to utilize cloud GPU like Google CoLab***

---
# Getting Started

## Setup Environment
Ethoscope uses [RapidsAI](https://rapids.ai/), an open-source software libraries with GPU compatibile models. Currently, RapidAI is compatible only with Linux (Ubuntu 16.04 and up) system. The GPU compatible models from RapidsAI significantly reduces the computation time to get the behavioral space compared to other pipelines (that primarily uses CPU or multi-CPU). To create a rapids.ai Docker container, users will first need to install Docker onto their computer. If you look at their [website](https://rapids.ai/start.html), RapidsAI has several prerequisites that must be met so that their program will seamlessly function with your GPU. Therefore, please check to make sure that you have compatible GPUs, CUDA, Nvidia Drivers, and OS system installed onto your computer.

Below, we have provided a shorter, succinct version of getting started with running Rapids Docker environment, but if you would like more details, then please visit their ["Getting Started"](https://rapids.ai/start.html) webpage. We also highly recommend that you visit Docker's main [website](https://www.docker.com/) to gain a better understanding of why we create containers and how it works.

### Step 1: Pull the RapidsAI Image from DockerHub
> RapidsAI Image is located on DockerHub Website, which you can conveniently **pull** from the cloud to your computer using the following command. Please note that this pulls the Linux, CUDA 11.0, Ubuntu 18.04, and Python 3.7 environment that was compatible with our computer. Visit RapidsAI [website](https://rapids.ai/start.html) to get the specific image link you should download for your computer setup.
```sh
docker pull rapidsai/rapidsai:21.06-cuda11.0-runtime-ubuntu18.04-py3.8
```

### Step 2: Create RapidsAI Docker Container
> After you have successfully pulled a copy of RapidsAI image from DockerHub to your computer, you need to unravel it and create a proper container to run Ethoscope. Images are not enough to run a program because it only contains the bare bone information and instructions necessary for Docker to install the proper program into your container. Don't worry about having duplicate software or version conflict while Docker is creating the container because the installed product will only be contained with the container and not your main computer root environment. 

> Change the **<CONTAINER_NAME>** and **<PROJECT_PATH>** depending on what you want to call your container and where your BM_GPU directory path is on your computer respectively. Also make sure to change the RapidsAI image name (rapidsai/rapidsai-core:cuda11.0-runtime-ubuntu18.04-py3.8) to what you pulled from the previous step (you cn copy that name from the last line appears on your Terminal after you run the ```docker pull``` command).

*Note: Be careful when you change the path of BM_GPU directory in the future because the container may not find your project* 

```sh
docker run --gpus all -it -p 8888:8888 -p 8787:8787 -p 8786:8786 \
   --name <CONTAINER_NAME> \
   -v <PROJECT_PATH>:/rapids/notebooks/host \
   -w /rapids/notebooks/host \
    rapidsai/rapidsai-core:cuda11.0-runtime-ubuntu18.04-py3.8
```

### Step 3: Start Your Docker Container
> Once the container is successfully created and the project path is linked to it, you can now start analyzing your data. For that, first you need to enter the container environment by simpally running the following code and replacing *<CONTAINER_NAME>* with whatever name you chose for the container in the previous step.
> If at any moment you want to exit the container and go back to your root environment just run ```docker stop``` or click "Control-C"

```sh
docker start -i <CONTAINER_NAME>
```

### Step 4: Install Other Libraries Into Your Contianer
> Now that you are in your container, please install the following packages into your container. You will need them when you are analyzing your videos later on. You do not have to install these packages every time your start your Docker. The Container will maintain all of the updates and installs you have done even when you close out of it.
* Installing [ffmpeg (linux)](https://linuxize.com/post/how-to-install-ffmpeg-on-ubuntu-18-04/)   
```sh
apt update
```   
```sh
apt install ffmpeg
```
* Installing [scikit-video](http://www.scikit-video.org/stable/)   
```sh
pip install scikit-video
```
* Installing [tables](http://www.pytables.org/)
```sh
pip install tables
```

A big step is done! You can get out of the docker, if you wish. But you will need to restart the docker to run Ethoscope. Run ```docker restart <CONTAINER_NAME>``` on your command line.

## Run Ethoscope

### Step 1: Upload Your Dataset
> Create a folder named *data* inside BM_GPU, and create another folder inside the *data* with a name you prefer (without space); you can name this folder to whatever you would like to call the overall project name or iteration. For instance, let's name it MY_DATA. Upload your bodypoint dataset (like the .h5 files) into the inner-most folder (MY_DATA).


### Step 2: Update *config.yaml*
> The configuration file contains all the necessary information required to be set for extracting the proper features and parameters, and creating a behavioral space based on the bodypoints saved in the MY_DATA folder. This is where you need to specify the **input_data_path** (should be something like data/MY_DATA) and **result_path** (results/MY_DATA), as well as other parameters specific to your experiment like the digital skeleton of the animal and the clustering methods. Please check this [document](docs/config.md) for specific information and instruction for each parameter.

### Step 3: Switch to Terminal
> Open the **Terminal** on the computer and navigate to the the directory where you saved BM_GPU.

### Step 3: Start the docker
> As described before, run the following command.
```sh
docker start -i <CONTAINER_NAME>
```

### Step 4: Set the path
> Enter the BM_GPU directory by simply typying ```cd BM_GPU```

### Step 4: Preprocessing

Change the *config_file_name* to the name of your config file, and run the following command. This step should be complete pretty quickly.
```sh
python preprocessing.py config_file_name.yaml
```
### Step 5: Feature extraction
Run the following command. Again, don't forget to change the name of the config file. This process should take some time.

```sh
python features_gpu.py config_file_name.yaml
```

### Step 6: Embedding
Run the following command after changing the name of the config file. This process also should take some time.
```sh
python embed_gpu.py config_file_name.yaml
```

### Step 7: Getting clusters
Same as before, run the following command after changing the name of the config file. This step can take some substantial time.
```sh
python cluster.py config_file_name.yaml
```

### Step 8: Using the cluster data for further analysis
Check the **Notebook** folder for codes that can be executed using Jupyter notebook for various types of analyses.

---
# For Developers

If you don't want to restart your kernel in jupyterlab to update your functions, use the following python codes so that your function of concern is updated whenever changes are made.   
%load_ext autoreload   
* %autoreload 0 - disables the auto-reloading. This is the default setting.
* %autoreload 1 - it will only auto-reload modules that were imported using the %aimport function (e.g %aimport my_module). It’s a good option if you want to specifically auto-reload only a selected module.
* %autoreload 2 - auto-reload all the modules. Great way to make writing and testing your modules much easier.

### TODO
- [X] TODO 1
- [ ] TODO 2
- [ ] TODO 3
- [ ] TODO 4
- [ ] TODO 5





