## Writeup Template

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)
[distortion1]: ./writeup_images/distortion_chess.png ""
[distortion2]: ./writeup_images/distortion_road.png ""
[color1a]: ./writeup_images/colorprocessing1_beforeafter.png ""
[color1b]: ./writeup_images/colorprocessing1_hsl_l.png ""
[color1c]: ./writeup_images/colorprocessing1_lab_b_adapt.png ""
[color2a]: ./writeup_images/colorprocessing2_beforeafter.png ""
[color2b]: ./writeup_images/colorprocessing2_hsl_l.png ""
[color2c]: ./writeup_images/colorprocessing2_lab_b_adapt.png ""
[warp1]: ./writeup_images/warp1.png ""
[warp2]: ./writeup_images/warp2.png ""
[lines1]: ./writeup_images/lines1.png ""
[linesprojected1]: ./writeup_images/linesprojected1.png ""
[lines2]: ./writeup_images/lines2.png ""
[linesprojected2]: ./writeup_images/linesprojected2.png ""
[video]: ./test_videos/project_video_output.mp4 ""
[challengevideo]: ./test_videos/challenge_video_output.mp4 ""

## List of files
* Notebook: [lanes.ipynb](lanes.ipynb)
* Video files: see folder "test_videos"
* Images: test images are in folder "test_images", these are the ones provided with the project and those I added to calibrate my algorithm.

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

Here I will consider the rubric points individually and describe how I addressed each point in my implementation.

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in section 1 of the IPython notebook.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  This distortion correction was then applied to every frame of the video using the `cv2.undistort()` function. An example of distortion correction on one chessboard calibration image is reported as follows.

![alt text][distortion1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

Here is an example of distortion correction applied to one of the test images:

![alt text][distortion2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The code for this step is contained in section 2 of the IPython notebook.

A number of combinations of gradients (see `gradient_magnitude_threshold`, `gradient_direction_threshold` and `gradient_x_threshold` functions) and color spaces (RGB, HLS, LUV, LAB, see `color_select` function) with a number of thresholds and combinations were tested.

In the end only color spaces were used. I decided not to use gradients I found difficult to tell lines and road sides and imperfections apart. On the contrary, I found that combining different color spaces allowed a more accurate detection of the lines, without being too much disturbed by other road imperfections.

The final combination is an OR of:
* HLS L layer to identify white lines, `cv2.threshold` was used, selecting values above 200
* LAB B layer to identify yellow lines, `cv2.adaptiveThreshold` was used, with parameters `cv2.ADAPTIVE_THRESH_MEAN_C`, a "kernel" size of 21 and a `C` of -6. I found this solution to be more robust than `cv2.threshold` with threshold 160 in isolating lines in the "challenge" video where the yelow line is quite faded.

First example:

![alt text][color1a]
![alt text][color1b]
![alt text][color1c]

Second example:

![alt text][color2a]
![alt text][color2b]
![alt text][color2c]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for this step is contained in section 2 of the IPython notebook. In particular, refer to function `warp_image`.

The code makes use of cv2 functions `cv2.getPerspectiveTransform` to compute the transformation matrix and `cv2.warpPerspective` to apply it to the image. The inverse transformation matrix is also saved, as it will used later on to project the detected lines back on the original camera image.

The following points were used to compute the transformation matrix. They're mostly symmetrical, although some local corrections were necessary.

```python
    src = np.float32([[575+5, 460],
                      [w-575, 460],
                      [260, 685],
                      [w-260+20, 685]])
    dst = np.float32([[side, 0],
                      [w-side, 0],
                      [side, h],
                      [w-side, h]])
```

where `w` and `h` are the image dimensions and `side` is a parameter that specifies how far from the sides two straight lines are projected (the larger, the wider the area covered). This results in:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 580, 460      | 300, 0        | 
| 705, 460      | 980, 0        |
| 260, 685      | 300, 720      |
| 1040, 685     | 970, 720      |

Here are two examples:

![alt text][warp1]
![alt text][warp2]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The code for this step is contained in section 3 of the IPython notebook. In particular, refer to functions `lines_fit_new_frame` and `lines_fit_based_on_previous_frame`.

![alt text][lines1]
![alt text][lines2]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The code for this step is contained in section 3 of the IPython notebook. In particular, refer to function `curvature_radius_and_distance_from_centre`. The sliding window (with 10 vertical windows) based on the horizontal hystogram technique has been used, as presented in the course.

Fuctions `detect_linetype` and `lane_width_validation` are also used. The first estimates whether the line is continuous or dashed and it's used to better average left and right line curvatures (a continuous line, having more points, is interpolated better by the polynomial and the curvature can be trusted more than those computed on a dashed line) while the second checks that the line width is meaningful (this reduces the possibility of the road side to be detected as line, as the lane would be too wide).

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The code for this step is contained in section 3 of the IPython notebook. In particular, refer to function `draw_on_image`.

Here are two wxamples, relative to the two images shown before:

I implemented this step in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`.  Here is an example of my result on a test image:

![alt text][linesprojected1]
![alt text][linesprojected2]


### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

The project video is [here](./test_videos/project_video_output.mp4).

![alt text][video]

The same system is working also on the "challenge" video [here](./test_videos/challenge_video_output.mp4).

![alt text][challengevideo]

The code for this step is contained in section 3 and 4 of the IPython notebook. In particular, refer to class class `Line` in section 3 and to function `pipeline` in section 4. The first contains the logic that makes use of line detection in previous frames to better estimate the lines while the latter is the final processing pipeline that contains all the steps necessary to process a video frame.

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

A few problems observed are:
* Bad detection when the car is passing under the bridge in the "challenge" video. I tried to tackle this with adaptive thresholding that, while is working well for the yellow line, has still problems on the white one (that's why I resorted to a fixed threshold for the white lines)
* When the curves are too narrow, it can happen that the right line detectes points that actually belong to the left one (or viceversa).

A few improvements I thought about are:
* Implementing a convolution-based sliding window search to detected the line points.
* Lightness adaptation to better detect lines in case of strong variations of lightness.
