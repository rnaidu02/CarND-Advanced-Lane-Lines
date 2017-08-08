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
[image7]: ./output_images/curve_radius_formula.png "Curve radius"
[video1]: ./output_images/output.m4v "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---



### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the 2nd (#2) code cell - lines 6 to 46 of the IPython notebook located in [notebook](./Advanced-Lane-Detection.ipynb) inside the function `calibCamera()`.  

I started by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image. All of the calibration images are of having 9x6 corners.  Thus, `objp` is just a replicated array of coordinates (9x6), and `objpoints` will be appended with a copy of it every time I successfully detected all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection using `cv2.findChessboardCorners()`. Just to make sure that the corners are identifed properly, I've visualized all of the calibrated images (commented the code once it is verified) and they are working fine.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result. Distorted image is on the left and the undistorted is on the right. 

![checker images][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I applied the distortion correction to one of the test images like this one. The code cell #5 is having the code to take an distorted image that produces an undistorted image. `dst = cal_undistort(img, mtx, dist)`, In this, `img` is the distorted input, `mtx` is the camera matrix coefficients and dist is the distortion coefficients array. Both `mtx` and `dis`t are derived from the camera calibration that is explained before. Distorted image is on the left and the undistorted is on the right. One of the visible fix you can observe is the overhead display on the top of the road in distorted image is parabolic and it is converted to straight in the undistorted image.

![undistorted Images][image2]


#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (thresholding steps in cell# 5 [notebook](./Advanced-Lane-Detection.ipynb)). The following functions are used to generate the binrary threshold image
* abs_sobel_thresh(): Apply Sobel kernel of size 15 and filter further with only pixel values of range 20 - 100 (Both X and Y directions)
* mag_thresh(): Scaled maginiture threshold (Take squareroot of both x and y directions). Kernel size of 15 and pixel values of range 30 - 100
* dir_threshold(): Directional threshold using the gradient of the pixels. Kernel size of 15 and the gradient values of range  0.7 - 0.13 
      
Combine the above binary images with the following logic that generates the combined binary image:
```python
combined[((gradx == 1) & (grady == 1)) | ((mag_binary == 1) & (dir_binary == 1))] = 1
```
Here's an example of my output for this step. 

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `corners_unwarp()`, which appears in lines 18 through 53 in the  3rd code cell of the IPython [notebook](./Advanced-Lane-Detection.ipynb) .  The `corners_unwarp()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcoded source and destination points in the following manner:

First I took `straight_lines1.jpg` image from the test_images folder, using gimp imaging software, located the point of the road lanes (trapezoid shape) as given in src. Since the warped image should be rectangle, I've used the points in `src` to construct `dst`  as given blo0w.

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
In python [notebook](./Advanced-Lane-Detection.ipynb) Cell #5 lines 168 - 257 has the code to find the lane-line pixels and its fit in a polynomial function (x = Ay^2+By+C form). This logic is encompassed in `locate_lanes()` function. Here is the summary of what it does.

Input: binary warped image
Output: line fit params for left and right lanes along with non-zeros pixels, identified co-ordinates for this image, and output image with blobs colored for the identified lane.

Logic:

Take the histogram of bottom half of the binary image to find out where the peak signals occur (area of the lane markings). Divide it into left half and right half to get the approximate positions of the lane positions. This is the starting point for the lane positions.
To accuratley find the lane positions (as it is curved/not straight), split the binary image into 9 slices (each slice 80 pixels)
For each slice of images:
* Find the histogram to find the lane location and the active pixels with a bondary of +/-100 pixels from left/right lane center
* Within this bounds find the pixels that are non-zero and add to the list, and recenter the location if there are more pixels are positive than the min_required (50)
* Store the active pixels in an array for both left and right lanes
* Find x and y locations for the above pixels identifed
* Try to find a 2nd order polynomial function using the above x, y co-ordinates for left and right lanes `np.polyfit(lefty, leftx, 2)`
* Store all of the above key values in an map and return for further use
    
Here is the image that is obtained after finding line fit on both right and left lanes with the binary image. This image is genrated using the exercise code with this python [notebook](https://github.com/rnaidu02/CarND-Camera-Calibration/blob/master/Finding%20the%20Lanes.ipynb)

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in cell #5 from lines #380 through #430 in my ipython [notebook](./Advanced-Lane-Detection.ipynb).

The radius of the curve is calculated inside `return_curve_radius_in_mts(ret_data_from_lines)` function defined in cell #5 inside the [notebook](./Advanced-Lane-Detection.ipynb).
* Input for this function is the data returned from `locate_lanes()` function
* The following constants are used  as factors to convert the co-ordinates to world space 
* ym_per_pix = 30/720 # meters per pixel in y dimension
* xm_per_pix = 3.7/700 # meters per pixel in x dimension
* extract x and y co-ordinates for the left lane and find the left and right lanes fit coefficients (A, B, C)
* Find the left curve and right curve radius using the following formula
      ![curve_radius][image7]
* Take the average of left and right curve radius to find the radius of the lane

The position of the vehicle relative the center is defined in function `return_distance_from_center()` in cell #5 of  my ipython [notebook](./Advanced-Lane-Detection.ipynb). Here are the details of the implementation:
* Find the bottom center of the view by taking half size of its width 
* Find the positions of left and right curve at the bottom of the view
* Take the average or the mid point of the left and right lane positions as calculated before
* Deduct mid point of lanes from mid point of the view - which is the position of the vehicle
* If the above value is -ve, then the vehivle is left of the center - if it is +ve, the nt he vehicle to the right of the center

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in cell#5 of lines 470 to 522 and 338 to 380 in my code in ipython [notebook](./Advanced-Lane-Detection.ipynb) in the functions `process_image(image)` and `return_lanes_drawn_on_image()`.  
`process_image()` function is the key function that uses all of the above functions that were discussed before and gets the left and right line co-efficients, perspective inverse matrix, radius and the position of the vehicle. It calls `return_lanes_drawn_on_image()`

In `return_lanes_drawn_on_image()` function
* For each y value, find left and right lanes x values to find the lane boundary.
* Fill in the area between the lanes with green color
* Warp the fill in image back to the original space and overlay onto the original image
* Overaly the radius and offset details onto the combined image

Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a ![link to my video result][video1]

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

Here are the issues I have faced during the implementation of the project:
* Determining the src co-ordinates for the perspective transform (unwarp/warp) functions. Initially I tried to have generic co-orginates for the four corners. However I decided to go with manual selection of the edges by looking at some of the images from the straight line images. If have more time I would like to get generalized formual to find the co-ordinates.
* Though not a big issue, but it took sometime to form the pipeline to use all of the key functions and stich them together for detecting the lanes in the video.

Some observations on the output video generated after passing through the pipeline:
* For straigh lanes, the marking of the lanes worked pretty well.
* The markings went little outside whenever the vehicle has bumpy road (at bridges). This could be due to the abrupt change of the video of the camera capture.

Where this pipeline likely to fail:
* I have tried this on the challenge video and saw this failed most of the time. My observation is that there are many curves with relatively small radius compared to the video on which the pipeline works. Since the sensor here is the camera and its range is relativly small (~20m), so any line fit params work within a distance of 20m. I guess there needs to be a different strategy to take care of transitioning from curves to straight line vice versa.


What could be done to make it robust:
* Increase the degree of ploynomial so that it fits well with the curves?
* Play with increasing the number of observation for the line fit params (currently it is 5) so that it might work relativly better in the case of curves?
