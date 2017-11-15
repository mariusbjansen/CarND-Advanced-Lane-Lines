## Advanced Lane Finding Project Writeup


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

  

* Encapcsulate image processing steps in functions.
* Apply image processing chain on a video.

[//]: # (Image References)

[image1]: ./output_images/undistorted_chessboard.png "Undistorted"
[image2]: ./output_images/undistorted_sample.png "Road Transformed"
[image3]: ./output_images/binary_threshold.png "Binary Example"
[image4]: ./output_images/warped.png "Warp Example"
[image5]: ./output_images/line_pixels_and_fit.png "Fit Visual"
[image6]: ./output_images/result_layered_lane_boundaries.png "Output"
[video1]: ./project_video.mp4 "Video"


## Here I will consider the rubric points individually and describe how I addressed each point in my implementation. 

## Since my jupyter notebook includes a lot of comments and explanation I want to condense information in this document making it easier being reviewed and avoiding to much redundancy.

---


### Camera Calibration

The code for this step is contained in the jupyter notebook 

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

The code is basically a copy of the lecture example.

### Pipeline (single images)

#### 1. Distortion-correction

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image. 

I used a combination of color and gradient thresholds to generate a binary image. Here's an example of my output for this step. 

![alt text][image3]

#### 3. Perspective transform 

The code for my perspective transform is a function called `warp()`. The `warp()` function takes as inputs an image (`img`). I chose the hardcode the source and destination points in the following manner:

```python
    corners_src = np.float32([[190,720],[585,458],[698,458],[1145,720]])

    new_width_shrink_offset=np.float32([200,0])
    img_size = (input.shape[1], input.shape[0])

    corner_dst_bottom_left  = corners_src[0]+new_width_shrink_offset
    corner_dst_top_left     = [corners_src[0,0],0]+new_width_shrink_offset
    corner_dst_top_right    = [corners_src[3,0],0]-new_width_shrink_offset
    corner_dst_bottom_right = corners_src[3]-new_width_shrink_offset
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 190, 720      | 390, 720      | 
| 585, 458      | 390, 0        |
| 698, 458      | 945, 0        |
| 1145, 720     | 945, 720      |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Identify lane-line pixels and fitting their positions with a polynomial

The function identifying the lane pixel is called `find_lane_lines`. It takes three arguments: `warped`, `left_fit`, `right_fit` and `tracked`. `tracked` default value is False.

The function takes the bottom half of a binarized and warped lane image and computes a histogram of candidate pixels. This is the starting point for following calculations.

The function uses a sliding window and iteratively moves up the image finding the lane lines.

Then a second order polynomial is used to fit the found lane line pixels.

#####  Deviations from standard function presented in the course

In contrast to the examples already presented in the course I applied a mask making use of a prioiri knowledge where the lane must be (while driving in the lane). I multiply this places with a value > 1. Note: The processing chain needs to be extended  for tracking this area when changing the lane. I did not do it because no lane change present in the sample data.

Also I do not run the whole code with initialization (histogram) all the time. If a have a valid lane I only search in the area around the last detected lane.

I perform a sanity check: Lane curve radius must not differ more than factor 2 from left to right and lane width must be less than 3.9m to prohibit re-initialization.

![alt text][image5]

#### 5. Calculating the radius of curvature of the lane and the position of the vehicle with respect to center.

The radius of the line is also calculted in the function `find_lane_lines`. The mathematics involved is summarized in this tutorial here.
For a second order polynomial f(y)=A y^2 +B y + C the radius of curvature is given by R = [(1+(2 Ay +B)^2 )^3/2]/|2A|.

The distance from the center is computed like this

```python
dy_left = (center-leftx[0])
dy_right = -(rightx[0]-center)
center_deviation = (dy_left+dy_right)*xm_per_pix*100/2

```

A sign of "+" means to the right, "-" means to the left

#### 6. Example image of result plotted back down onto the road such that the lane area is identified clearly

I implemented this step in `drawLaneOnImage`. Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Link to final video output.

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The pipeline performs worse when a lane change is performed because I make use of a prioire knowledge where the lines are likely present. 

Tracking is taken care of in my pipeline with the reinitialization step. Alternatively an predict and update Kalman filter could be introduced.

Also the pipeline is not very robust against shadows and curbstones guardrails etc. Additionally use cases where more than two lines (an unknown number of lines) need to be detected is difficult.

This issue could be solved by introducing separate line detectors with moving a prioiri knowledge.

A real world problem could also be: In construction zones there are white and yellow lines present and only the yellow ones are valid. A detector to take color into account would definitely be necessary to really develop a self-driving car. Also there are situations where no line at all has a validity and one would need a totally different strategy to control the car.
