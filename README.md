# Supervising camera calibration using optimized chessboard poses

## :information_source: Intro

*Camera calibration* is a crucial step in computer vision applications, as it allows to estimate the scene's structure and remove lens distortion. 

The camera is typically modeled with a set of intrinsic (i.e. focal length, principal point and axis skew) and extrinsic parameters (i.e. rotation and translation).
The most common calibration procedure estimates such parameters using known points in the real world and their projection in the image plane (see [References](#references)), presented as a pattern with known geometry, usually a *flat chessboard*. During the calibration, the user has to move the chessboard around in order to cover the whole field of view, including different orientations of the pattern. The calibration result is *highly affected* by such positions and it is not unusual that several trials have to be carried out in order to have a proper calibration result. It is therefore fundamental to place the pattern properly to avoid the final image to be distorted.
 
This repository contains the software required to identify **the best chessboard poses** and to **visually guide the user** to place the chessboard in the selected poses. The set of poses which maximizes the result of the calibration procedure is selected as best set. 

## :robot: Camera calibration on the iCub

[`stereoCalib`](https://robotology.github.io/robotology-documentation/doc/html/group__icub__stereoCalib.html) is the module used on the iCub to perform stereo calibration (both intrinsic and extrinsic parameters). The module saves the final result onto a file called `outputCalib.ini`. 

Typically, such file is manually copied into `icubEyes.ini`, which is in turn loaded by [`camCalib`](https://robotology.github.io/robotology-documentation/doc/html/group__icub__camCalib.html), i.e. the module performing camera undistortion.

Our procedure uses `stereoCalib` to perform the calibration. Then we copy `outputCalib.ini` into the file loaded by `camCalib` to evaluate the quality of the calibration, by assessing how *similar* the undistorted image produced by `camCalib` is to the ideal image.

## :new: The devised procedure

The devised procedure consists mainly of two steps:

- [identifying the **best set** of poses](#identifying-the-best-set-of-poses);
- [use the selected set of poses to **supervise the calibration procedure**](#supervise-the-calibration-procedure).


### Identifying the best set of poses

We devised this part entirely in `Gazebo`, through the following steps: 

- **Simulate distortion and ideal pinhole camera**:

    We simulate both an *ideal* and a *distorted* camera on the robot, in the same position. The ideal camera allows us to have a ground truth image to evaluate the quality of the calibration procedure. 
    The robot model can be found in the [`gazebo/models/icub-distorted-camera` directory](gazebo/models/icub-distorted-camera).
    
    A `8x6` chessboard is then placed in front of the robot. 

- **Generate candidate poses**: 
    
    We move the chessboard in order to cover the whole field of view. More specifically, the FOV is uniformly sampled over a grid, while the chessboard orientations over the three axes change randomly at each point of the grid, with a maximum variation of `20 degrees`. We generate a total of `200` sets of candidate poses. 
    
    This step was performed by running:
     
     `find-best-positions-calibration.sh run camera-calibration-best-pos/rgb-cameras/320x240 200`
    
    :point_up:  **Important note**:
 
    By default, the cameras considered are _rgb_ with a resolution of `320x240 px`. We currently support **higher resolution** (`640x480 px`) and **event cameras** with `304x240` resolution.
    To generate the candidate positions, the _proper context_ and the _camera resolution_ have to be specified, specifically:
    - for _rgb cameras with `640x480 px`_: `find-best-positions-calibration.sh run camera-calibration-best-pos/rgb-cameras/640x480 200 640 480`;
    - for _event cameras with `304x240 px`_: `find-best-positions-calibration.sh run camera-calibration-best-pos/event-cameras/304x240 200 304 240 0.1734 0.13`. Notice that event-cameras require two additional parameters, i.e. the board parameters. This is because, in order to generate events, the pattern is shown flashing on a tablet, thus the board parameters are derived from the tablet dimensions.


- **Run the calibration**: 

    For each set of candidate poses, we run the calibration procedure by using the `stereoCalib` module, as shown in the following image:

    <p align="center">
    <img src="https://user-images.githubusercontent.com/9716288/104918603-48b8c580-5995-11eb-9d1a-dc25f6360f4b.gif" width="500">
    </p>

- **Evaluate the calibration**: 

    We define a test set containing 100 images geneared randomly both in positions and orientation. Importantly, the test set is unique for all the candidate sets, therefore the evaluation scores can be fairly compared. 
    
    This step was performed by running:
    
    `create-calibration-test-set.sh run camera-calibration-best-pos/rgb-cameras/320x240 1 100`

    For each set we save the calibration result `outputCalib.ini` and copy it into `camCalib`.  We thus evaluate the quality of the calibration procedure by comparing the undistorted image provided by `camCalib` using the parameters estimated by the calibration procedure and the ground truth image provided by the ideal undistorted camera in `Gazebo`, as shown by the following: 

    <p align="center">
    <img src="https://user-images.githubusercontent.com/9716288/104717213-405c5280-5729-11eb-956b-dd146b4a78a3.png" width="700">
    </p>

    This step was performed by running:
    
    `evaluate-calibration.sh run camera-calibration-best-pos/rgb-cameras/320x240 $ROBOT_CODE/camera-calibration-supervisor/testsets/rgb-cameras/320x240 200`

    :point_up:  **Important note**:

     To create the test set and run the evaluation for different resolution or camera, the _proper context_ and the _camera resolution_ have to be specified, specifically:

    - for _rgb cameras with `640x480 px`_: 
        - `create-calibration-test-set.sh run camera-calibration-best-pos/rgb-cameras/640x480 1 100 640 480`;
        - `evaluate-calibration.sh run camera-calibration-best-pos/rgb-cameras/640x480 $ROBOT_CODE/camera-calibration-supervisor/testsets/rgb-cameras/640x480 200`
    - for _event cameras with `304x240 px`_:
        -  `create-calibration-test-set.sh run camera-calibration-best-pos/event-cameras/304x240 1 100 304 240`. 
        -  `evaluate-calibration.sh run camera-calibration-best-pos/event-cameras/304x240 $ROBOT_COTDE/camera-calibration-supervisor/testsets/event-cameras/304x240 200`

- **Choose the best set of poses**: 

    The set of poses that generates the highest score, as computed through the previous step, is selected. 

### :mag_right:  Supervise the calibration procedure

The supervisor provides an interface between the user and the calibration, in order to guide to user to place the chessboard in the poses identified through the above described pipeline. 
Specifically, the module loads the best poses (i.e. the corner pixels of the bounding boxes) and the corresponding images and:

- visually shows where to place the chessboard;
- evaluates the content of the chessboard placement by:
    1. checking that a chessboard is present;
    2. if a chessboard is found, doing element-wise comparisons between corresponding pixels of the current content and the one generated in `Gazebo`;
    3. giving a visual colour coding that ranges from `red` to `green` to assist the user while placing the board.
    4. providing the accuracy in percentage, computed as described in 2., as extra input to the user. 

    <p align="center">
    <img src="https://user-images.githubusercontent.com/4537987/104767254-2001b780-576c-11eb-87cb-1c6897cc4f43.png" width="400">
    </p>

- sends both the **left** and **right** frames to `stereoCalib` for processing, *once the board is correctly placed*.

The described procedure is iterated through all the positions generated by the evaluation process (currently 30).

Once all the frames are processed, the `stereoCalib` module generates the `outputCalib.ini` with the correct parameters that can be used to remove the camera distortion eg: camCalib.    

## :arrow_forward: How to install and use it

### Manual installation

If you want to install the repository manually, you can refer to the following instructions.

#### Dependencies

- [YARP](https://github.com/robotology/yarp)
- [iCub](https://github.com/robotology/icub-main)
- [YCM](https://github.com/robotology/ycm)
- [icub-contrib-common](https://github.com/robotology/icub-contrib-common)
- [OpenCV](https://github.com/opencv/opencv)

#### Compilation

On `Linux` system, the code can be compiled as follows:

``` 
git clone https://github.com/robotology/camera-calibration-supervisor.git
cd camera-calibration-supervisor
mkdir build && cd build
ccmake ..
make install 
```

#### Usage

- **Running the supervision pipeline**

    First, run the [cameras](https://github.com/robotology/icub-main/blob/master/app/default/scripts/cameras.xml.template) and the `yarprobotinterface` running (needed by `stereoCalib`).

    In order to run the supervision pipeline, you can use [this application](https://github.com/robotology/camera-calibration-supervisor/blob/main/modules/calibSupervisor/app/scripts/calibSupervisor-rgb-cam.xml). Open the application in `yarpmanager`, run all modules and hit connect.

    Now `calibSupervisor` waits for the images to be placed in the correct positions. 

    :warning: **Supervising event-cameras calibration**: You can rely on [this application](https://github.com/robotology/camera-calibration-supervisor/blob/main/modules/calibSupervisor/app/scripts/calibSupervisor-event-cam.xml). Be sure that `yarprobotinterface` and the [event-cameras](https://github.com/robotology/event-driven/blob/master/src/applications/view/app_event-viewer-example.xml) are running, with the following parameters:

    - `vPreProcess --flipy --filter_temporal false --filter_spatial false --sf_tsize 0.05 --split_stereo`
    - `vFramerLite --displays "(/left (BLACK) /right (BLACK))" --frameRate 30 --eventWindow 0.1`

- **Copying the parameters in the correct place**

    You can run [this script](https://github.com/robotology/camera-calibration-supervisor/blob/main/app/modify-params.sh) specifying `icubEyes.ini`, `outputCalib.ini` and `cameraCalibration` context:

    `modify-params.sh icubEyes.ini outputCalib.ini cameraCalibration`

- **Checking the result**

    You can finally run the [`camCalib` application](https://github.com/robotology/icub-main/blob/master/app/default/scripts/cameras_calib.xml.template), which will load the correct file, and visualize the undistorted images.


### Deployment

If you want to run the automatic deployment, you can refer to the instructions in the Applications section of [Robot Bazaar](https://robot-bazaar.iit.it/apps/academy/courses/25).


## :dart: Final result

The images below show the before and after calibration using the devised pipeline. We can notice that the distortion is correctly handled by the calibration. 

| Before calibration | After calibration |
| ------------- | ------------- |
|<p align="center"> <img src=https://user-images.githubusercontent.com/4537987/104766059-39096900-576a-11eb-997c-a187703cdedb.png width="400"> </p> | <p align="center">  <img src=https://user-images.githubusercontent.com/4537987/104766080-4292d100-576a-11eb-8c0c-2d727d692e45.png width="400"> </p> |

[This video](https://user-images.githubusercontent.com/9716288/111452620-fe449280-8712-11eb-8cc3-771487b0a764.mp4
) shows the entire pipeline working on the robot. When the 30 images are covered, the calibrated images are shown on the right.


## :book: References

[1] Z. Zhang, "A flexible new technique for camera calibration," in IEEE Transactions on Pattern Analysis and Machine Intelligence, vol. 22, no. 11, pp. 1330-1334, Nov. 2000, doi: 10.1109/34.888718.

