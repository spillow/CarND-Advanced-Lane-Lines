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
[image7]: ./output_images/remap.png "Lane Remapping"
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

This is implemented in cell `In [143]` in the function `calc_poly_fit_and_curvature()`.

Having calculated both degree-2 polynomials to fit the lane lines as Ay^2 + By + C, the radius of curvature of one of the lines is:

`R = (1 + (2Ay + B)^2)^1.5 / |2A|`

where y = 1279 in this case (the nearest portion of the lane lines to the camera).  It was found that the most stable radius was determined by the minimum of the two lanes.
Depending on how far the perspective transform was taken, one line could appear more straight than the other and give inflated measurements.

Having selected a perspective transform the maps the left line to x = 200 and the right to x = 1000, instead of the center of the image (1280/2=640) 600 ((200+1000)/2) was used here.
Selecting the value of the polynomials of both lanes at the bottom of the image and calculating the midpoint between both points, we simply measurement the offset from that value.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

This is implemented in `In [145]` in the function `remap_lines()`.  After filling in the area of the polynomial curves, the inverse of the perspective transform
is used to map the curves back to the surface of the road:

![alt text][image7]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to the annotated video result](https://github.com/spillow/CarND-Advanced-Lane-Lines/blob/master/output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

* As usual, the difficulties often involve experimeting with the parameters of the algorithms.  Extracting the lane lines involved tweaking the thresholds of the Sobel operators and
how to combine them with the magnitude and direction thresholds to achieve reasonable results.

* The perspective transform points needed to be selected such that they encompassed the lane of interest but didn't select any other regions that trigger false lane lines.  For example,
an early transform that was too large picked up the car that appears at the 0:27 mark of the test video and moved the lane lines over.

* Although the S channel of the HLS color space performs fairly well in general, some lane markings that are quite visible with grayscale now disappear.  Often, this would happen with the
dashed right lane.  A moving average of four frames (via `inter_frame_lane_estimation()`) and outlier rejection were implemented to avoid those pesky few frames that would spoil lane tracking.

* I think a more robust implementation could combine different color spaces in different tiles of the image by selecting which one works best in which part (by, say, has the strongest
signal-to-noise ratio in that region).

* As mentioned earlier, the pipline would benefit from a general road detector that would allow the perspective transform to be modified on a frame by frame basis to account for more winding
roads and changes in road slope.

* A convolutional net approach would be interesting to experiment with.  Perhaps the sliding window approach coupled with a network that was trained to discriminate between image tiles that
did or did not contain a lane line could yield a much more robust detector.

