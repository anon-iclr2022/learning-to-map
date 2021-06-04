## Learning to Map for Active Semantic Goal Navigation
Anonymized code for NeurIPS 2021 submission.

### Dependencies
```
pip install -r requirements.txt
```
[Habitat-lab](https://github.com/facebookresearch/habitat-lab) and [habitat-sim](https://github.com/facebookresearch/habitat-sim) need to be installed before using our code. We build our method on the latest stable versions for both, so use `git checkout tags/v0.1.7` before installation. Follow the instructions in their corresponding repositories to install them on your system. Note that our code expects that habitat-sim is installed with the flag `--with-cuda`. 


### Data
We use the Matterport3D (MP3D) dataset (the habitat subset and not the entire Matterport3D) for our experiments. Follow the instructions in the [habitat-lab](https://github.com/facebookresearch/habitat-lab) repository regarding downloading the data and the dataset folder structure. In addition we provide the following:

- [MP3D Scene Pclouds](): An .npz file for each scene that we generated and that contains the 3D point cloud with semantic category labels (40 MP3D categories). This was done for our convenience because the semantic.ply files for each scene provided with the dataset contain instance labels. The folder containing the .npz files should be under `/data/scene_datasets/mp3d`.
- [Training Episode Data](): The set of training examples that we generated for the semantic map predictor pre-training and active training (as described in section 4.1 of the paper). The folders `mp3d_objnav_episodes_final` and `mp3d_objnav_episodes_final_imgSegmOut` are used for pre-training. All folders should be under `/data/scene_datasets/mp3d`.
- [Test Episodes](): The test episodes we generated to evaluate our method. We provide the easy `v3` and hard `v5` sets as described in section 4 of the supplementary material. Note that for the final evaluation we used half of the `v3` set (25 episodes per object instead of the 50 available) and the entirety of `v5`. These should be under `/data/datasets/objectnav/mp3d`.


### Trained Models
We provide the trained map predictor ensembles L2M-Active [here]() and L2M-Offline [here](), and the trained image segmentation model [here]().


### Instructions
Here we provide instructions on how to use our code. It is advised to set up the root_path (directory that includes habitat-lab), log_dir, and paths to data folders and models before-hand in the `train_options.py`.

#### Testing on our episodes for object-goal navigation
To run an object-goal navigation evaluation of our method on two scenes in parallel and on the hard episodes:
```
python main.py --name test_exp --ensemble_dir /path/to/ensemble/folder --with_img_segm --img_segm_model_dir /path/to/img/segm/model/folder --root_path /home/user/ --log_dir /path/to/log/dir --gpu_capacity 2 --ensemble_size 4 --scenes_list 2azQ1b91cZZ 8194nk5LbLH --test_set v5
```

#### Testing the map predictors
To evaluate the map predictors in terms of semantic prediction:
```
python main.py --name test_map_exp --ensemble_dir /path/to/ensemble/folder --root_path /home/user/ --log_dir /path/to/log/dir --sem_map_test --with_img_segm --img_segm_model_dir /path/to/img/segm/model/folder
```

#### Training the map predictor models
If you wish to train your own ensemble, first make sure all data are downloaded, and then each model in the ensemble can be trained separately:
```
python main.py --name train_model_0 --batch_size 12 --map_loss_scale 1 --is_train --num_workers 4 --root_path /home/user/ --log_dir /path/to/log/dir --with_img_segm --img_segm_model_dir /path/to/img/segm/model/folder --stored_episodes_dir /path/to/mp3d_objnav_episodes_final/ --stored_imgSegm_episodes_dir /path/to/mp3d_objnav_episodes_final_imgSegmOut/
```
#### Training the image segmentation model
If you wish to re-train the image segmentation:
```
python main.py --name img_segm_train --num_workers 4 --batch_size 5 --img_segm_loss_scale 1 --is_train --img_segm_training --root_path /home/user/ --log_dir /path/to/log/dir --stored_episodes_dir /path/to/mp3d_objnav_episodes_final/
```

#### Finetuning the map predictors with active data
To finetune the L2M-Offline ensemble models (choose the particular model with --model_number) with the active data:
```
python main.py --name finetune_ensemble_model_0 --num_workers 4 --batch_size 6 --map_loss_scale 0.2 --summary_steps 1000 --image_summary_steps 5000 --checkpoint_steps 4000 --is_train --finetune --with_img_segm --img_segm_model_dir /path/to/img/segm/model/folder --model_number 1 --stored_episodes_dir /path/to/mp3d_objnav_episodes_active/ --ensemble_dir /path/to/ensemble/folder --root_path /home/user/ --log_dir /path/to/log/dir --lr 0.0001
```

#### Generating training data
If you do not want to use our provided training data then you need to generate first the initial training examples (mp3d_objnav_episodes_final), then the grounded image segmentations (mp3d_objnav_episodes_final_imgSegmOut), and finally the active data (mp3d_objnav_episodes_active).

First, to generate the initial training examples on two scenes in parallel:
```
python store_episodes_parallel.py --gpu_capacity 2 --scenes_list HxpKQynjfin JF19kD82Mey --episodes_save_dir /path/to/save/dir/ --root_path /home/user/ --max_num_episodes 20000
```
Note that there is a dedicated options list in `store_episodes_parallel.py`.

To generate the grounded image segmentations on two scenes in parallel:
```
python store_img_segm_ep.py --gpu_capacity 2 --img_segm_model_dir /path/to/img/segm/model/folder --episodes_save_dir /path/to/save/dir/ --stored_episodes_dir /path/to/initial/training/examples/ --root_path /home/user/ --scenes_list 7y3sRwLe3Va GdvgFV5R1Z5
```
This step reads from the initial training examples and runs the image segmentation model to store its output. Note that the image segmentation model needs to be trained first.

To generate active data on two scenes in parallel:
```
python main.py --name active_train_data --is_train --active_training --ensemble_size 4 --ensemble_dir /path/to/ensemble/folder --gpu_capacity 2 --with_img_segm --img_segm_model_dir /path/to/img/segm/model/folder --active_ep_save_dir /path/to/save/dir/ --max_num_episodes 10000 --root_path /home/user/ --log_dir /path/to/log/dir --scenes_list 17DRP5sb8fy 1LXtFkjw3qL
```
Note that to generate the active data an ensemble of map predictors needs to be trained first.
