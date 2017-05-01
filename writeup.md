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

[image1]: ./output_images/calibration.png "Undistorted"
[image2]: ./output_images/undistored_image.png "Road Transformed"
[image3]: ./output_images/binary_image.png "Binary Example"
[image4]: ./output_images/perspective.png "Warp Example"
[image5]: ./output_images/top_down_extract.png "Top Down Extract"
[image6]: ./output_images/traced_path.png "Traced Path"
[image7]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

----------------------

The project includes the following files:
* [Lane_Detection.ipynb](https://github.com/spillow/CarND-Advanced-Lane-Lines/blob/master/Lane_Detection.ipynb) The code
* [output.mp4](https://github.com/spillow/CarND-Advanced-Lane-Lines/blob/master/output.mp4) Video with annotated lane region

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained under the "Camera Calibration" heading in the ipython [notebook](https://github.com/spillow/CarND-Advanced-Lane-Lines/blob/master/Lane_Detection.ipynb).

First, the "object points" are created, which will be the (x, y, z) coordinates of the chessboard corners in the world. Here it is assumed that the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time we successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.

We then use the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  The resulting camera calibration matrix and distorition coefficients are then applied to the test image using the `cv2.undistort()` function with this result:

![alt text][image1]

With the left image the original and the right having removed distortion (note the more parallel lines).

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

Having obtained the camera calibration matrix and distortion coefficients above, the same procedure of image undistortion via `cv2.undistort()` may be used on road images:
![alt text][image2]

(Note the position of the white car in both images with respect to the right edge of the image)

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The implementation for image binarization to extract candidate pixels for lane lines in under the "Thresholding" section of the ipython notebook in `lane_line_extract()`.

The image is first converted from RGB to HLS and the S (Saturation) channel is extracted.  This channel is more immune to changes from lighting conditions than, say, a grayscale image.

Then, a 3x3 sized Sobel operator is applied to the S channel in the x and y directions.  This tells us whether the image is changing in either direction.  These features are combined
to calculate the magnitude and direction of the gradient of each point.  For example, we can reject gradients that are very "flat" (i.e., little change in the x-direction but mostly in the y) as these are very unlikely to be lane lines (or possibly the car is in a very dicey situation!).  Given the code in cell `In [128]` we extract:

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code to calculate the perspective transform is in cell `In [127]` in `ex2()`.  The idea is to take the head on view of the road from the camera and transform it into a
top down perspective that will make it easier to fit a curve to the lines and calculate the radius of curvature.

To calculate a perspective transform, a four point correspondence is needed between the source image and where we want those points mapped to in the resulting image.

Four points are manually selected that draw out a trapezoid of the lane lines to map to a top down rectangle:

| Source        | Destination   |
|:-------------:|:-------------:|
| 200, 1279     | 200, 1279     |
| 590, 450      | 200, 0        |
| 690, 450      | 1000, 0       |
| 1100, 1279    | 1000, 1279    |

Although a more robust implementation may dynamically calculate this as the road shape/angle changes, for our purposes we use the same transformation throughout.

The perspective transform was verified by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image:

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The lane line pixels are identified in the function `get_lane_line_pixels()` in cell `In [142]`.  General approach:

* Find the base of the lane lines:
  - Looking at the bottom quarter of the image, generate a histogram of the pixel counts in each column in the left and right halves of the image.  The max position is a good candidate for the bottom of that lane line.
* Starting at the positions found above, partition the image vertically into `nwindows` # of windows and center the window on the mean of the pixels within that window or just slide the
  previous window up if not enough pixels were found.

With this approach, we can trace the arc of a lane line with windows sliding to follow its path:

![alt text][image5]
![alt text][image6]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in lines # through # in my code in `my_other_file.py`

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`.  Here is an example of my result on a test image:

![alt text][image7]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.
