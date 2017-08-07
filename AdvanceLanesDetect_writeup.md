## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

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

[image1]: ./output_images/undistorted_checker.png "undistortedchecker"
[image2]: ./output_images/undistorted_image.png "UndistortedReal"
[image3]: ./output_images/combined_gradient_binary.png "Binary Example"
[image4]: ./output_images/warped_image_sample.png "Warp Example"
[image5]: ./output_images/find_lanes.png "Fit Visual"
[image6]: ./output_images/lane_indentified.png "Output"
[video1]: ./output_images/output.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---



### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the 2nd (#2) code cell of the IPython notebook located in "./Advanced-Lane-Detection.ipynb" .  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image. All of the calibration images are of having 9x6 corners.  Thus, `objp` is just a replicated array of coordinates (9x6), and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection using `cv2.findChessboardCorners()`. Just to make sure that the corners are identifed properly, I've visualized all of the calibrated images (commented the code once it is verified) and they are working fine.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result. Distorted image is on the left and the undistorted is on the right. 

![checker images][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one. The code cell #5 is having the code to take an distorted image that produces an undistorted image. `dst = cal_undistort(img, mtx, dist)`, In this img is the distorted input, mtx is the camera matrix coefficients and dist is the distortion coefficients array. Both mtx and dist are derived from the camera calibration that is explained before. Distorted image is on the left and the undistorted is on the right. One ofthe visible fix is the overhead display on the top of the road in distorted image is parabolic and it is converted to straight in the undistorted image.

![undistorted Images][image2]


#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps in cell# 5 [notebook](./Advanced-Lane-Detection.ipynb)). The following functions are used to generate the binrary threshold image
      abs_sobel_thresh(): Apply Sobel kernel of size 15 and filter further with only pixel values of range 20 - 100 (Both X and Y directions)
      mag_thresh(): Scaled maginiture threshold (Take squareroot of both x and y directions). Kernel size of 15 and pixel values of range 30 - 100
      dir_threshold(): Directional threshold using the gradient of the pixels. Kernel size of 15 and the gradient values of range  0.7 - 0.13 
      
Combine the above binary images with the following logic that generates the combined binary image:
```python
combined[((gradx == 1) & (grady == 1)) | ((mag_binary == 1) & (dir_binary == 1))] = 1
```
Here's an example of my output for this step. 

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `corners_unwarp()`, which appears in lines 18 through 53 in the  3rd code cell of the IPython [notebook](./Advanced-Lane-Detection.ipynb) .  The `corners_unwarp()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

First I took `straight_lines1.jpg` image from the test_images folder, using gimp software, located the point of the road lanes (trapezoid shape) as given in src. Since the warped image should be rectangle, I've used the points in `src` to construct `dst`  as given blo0w.

```python
src = np.float32([ [525,500], [764,500],  [1040,680], [261,680]])
dst = np.float32([ [261,500], [1040,500],  [1040,680], [261,680]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 525, 500      | 261, 500        | 
| 764, 500      | 1040, 500      |
| 1040, 680     | 1040, 680      |
| 261, 680      | 261, 680        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

Just to make sure that the warped image can be unwarped back and overlapped on to the original image (to overlay the lane detection), I have unwarped back and found that it is working fine as shown in the following figure.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?
In python [notebook](./Advanced-Lane-Detection.ipynb) Cell #5 lines 168 - 257 has the code to find the lane-line pixels and its fit in a polynomial function (x = Ay^2+By+C form). This logic is encompassed in `find_lanes()` function. Here is the summary of what it does.

Input: binary warped image
Output: linefit params for left and right lanes along with non-zeros pixels, identified co-ordinates for this image, and output image with blobs colored for the identified lane.

Logic:

Take the histogram of bottom half of the binary image to find out where the peak signals occur (identify the lanes). Divide it into left half and right half to get the approximate positions of the lane positions. This is the starting point for the lane positions.
To accuratley find the lanes, split the binary image into 9 slices (each slice 80 pixels)
For each slice of images:
    Find the histogram to find the lane location and the active pixels with a bondary of +/-100 pixels from left/right lane center
    Within this bounds find the pixels that are non-zero and add to the list, and recenter the location if there are more pixels are positive than the min_required (50)
    Sore the active pixels in an array for both left and right lanes
    Find x and y locations for the above pixels identifed
    Try to find a 2nd order polynomial function using the above x, y co-ordinates for left and right lanes `np.polyfit(lefty, leftx, 2`
    Store all of the above key values in an map and return for further use
    
Here is the image that is obtained after finding line fit on both right and left lanes with the binary image. This image is genrated using the exercise code with this python [notebook](https://github.com/rnaidu02/CarND-Camera-Calibration/blob/master/Finding%20the%20Lanes.ipynb)

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in cell #5 from lines #380 through #430 in my ipython [notebook](./Advanced-Lane-Detection.ipynb)

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
