# Supervising camera calibration using optimized chessboard poses

*Camera calibration* is a crucial step in computer vision applications, as it allows to estimate the scene's structure and remove lens distortion. 

The camera is typically modeled with a set of intrinsic (i.e. focal length, principal point and axis skew) and extrinsic parameters (i.e. rotation and translation).
The most common calibration procedure estimates such parameters using known points in the real world and their projection in the image plane (see [References](#references)), presented as a pattern with known geometry, usually a *flat chessboard*. During the calibration, the user has to move the chessboard around in order to cover the whole field of view, including different orientations of the pattern. The calibration result is *highly affected* by such positions and it is not unusual that several trials have to be carried out in order to have a proper calibration result. It is therefore fundamental to place the pattern properly to avoid the final image to be distorted.
 
This repository contains the software required to identify **the best chessboard poses** and to **visually guide the user** to place the chessboard in the selected poses. The set of poses which maximizes the result of the calibration procedure is selected as best set. 

## Camera calibration on the iCub

[`stereoCalib`](https://robotology.github.io/robotology-documentation/doc/html/group__icub__stereoCalib.html) is the module used on the iCub to perform stereo calibration (both intrinsic and extrinsic parameters). The module saves the final result onto a file called `outputCalib.ini`. 

Typically, such file is manually copied into `icubEyes.ini`, which is in turn loaded by [`camCalib`](https://robotology.github.io/robotology-documentation/doc/html/group__icub__camCalib.html), i.e. the module performing camera undistortion.

Our procedure uses `stereoCalib` to perform the calibration. Then we copy `outputCalib.ini` into the file loaded by `camCalib` to evaluate the quality of the calibration, by assessing how *similar* the undistorted image produced by `camCalib` is to the ideal image.

## The devised procedure

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
     
     `find-best-positions-calibration.sh run camera-calibration-best-pos 200`

    _Note: To run the script for event-cameras, you will need to specify a different context:_
    - `find-best-positions-calibration.sh run camera-calibration-best-pos/event-cameras 200 0.1734 0.13`.

- **Run the calibration**: 

    For each set of candidate poses, we run the calibration procedure by using the `stereoCalib` module, as shown in the following image:

    <p align="center">
    <img src="https://user-images.githubusercontent.com/9716288/104918603-48b8c580-5995-11eb-9d1a-dc25f6360f4b.gif" width="500">
    </p>

- **Evaluate the calibration**: 

    We define a test set containing 100 images geneared randomly both in positions and orientation. Importantly, the test set is unique for all the candidate sets, therefore the evaluation scores can be fairly compared. 
    
    This step was performed by running:
    
    `create-calibration-test-set.sh run camera-calibration-best-pos 1 100`

    For each set we save the calibration result `outputCalib.ini` and copy it into `camCalib`.  We thus evaluate the quality of the calibration procedure by comparing the undistorted image provided by `camCalib` using the parameters estimated by the calibration procedure and the ground truth image provided by the ideal undistorted camera in `Gazebo`, as shown by the following: 

    <p align="center">
    <img src="https://user-images.githubusercontent.com/9716288/104717213-405c5280-5729-11eb-956b-dd146b4a78a3.png" width="700">
    </p>

    This step was performed by running:
    
    `evaluate-calibration.sh run camera-calibration-best-pos $ROBOT_CODE/camera-calibration-supervisor/testsets/rgb-cameras 200`

    _Note: To run the scripts for event-cameras, you will need to specify a different context:_ 
    - `create-calibration-test-set.sh run camera-calibration-best-pos/event-cameras 1 100`;
    - `evaluate-calibration.sh run camera-calibration-best-pos/event-cameras $ROBOT_CODE/camera-calibration-supervisor/testsets/event-cameras 200`.

- **Choose the best set of poses**: 

    The set of poses that generates the highest score, as computed through the previous step, is selected. 

### Supervise the calibration procedure

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

## Final result

The images below show the before and after calibration using the devised pipeline. We can notice that the distortion is correctly handled by the calibration. 

| Before calibration | After calibration |
| ------------- | ------------- |
|<p align="center"> <img src=https://user-images.githubusercontent.com/4537987/104766059-39096900-576a-11eb-997c-a187703cdedb.png width="400"> </p> | <p align="center">  <img src=https://user-images.githubusercontent.com/4537987/104766080-4292d100-576a-11eb-8c0c-2d727d692e45.png width="400"> </p> |


## References

[1] Z. Zhang, "A flexible new technique for camera calibration," in IEEE Transactions on Pattern Analysis and Machine Intelligence, vol. 22, no. 11, pp. 1330-1334, Nov. 2000, doi: 10.1109/34.888718.

