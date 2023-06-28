# SceneReplica
Code release for SceneReplica paper: [ArXiv](https://arxiv.org/abs/2306.15620) | [Webpage](https://irvlutd.github.io/SceneReplica/)

## Index

1. [Data Setup](#data-setup): Setup the data files, object models and create a ros workspace with source for the different algorithms
2. [Scene Setup](#reference-scene-setup): Setup the Scene in either simulation or real world
3. [Experiments](#experiments): Run experiments from different algorithms listed 
4. [Misc](#misc): Video recording, ROS workspace setup

# Data Setup

For ease of use, the code assumes that you have the data files (model meshes, generated scenes etc) under a single directory.
As an example it can be something simple like: `~/Datasets/benchmarking/`

```
~/Datasets
   |--benchmarking
      |--gazebo_models_all_simple/
      |--grasp_data
         |--refined_grasps
            |-- fetch_gripper-{object_name}.json
      |--final_scenes
         |--scene_data/
            |-- scene_id_*.pk scene pickle files
         |--metadata/
            |-- meta-00*.mat metadata .mat files
            |-- color-00*.png color images for scene
            |-- depth-00*.png depth images for scene
         |--scene_ids.txt : selected scene ids on each line
```

Follow the steps below, download and extract the zip files:

- Download and extract Graspit generated grasps for YCB models: [`grasp_data.zip`](https://utdallas.box.com/s/jz3fin85v7gyv8ls1tqcs95qoye569g4) as `grasp_data/`
- Download and extract Scenes Data : [`final_scenes.zip`](https://utdallas.box.com/s/47foq3nri7ob3gym853ynwrenfvv6oix) as `final_scenes/`
- [*Optional*] Download and extract YCB models for gazebo (using `textured_simple` meshes): [`gazebo_models_all_simple.zip`](https://utdallas.box.com/v/scenereplica-gazebo-models)
  - Only needed if you want to play around with the YCB models in Gazebo simulation
  - This already includes the edited model for cafe table under the name `cafe_table_org`
  - Create a symlink to the gazebo models into your Fetch Gazebo src/models/: `~/src/fetch_gazebo/fetch_gazebo/models`


# Reference Scene Setup

Scene Idxs: `10, 25, 27, 33, 36, 38, 39, 48, 56, 65, 68, 77, 83, 84, 104, 122, 130, 141, 148, 161`

## Real World Usage

Setup the data folders as described and follow the steps below:

1. Setup the YCB scene in real world as follows. `--datadir` is the dir with `color-*.png, depth-*.png, pose-*.png and *.mat` info files. Using the above Data Setup, it should be the folder: `final_scenes/metadata/`
   ```shell
   cd src
   python setup_robot.py # will raise torso and adjust head camera lookat
   python setup_ycb_scene.py --index [SCENE_IDX] --datadir [PATH TO DIR]
   ```

2.  `setup_ycb_scene.py` starts publishing an overlay image which we can use as a reference for real world object placement

3. Run rviz to visualize the overlay image and adjust objects in real world
    ```Shell
    rosrun rviz rviz -d config/scene_setup.rviz
    ```


## Gazebo (Simulation) Usage

[Optional]

0. Download the Gazebo models of YCB objects as described above

1. launch the tabletop ycb scene with only the robot
    ```Shell
    roslaunch launch/just_robot.launch
    ``` 
  
2. Setup the desired scene in Gazebo:
   ```Shell
   cd src/
   python setup_scene_sim.py
   ```
   Preferred that all data is under `~/Datasets/benchmarking/{scene_dir}/`(e.g. "final_scenes"). 
   It runs in a loop and asks you to enter the scene id at each iteration. Loads
   the objects in gazebo and waits for user confirmation before next scene.

# Experiments

- We use a multi-terminal setup for ease of use (example: [Terminator](https://gnome-terminator.org/))
- In one part, we have the perception repo and run bash scripts to publish the required perception components (e.g., 6D Object Pose or Segmentation Masks)
- We also have a MoveIt motion planning launch file as for the initial experiments, we have only benchmarked against MoveIt
- Lastly, we have a rviz visualization terminal and a terminal to run the main grasping script.
- Ensure proper sourcing of ROS workspace and conda environment activation.

## Model Based Grasping

1. Start the appropriate algorithm from below in a separate terminal and verify if the required topics are being published or not?

2. Once verified, `cd src/` and you can start the grasping script: Run `python bench_model_based_grasping.py`. See its command line args for more info.

- `--pose_method` : From {"gazebo", "posecnn", "poserbpf"}
- `--obj_order` : From {"random", "nearest_first"}
- `--scene_idx` : Scene id for which you'll test the algorithm

### PoseCNN
Reference repo: [`IRVLUTD/PoseCNN-PyTorch-NV-Release` ](https://github.com/IRVLUTD/PoseCNN-PyTorch-NV-Release/tree/fetch_robot)

- `cd PoseCNN-PyTorch-NV-Release`
- `conda activate benchmark`
- term1: `rosrun rviz rviz -d ./ros/posecnn.rviz`
- term2: `./experiments/scripts/ros_ycb_object_test_fetch.sh $GPU_ID`
- More info: check out [README](https://github.com/IRVLUTD/PoseCNN-PyTorch-NV-Release/blob/fetch_robot/README.md)

### PoseRBPF
Reference repo: [`IRVLUTD/posecnn-pytorch`](https://github.com/IRVLUTD/posecnn-pytorch)

- `cd posecnn-pytorch`
- `conda activate benchmark`
- term1: `./experiments/scripts/ros_ycb_object_test_subset_poserbpf_realsense_ycb.sh $GPU_ID $INSTANCE_ID`
- term2: `./experiments/scripts/ros_poserbpf_ycb_object_test_subset_realsense_ycb.sh $GPU_ID $INSTANCE_ID`
- Rviz visualization: `rviz -d ros/posecnn_fetch.rviz`
- Check out [README](https://github.com/IRVLUTD/posecnn-pytorch#running-on-realsense-cameras-or-fetch) for more info


## Model Free Grasping

1. Start the appropriate algorithm from below in a separate terminal and verify if the required topics are being published or not?
   - First run the segmentation script
   - Next run the 6dof grasping script
   - Finally run the grasping pipeline script

2. Once verified, you can start the grasping script: `cd src/` and run `python bench_6dof_segmentation_grasping.py`. See its command line args for more info.
   - `--grasp_method` : From {"graspnet", "contact_gnet"}
   - `--seg_method` : From {"uois", "msmformer"}
   - `--obj_order` : From {"random", "nearest_first"}
   - `--scene_idx` : Scene id for which you'll test the algorithm

### Segmentation Methods

#### UOIS
Reference repo: [`IRVLUTD/UnseenObjectClustering`](https://github.com/IRVLUTD/UnseenObjectClustering)

- `cd UnseenObjectClustering`
- seg code: `./experiments/scripts/ros_seg_rgbd_add_test_segmentation_realsense.sh $GPU_ID`
- rviz: `rosrun rviz rviz -d ./ros/segmentation.rviz`

#### MSMFormer
Reference repo: [`YoungSean/UnseenObjectsWithMeanShift`](https://github.com/YoungSean/UnseenObjectsWithMeanShift)

- `cd UnseenObjectsWithMeanShift`
- seg code: `./experiments/scripts/ros_seg_transformer_test_segmentation_fetch.sh $GPU_ID`
- rviz: `rosrun rviz rviz -d ./ros/segmentation.rviz`

### 6DOF Point Cloud based Methods

#### 6DOF GraspNet
Reference repo: [`IRVLUTD/ pytorch_6dof-graspnet`](https://github.com/IRVLUTD/pytorch_6dof-graspnet)

- `cd pytorch_6dof-graspnet`
- grasping code: `./exp_publish_grasps.sh` (chmod +x if needed)

#### Contact GraspNet
Reference repo: [`IRVLUTD/contact_graspnet`](https://github.com/IRVLUTD/contact_graspnet)

**NOTE:** This has a different conda environment (`contact_graspnet`) than others due to a tensorflow dependency.
Check the `env_cgnet.yml` env file in the reference repo.

- `cd contact_graspnet`
- `conda activate contact_graspnet`
- Run `run_ros_fetch_experiment.sh` in a terminal. 
  - In case GPU usage is too high, (check via `nvidia-smi`): reduce the number for `--forward_passes` flag in shell script
  - If you dont want to see generated grasp viz, remove the `--viz` flag from shell script  


# Misc

## Usage of the Video Recorder    
1. Launch the RealSense camera
    ```Shell
    roslaunch realsense2_camera rs_aligned_depth.launch tf_prefix:=measured/camera
    ``` 
  
2. Run the video recording script: `src/video_recorder.py`
    ```Shell
    cd src/
    python3 video_recorder.py -s SCENE_ID -m METHOD -o ORDER [-f FILENAME_SUFFIX] [-d SAVE_DIR]
    ```
    To stop capture, `Ctrl + C` should as a keyboard interrupt.


## ROS Setup
We used the following steps to setup our ROS workspace. Install the following dependencies into a suitable workspace folder, e.g. `~/bench_ws/`

  **1.** Install ROS Noetic 

     Install using the instructions from here -  http://wiki.ros.org/noetic/Installation/Ubuntu
  
  **2.** Setup ROS workspace
  
     Create and build a ROS workspace using the instructions from here - http://wiki.ros.org/catkin/Tutorials/create_a_workspace
  
  **3.** Install fetch ROS, Gazebo packages and custom scripts
  
  Install packages using following commands inside the src folder 
  
    `git clone -b melodic-devel https://github.com/ZebraDevs/fetch_ros.git`

    `git clone –branch gazebo11 https://github.com/fetchrobotics/fetch_gazebo.git`

    `git clone https://github.com/ros/urdf_tutorial`

    `git clone -b devel https://github.com/IRVLUTD/R-Bench.git`
  
  If you see missing package erros: Use `apt install` for `ros-noetic-robot-controllers` and `ros-noetic-rgbd-launch`.
  
**Gazebo or MoveIt Issues**:

- `alias killgazebo="killall -9 gazebo & killall -9 gzserver  & killall -9 gzclient"` and then use `killgazebo` command to restart
- Similarly for MoveIt, you can use `killall -9 move_group` if `Ctrl-C` on the moveit launch process is unresponsive
