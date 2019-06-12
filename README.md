# Variational Inference Models: A study
This repository contains the code used to train and evaluate a framework that can predict steering angle from single images, together with epistemic, aleatoric and total variance.

## Introduction
The project is centered around the problem of probabilistic modelling for the output of neural networks. We investigated many possibilities to make this, evaluating their pro/cons and getting an idea on how to improve current methods. With a particular focus on [Lightweight Probabilistic Deep Networks](https://arxiv.org/pdf/1805.11327.pdf), a lightweight method for estimating aleatoric variance, we decided to prove how, in many real-life applications, model variance plays a relevant role in variance estimation.  This is in contrast with Gast and Roth claim. They indeed stated that, with a sufficient amount of training data, epistemic (model) variance can be explained away. Hence, they developed an approach that estimates aleatoric variance with a single forward pass. In this repository we provide the implementation of a framework that trains a CNN model and evaluates all the variance components, showing that the order of magnitude of epistemic uncertainty is comparable to aleatoric uncertainty one. 

### Models
We took inspiration from [DroNet:Learning to Fly by Driving](https://github.com/uzh-rpg/rpg_public_dronet) for our CNN model. DroNet has been designed as a forked CNN that predicts, from a single 200×200 frame in gray-scale, a steering angle and a collision probability. The shared part of the architecture consists of a fast ResNet-8 with 3 residual blocks, followed by dropout and ReLU non-linearity. After them, the network branches into two separated fully-connected layers, one to carry out steering prediction, and the other one to infer collision probability. For our analysis, we decided to reproduce in PyTorch only the steering angle prediction branch, introducing Dropout layers after every Conv2d layer in order to estimate model variance with Monte Carlo Dropout. We called the resulting model resnet8_MCDO. See ```src/model_zoo/models_resnet8.py``` for more details.

The same CNN architecture is here used for aleatoric or total variance estimation, with little modifications to model's layers to enable them to propagate mean and variance of their outputs. See ```src/model_zoo/resnet8_MCDO_adf.py``` and ```src/contrib/adf.py``` for more details.

### Data
In order to learn steering angles, the publicly available [Udacity dataset](https://github.com/udacity/self-driving-car/tree/master/datasets/CH2) has been used. It provides several hours of video recorded from a car. 

## Running the code

### Software requirements
This code has been tested on Ubuntu 18.04, and on Python 3.7.

Dependencies:
* PyTorch 1.1
* NumPy numpy 1.16.3 
* OpenCV 4.1.0


### Data preparation

#### Steering data
Once Udacity dataset is downloaded, extract the contents from each rosbag/experiment. To have instructions on how to do it, please follow the instructions from the [udacity-driving-reader](https://github.com/rwightman/udacity-driving-reader), to dump images + CSV. In alternative, you can use the script [bagdump.py](script_bagdump/bagdump.py) to accomplish this step. 
After extraction, you will only need ```center/``` folder and ```interpolated.csv``` file from each experiment to create the steeering dataset.
To prepare the data in the format required by our implementation, follow these steps:

1. Process the ```interpolated.csv``` file to extract data corresponding only to the central camera. It is important to sync images and the corresponding steerings matching their timestamps. You can use the script [time_stamp_matching.py](preprocessing/time_stamp_matching.py) to accomplish this step. 

The final structure of the steering dataset should look like this:
```
data
    training/
        HMB_1_3900/*
            center/
            sync_steering.txt
        HMB_2/
        HMB_4/
        HMB_5/
        HMB_6/
    validation/
        HMB_1_501/*
    testing/
        HMB_3/
```
*Since Udacity dataset does not provide validation experiments, we split HMB_1 so that the first 3900 samples are used for training and the rest for validation.


### Training resnet8_MCDO
Train resnet8_MCDO from scratch:
```
cd ./src
python train.py [args]
```
Use ```[args]``` to set batch size, number of epochs, dataset directories, etc. Check ```opts.py``` to see the description of each arg, and the default values we used for our experiments.

Example:
```
cd ./src
python train.py --experiment_rootdir='./exp/' --train_dir='../data/training' --val_dir='../data/validation' --batch_size=16 --epochs=150
```

### Evaluating resnet8_MCDO
In this repository you can find two evaluation scripts:

1) Use [evaluation.py](src/evaluation.py) to evaluate our model on the testing data from each dataset. If is_MCDO=True, the script writes also a file including estimates of model variance for each input sample. When is_MCDO=True, you can change the number of Monte Carlo samples to take by setting T.
```
cd ./src
python evaluation.py [args]
```
Example:
```
cd ./src
python evaluation.py --experiment_rootdir='./exp' --test_dir='../testing' --is_MCDO=True --T=10 
```
2) Use [evaluation_adf.py](src/evaluation_adf.py) to evaluate our model on the testing data from each dataset. If is_MCDO=False, the script writes a file including estimates of aleatoric variance for each input sample. If is_MCDO=True, the script writes a file including estimates of total variance for each input sample. When is_MCDO=True, you can change the number of Monte Carlo samples to take by setting T.
```
cd ./src
python evaluation_adf.py [args]
```
Example:
```
cd ./src
python evaluation_adf.py --experiment_rootdir='./exp' --test_dir='../testing' --is_MCDO=True --T=10 
```

### Comparing MCDO and ADF performaces
In this repository you can find three comparison scripts:

1) Use [compare_stats.py](src/compare_stats.py) to evaluate performances (RMSE and EVA) of MCDO on resnet8_MCDO at varying T on the testing data from each dataset. Disregarding of how you set is_MCDO(=True or False), the code will collect statistics on RMSE and EVA for each number of samples T from 0 (is_MCDO=False) to T = FLAGS.T. Thus, you can change the maximum number of Monte Carlo samples to take by setting T. The script will also show samples of images with low/high epistemic and total uncertainty.
```
cd ./src
python compare_stats.py [args]
```
Example:
```
cd ./src
python compare_stats.py --experiment_rootdir='./exp' --test_dir='../testing' --is_MCDO=True --T=10 
```
2) Use [compare_var.py](src/compare_var.py) to evaluate variance estimates taken with MCDO on resnet8_MCDO (epistemic uncertainty), ADF on resnet8_ADF (aleatoric uncertainty) and MCDO on resnet8_ADF (total uncertainty) at varying T on the testing data from each dataset. Disregarding of how you set is_MCDO(=True or False), the code will collect statistics on variances from T = FLAGS.T forward passes. Thus, you can change the maximum number of Monte Carlo samples to take by setting T.
```
cd ./src
python compare_var.py [args]
```
Example:
```
cd ./src
python compare_var.py --experiment_rootdir='./exp' --test_dir='../testing' --is_MCDO=True --T=10 
```
3) Use [compare_attacks.py](src/compare_attacks.py) to evaluate uncertainty estimates (epistemic and total) under adversarial attacks. Disregarding of how you set is_MCDO(=True or False), the code will collect statistics on epistemic and total uncertainty from T = FLAGS.T forward passes. Thus, you can change the maximum number of Monte Carlo samples to take by setting T. The script will also show samples of images and corresponding adversarial images with respective low/high epistemic, aleatoric and total uncertainty.
```
cd ./src
python compare_attacks.py [args]
```
Example:
```
cd ./src
python compare_attacks.py --experiment_rootdir='./exp' --test_dir='../testing' --is_MCDO=True --T=10 --gen_adv_key='low_var'
```


#### Acknowledgements

This code follows the Keras implementation of [DroNet:Learning to Fly by Driving](https://github.com/uzh-rpg/rpg_public_dronet) for the CNN architecture, with minor changes to adapt the CNN for the study of model variance.

```
@article{Loquercio_2018,
	doi = {10.1109/lra.2018.2795643},
	year = 2018,
	author = {Antonio Loquercio and Ana Isabel Maqueda and Carlos R. Del Blanco and Davide Scaramuzza},
	title = {Dronet: Learning to Fly by Driving},
	journal = {{IEEE} Robotics and Automation Letters}
}
```
This code follows ADF implementation from the paper [Lightweight Probabilistic Deep Networks](https://arxiv.org/pdf/1805.11327.pdf), introducing the probabilistic version of more PyTorch layers.

```
@inproceedings{Gast:2018:LPD,
  title={Lightweight Probabilistic Deep Networks},
  author={Jochen Gast and Stefan Roth},
  booktitle={Proceedings of the IEEE Conference on Computer Vision and Pattern Recognition},
  year={2018}
}
```
