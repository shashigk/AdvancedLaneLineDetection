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
* Project submission files are listed below:
  * P4.ipynb -- The python notebook with the image pipeline. The initial part of the notebook
                has exploratory code along with utility functions used later. The image pipeline
                is implemented as part of the python `class Line`, particularly, the member method `process_image`.
  * output_images -- Contains result of running the image pipeline on test images.
  * project_video_result.mp4 -- Project video annotated with lane, curvature and vehicle position
  * challenge_video_result.mp4 -- Annotated challenge video
  * harder_challenge_video_result.mp4 -- Annotated harder challenge video
  * examples 		-- Examples for the writeup.
  * persp_points.txt    -- Source and Destination points used for perspective transform.

[//]: # (Image References)

[imageCBORG]:	 ./examples/chessboard_2_orig.png 	             "Original Chess board Image"
[imageCBWarped]: ./examples/chessboard_2_warped.png 	             "Warped Chess board Image"
[imageCBUndist]: ./examples/chessboard_2_undistort.png  	     "Chess Board Undistorted image"
[imageSLWarped]: ./examples/staright_lines_1_warped.png 	     "Straight Lines Image Warped"
[imageSLOrig]:	 ./examples/straight_lines1.jpg			     "Straight Lines original image"
[imageSLBinary]: ./examples/staright_lines_1_color_sel_threshold.png "Binary gradient thresholded and "
[imageSLHist]:   ./examples/straight_lines_1_binary_hist.png  	     "Histogram of binary straight line image"
[imageSLUndist]: ./examples/straight_lines_1_undistort.png           "Undistored image of straight line image"
[imageHistSrch]: ./examples/histogram_search.png		     "Histogram search result"
[imagePolyPlot]: ./examples/lane_fit_poly.png			     "Polynomial plot of the lane lines"
[output1]:	 ./output_images/test1.jpg			     "Output of pipeline 1"
[output2]:	 ./output_images/test2.jpg			     "Output of pipeline 2"
[output3]:	 ./output_images/test3.jpg			     "Output of pipeline 3"
[output4]:	 ./output_images/test4.jpg			     "Output of pipeline 4"
[output5]:	 ./output_images/test5.jpg			     "Output of pipeline 5"
[output6]:	 ./output_images/test6.jpg			     "Output of pipeline 6"
[video1]:        ./project_video_result.mp4 			     "Result of running pipeline on video"
[video2]:        ./challenge_video_result.mp4 			     "Result of running pipeline on challenge video"
[video3]:        ./harder_challenge_video_result.mp4 	             "Result of running pipeline on harder challenge video"


---

### Writeup / README

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

I used the camera images in the folder "camera_cal" for camera calibration.
These were images of chessboard. I converted each of these image to grayscale 
and computed calibration points using the OpenCV API `cv2.findChessboardCorners`. 
The points returned were accumulated from the images into an array.
The object points (i.e., expected positions) were also accumulated into an array; see the function
`calibrationPoints` in the associated notebook for more details.
The calibration matrix and distortion coefficients were then computed using the
OpenCV API `cv2.calibrateCamera`. Then I used `cv2.undistort()` function to undistort
the images. The result of applying the undistoring on a sample chessboard image is shown below.

![origchessimg][imageCBORG]

### Pipeline (single images)

In the python notebook, the initial part of the code is more about experimenting with
the thresholds, the actual pipeline is implemented in the class Line, in particular,
in the method `process_image`, which inturn calls `process_image_combo`.
The steps in the pipeline are described below:

1. Undistort the input image
2. Color thresholding: Create a binary image by converting HLS space and thresholding the S-channel.
3. Gradient thresholding: Create a binary image using sobel operator the gray scaled image.
4. Combine the binary images from color thresholding and gradient thresholding.
5. Transform the image using perspective transformation, save the inverse transform.
6. Identify Lane pixels using histogram.
7. Fit polynomials for the left and the right lanes.
8. Compute curvature and vehicle position.
9. Fill a polygon whose vertices are computed using the polynomials.
10. Unwarp the polygon using inverse perspective transform and draw it on top of the original image.
11. Write the curvature and vehicle position on top of the original image.
12. Output annotated image.

Video processing is quite similar to the above steps, however, instead of computing
the lane pixels using histogram independently in each frame, a search for the lane pixels
is done in the vicinity of lane found in the previous frame. Further, a moving average of the
coefficients is maintained over 5 frames to achieve a more smoothed lane detection.
The moving average of the polynomial coefficients was maintained using separate class namely
`class PolyMovingAverage`.

#### 1. Distortion Correction.

The distortion parameters computed above using the chessboard images was used to undistort lane images.
See `UndistortUsingCalibrationParams` function for how this is done.
For example, see the picture below.

![straightLineUndist][imageSLUndist]

#### 2. Binary Image Creation.
Binary image creation is done in the function `undist_threshold_warp_img`, specifically, lines after
undistortion. There are two main operations performed, firstly, color thresholding 
converts the image to HLS space and then selects pixels at locations where S channel is in
the range 170, to 255. 	



The second is absolute gradient thresholding in the 'x' direction using sobel operator.
This step first converts converts the image to grayscale, and computes the absolute value of
gradient in the 'x' direction using Sobel operator with kernel size of 3.
The pixels with values in range 20, 100 are then selected in the output.

The output of the binary image creation is the pixels which have a 1 either using
color thresholding or using the sobel operator.

![binaryimage][imageSLBinary]

#### 3. Perspective transformation.

I manually selected the points on the image, by choosing two points towards the
bottom of the image near the lane lines and then constructing a rectangle through
these points in the destination image.
This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 258,  675	| 258, 675      |
| 1039, 675     | 1039, 675     |
| 590,  450     | 258, 0        |
| 695,  450     | 1039, 0       |

I applied the perspective transformation on the binary image to get the following
image where the lanes look almost parallel.

![lanewarped][imageSLWarped]

This was done towards the end of the function `undist_threshold_warp_img` in the python notebook.

#### 4. Lane Identification

The lane identification was done using histogram searching, by first computing the image histogram
and identifying two peaks as shown below.

![imagehist][imageSLHist]

Once the peaks were identified, a sliding window search was done to identify potential lane pixels
as shown below.

![HistSrch][imageHistSrch]

These pixels were then used to compute the coefficients of polynomial to identify the lane-lines.
This is illustrated in the figure below.

![PolyPlot][imagePolyPlot]

The polynomials were computed in the both in the pixel space and also in the space
of meters, using the following multipliers mentioned in the course.
`ym_per_pix = 30/720 # meters per pixel in y dimension`
`xm_per_pix = 3.7/700 # meters per pixel in x dimension`

#### 5. Calculation of Radius of Curvature, Lane Breadth, and, Vehicle Position.

The polynomial coefficients (the ones computed in meter space) were used to evaluate
the formula for curvature at the bottom of the image, where y-value will be the largest.
See the function `calculate_curvature` for details.

The lane breadth was computed by computing left point of the lane corresponding to
the maximum y-value using the left-polynomial. Similarly, the right point is computed.
The difference is value is the lane breadth; see the function `lane_breadth`.
The vehicle position is computed by computing the difference in the mid-point of the image
and the mid-point of the left and the right points computed from the polynomials above;
see the functions `vehicle_pos` and `vehicle_pos_msg`.

#### 6. Illustration of the Lane Identification on test images.

The result of running the image pipeline on test images in the `test_images` folder is shown below:

Input: test1.jpg

![output1][output1]

Input: test2.jpg

![output2][output2]

Input: test3.jpg

![output3][output3]


Input: test4.jpg

![output4][output4]


Input: test5.jpg

![output5][output5]


Input: test6.jpg

![output6][output6]


---

### Pipeline (video)

#### Video Processing -- filtering by lane width and reset moving average in case of large deviations.

For the purpose of processing videos, there were two main modifications done to ensure
smooth transition between frames. Firstly, the polynomial coefficients were maintained
over a window of 5 frames. This was done using the class `PolyMovingAverage`.
Further, for a new frame the histogram search was done in the vicinity of the pixels
found in the previous frame. A filter condition was used for resetting the moving
average or the histogram search, namely, the lane breadth computed had to be in the range
of 3, and 3.9. 

Further, if the norm of the coefficents for the polynomial in the new frame differed by
more than 10% to the norm of the average coefficients, then the moving average was reset.


#### 1. Results on the project video.

The video pipeline performed very well on the project video, see the result below.
[video1](./project_video_result.mp4)

However, it had difficulty in handling the challenge videos, [video2](./challenge_video_result.mp4),
[video3](./harder_challenge_video_result.mp4).

---

### Discussion

#### Scope for Improvement.

While the pipeline works well for the project video, there are issues in processing the challenge videos.
In one of the challenge videos a portion of the shoulder had
a white portion and was mistaken for a lane boundary. This indicates that color and gradient thresholds 
perhaps need more tuning. Another possibility is to explore different color spaces.

The other main problem, I think is the transition portion of the moving average approach to maintain
polynomial coefficients. It should be more smoothly adaptive, instead of resetting the window size.
Currently, I only check the lane breadth and coefficient norm, I think a better approach would
also involve curvature. If the curvature is too large then road is almost straight and one can use
a larger window, if the curvature is small then the road is changing potentially and we need a smaller
window.

