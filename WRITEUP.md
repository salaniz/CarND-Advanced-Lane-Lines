## **Advanced Lane Finding Project**

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

[undist1]: ./output_images/chessboard_undistort.png "Undistorted"
[undist2]: ./output_images/undist_example.png "Undistorted"
[warped1]: ./output_images/warped1.png "Road Transformed"
[warped2]: ./output_images/warped2.png "Road Transformed"
[warped3]: ./output_images/warped3.png "Road Transformed"
[warped4]: ./output_images/warped4.png "Road Transformed"
[warped5]: ./output_images/warped5.png "Road Transformed"
[warped6]: ./output_images/warped6.png "Road Transformed"
[warped7]: ./output_images/warped7.png "Road Transformed"
[warped8]: ./output_images/warped8.png "Road Transformed"
[pipeline1]: ./output_images/pipeline1.png "Pipeline"
[pipeline2]: ./output_images/pipeline2.png "Pipeline"
[pipeline3]: ./output_images/pipeline3.png "Pipeline"
[pipeline4]: ./output_images/pipeline4.png "Pipeline"
[pipeline5]: ./output_images/pipeline5.png "Pipeline"
[pipeline6]: ./output_images/pipeline6.png "Pipeline"
[pipeline7]: ./output_images/pipeline7.png "Pipeline"
[pipeline8]: ./output_images/pipeline8.png "Pipeline"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

**Here I will consider the rubric points individually and describe how I addressed each point in my implementation.**  
The code and referenced code cells can be found in the Jupyter Notebook accompanying the writeup.


### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

Code Cell No.: 2

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  


I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][undist1]

### Pipeline (single images)
An example pipeline output with all intermediate results is shown in at the end of this section.

#### 1. Provide an example of a distortion-corrected image.

Code Cell No.: 2

In the final pipeline, I substituted the function `cv2.undistort()` with a one time execution of `cv2.initUndistortRectifyMap()` which gives mappings for both dimensions x and y. These maps can then be applied on each image on the video steam with `cv2.remap()` for a faster calculation of the distortion correction.

To demonstrate this step, here is an example of an undistorted road image (especially visible with the road sign):
![alt text][undist2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

Code Cell No.: 3

I used a combination of color and gradient thresholds to generate a binary image. White lanes are detected with a absolute sobel threshold in x direction that also satisfy a lightness threshold. Yellow lanes are detected by a color threshold of the hue value together with a saturation and lightness threshold. Here's an example of my output for this step.

Note: see the pipeline images below for examples

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

Code Cell No.: 4

Perspective transform is done by calling `cv2.getPerspectiveTransform()` to get the transformation matrix that results in a birds-eye view of the road.
This is based on source (`src`) and destination (`dst`) points of straight lines. I chose the hardcode the source and destination points by looking at the two provided pictures that contains straight road lines and visually fitting two straight lines to them.

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 240, 700      | 380, 700      | 
| 596, 450      | 380, -200     |
| 686, 450      | 900, -200     |
| 1075, 700     | 900, 700      |


I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][warped1]
![alt text][warped2]
![alt text][warped3]
![alt text][warped4]
![alt text][warped5]
![alt text][warped6]
![alt text][warped7]
![alt text][warped8]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Code Cell No.: 5

In order to detect lanes a sliding window over a histogram of pixel values is used to determine the lanes initially.
Subsequent lanes are updated by detected pixels around the current lane lines using a margin hyperparameter. The detected lane line pixels are used to fit a second order polynomial. If a sequence of images are used (video detection) then the detected polynomial is based on both the current detection and the last fit. The new fit is calculated by weighting the previous coefficients of the polynomial with 0.9 and the new ones with 0.1. This corresponds to an exponential decay in contribution of the lane detections per frame.

Note: see the pipeline images below for examples

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

Code Cell No.: 6

Using the second order polynomial of each lane the radius of the curvature can be calculated by:
```python
radius = ((1 + (2*A*y + B)**2)**1.5) / np.absolute(2*A)
```
where A and B are the coefficients of the polynomial f(y) = Ay^2 + By + C and y is the position to be evaluated.
I evaluated the curvature at the very bottom of the image.

In order to convert the distances from pixel space to distances in meters I adjusted the coefficients:

A_m = A * xm_per_pix / ym_per_pix^2
B_M = B * xm_per_pix / ym_per_pix

The values for xm_per_pix and ym_per_pix are taken from the lecture notes:  
ym_per_pix = 30/720 # meters per pixel in y dimension
xm_per_pix = 3.7/700 # meters per pixel in x dimension

Finally, the radius of the road is calculated as the average of both radii from the left and right lane.

Similarly, the vehicle position can be calculated by looking at the x-positions of the lanes at the bottom of the image. The center of the line is the mid-point of these two positions and the cars deviation from the center lane is the difference of the center of the lane from the center of the image (car) regarding the x-position. This difference can be than easily turned in to meter scale by using xm_per_pix This difference can be than easily turned in to meter scale by using xm_per_pix as a scaling factor.


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Plotting back down onto the original (undistorted) road image is done simply by doing a perspective transformation of the detected lane lines with the inverse transformation matrix calculated earlier. The area of the road is highlighted additionally. 

The full pipeline is visualized here:

![alt text][pipeline1]
![alt text][pipeline2]
![alt text][pipeline3]
![alt text][pipeline4]
![alt text][pipeline5]
![alt text][pipeline6]
![alt text][pipeline7]
![alt text][pipeline8]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a link to my [video result](./project_video.mp4) for the project video as well as a [debug video](./project_video.mp4) that additionally shows the lane detection.

In order to detect lanes on the challenge video, the hyperparameters for color/gradient thresholds were slightly adjusted. This way the lanes could be detected: [video result](./project_video.mp4), [debug video](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

In my opinion, the most crucial part is lane detection. Whether it is through color/gradient thresholds or something else, it seems to be a non-trivial problem to detect lanes robustly in various conditions (lighting, obstacles, weather) without detecting other parts of the image. In my approach, I had to adjust the thresholds slightly to allow a better detection on the challenge video. To improve this, one could think about dynamically adjusting the hyperparameters given certain properties of the image such as lighting situation. However, when conditions are even worse (e.g., like in the harder challenge video) that not even a human can make out lane lines in certain images, the algorithms needs to be made more robust to not pick up unwanted signals as detected lane lines and instead keep old detections. For instance, if the shape of detected pixels do not look like a lane, that particular frame could be discarded.
