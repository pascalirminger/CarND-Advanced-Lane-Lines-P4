# Advanced Lane Finding Project

[//]: # (Image References)
[image1]: ./examples/calib_original_to_undist.png "Undistorted Calibration Image"
[image2]: ./examples/test_original_to_undist.png "Undistorted Test Image"
[image3]: ./examples/test_undist_to_binary.png "Binary Example"
[image4]: ./examples/test_undist_to_warped.png "Warp Example"
[image5]: ./examples/test_binary_to_binarywarped.png "Binary Warp Example"
[image6]: ./examples/test_binarywarped_to_lanelines.png "Detect Lane Lines"
[image7]: ./examples/test_undist_to_curvature.png "Lane Curvature and Vehicle Position"
[image8]: ./examples/test_undist_to_lanelines.png "Output"
[video1]: ./output_videos/project_video_output.mp4 "Video Output"
<img src="examples/title-image.jpg" width="480" alt="Traffic Sign Classifier" />

## Overview

This repo contains the code written to complete the fourth project on [Udacity Self-Driving Car Nanodegree Program](https://www.udacity.com/course/self-driving-car-engineer-nanodegree--nd013) (Term 1). This project uses computer vision to find lane lines on a video stream.

## Camera Calibration

The code for this step is contained in the `Camera` class in the third code cell of the IPython notebook located in [`Pipeline.ipynb`](Pipeline.ipynb).

The calibration of the camera is implemented in the `calibrate()` function of the `Camera` class. I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image. Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image. `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function. I applied this distortion correction to a calibration image using the `cv2.undistort()` function and obtained this result:

![Undistorted Calibration Image][image1]

## Pipeline (single images)

The following image shows the result of applying the distortion correction to one of the test images:

![Undistorted Test Image][image2]

I used a combination of color and gradient thresholds to generate a binary image. The code for this step is in the `ImageThresher` class in code cell 8 of the IPython notebook located in [`Pipeline.ipynb`](Pipeline.ipynb). The following thresholds are being used:

| Threshold          | min, max          |
|:------------------:|:-----------------:|
| Gradient Magnitude | 50, 255           |
| Gradient Direction | 0.7, 1.3          |
| Color (R-channel)  | 220, 255          |
| Color (Y-channel)  | 150, 255          |

They are combined as follows:

```python
binary = ( Magnitude & Direction ) | ( R-channel | S-channel )
```

Here is an example of the output for this step:

![Binary Example][image3]

The code for my perspective transform is in the `ImageWarper` class in code cell 9 of the [IPython notebook](Pipeline.ipynb). The `warp_image()` function takes an image as input and warps it to a top-down-view. The `unwarp_image()` function takes an image as input and warps it vice versa. I chose to hardcode the source and destination points in the following manner:

```python
y_top = 455
y_bottom = 690
x_bottom_l = 240
x_top_l = 585
x_top_r = 700
x_bottom_r = 1085
width, height = image_size

# Using an offset on the left and right side allows the lanes to curve
offset = 200

src = np.float32([ 
    [x_bottom_l, y_bottom],
    [x_top_l, y_top],
    [x_top_r, y_top],
    [x_bottom_r, y_bottom]
])
dst = np.float32([
    [offset, height],
    [offset, 0],
    [width-offset, 0], 
    [width-offset, height]
])
```

This resulted in the following source and destination points:

| Corner        | Source        | Destination   | 
|:-------------:|:-------------:|:-------------:| 
| Bottom, left  | 240, 690      | 200, 720      | 
| Top, left     | 585, 455      | 200, 0        |
| Top, right    | 700, 455      | 1080, 0       |
| Bottom, right | 1085, 690     | 1080, 720     |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![Warp Example][image4]
![Warp Example][image5]

The lane line detection code can be found in the functions `find_lane_lines()` and `find_and_visualize_lane_lines()` in code cell 15 of the [IPython notebook](Pipeline.ipynb). The algorithm calculates the histogram on the X axis, searches for the peaks on the left and right side of the image, and collects the non-zero points contained on those windows. When all the points are collected, a polynomial fit is used (`np.polyfit()`) to find the line model in pixels and meters. The following picture shows the points found on each window, the windows and the polynomials:

![Detect Lane Lines][image6]

I implemented this step in the function `calculate_lane_curvature_and_vehicle_position()` in code cell 17 of the [IPython notebook](Pipeline.ipynb). The formula for calculating the lane curvature is as follows:

```python
((1 + (2*fit[0]*y_eval*ym_per_pix + fit[1])**2)**1.5) / np.absolute(2*fit[0])
```

where ```fit``` is the array containing the polynomial, ```y_eval``` is the max Y value, and ```ym_per_pix``` is the meter-per-pixel-ratio. To find the vehicle position on the center:

* Calculate the lane center by evaluating the left and right polynomials at the maximum Y and find the middle point.
* Calculate the vehicle center transforming the center of the image from pixels to meters.
* A positive distance between the lane center and the vehicle center indicates a shift to the right of the road. Vice versa, a negative distance indicates a shift to the left of the road.

Here is the result on a test image:

![Lane Curvature and Vehicle Position][image7]

I implemented this step in the function `draw_lane_lines_on_image()` in code cell 19 of the [IPython notebook](Pipeline.ipynb). The generated points where mapped back to the image space using the inverse transformation matrix calculated during the perspective transformation. Here is the result on a test image:

![Output][image8]

## Pipeline (video)

Here is a [link to my video result](./output_videos/project_video_output.mp4)

## Discussion

* There are a several improvements that could be done on the performance of the process.
* More information could be used from previous frames to improve the robustness of the process.
* Finetuning threshold values could be driven much further than I did so far.
* Currently, the transform points for perspective transformation are based assumes center-lane-driving on the test image. Moving them to the center of the image will increase accuracy in lane curvature and vehicle position calculations.
