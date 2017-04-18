##Writeup Template

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



## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

###Writeup / README

###Camera Calibration

The camera calibration is used for determining the camera matrix of camera we use. For this we snap a picture of lets say a chessboard.
![chessboard](https://github.com/rohanmaan/udacity-sdcnd-P4/blob/master/Output_Images_for_writeup/calibration1.jpg)


The code for this step is contained in the first code cell of the IPython notebook located in "P4.ipynb" .  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![undistorted_chessboard](https://github.com/rohanmaan/udacity-sdcnd-P4/blob/master/Output_Images_for_writeup/undistort_output.png)

###Pipeline (single images)

1. Calibrate the camera with the input images once .
2. Undistort the image. 
3. Perform perspective transform on the undistorted image.
4. Create a binary mask with threshold of H < 100 and S > 100 n HLS
5. Create a binary mask with thresholded sobel gradients.
6. Take the combination of both masks using bitwise OR.
7. Perform sliding window search for the lane lines.
8. Calculate the polyfit polynomials of the lane lines .
9. Plot the polygon
10. Find the radius of curvature for lane lines.
11. Find the inverse perspective transform 
12. Draw on the image frame or video stream.
####1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:

![distorted_image](https://github.com/rohanmaan/udacity-sdcnd-P4/blob/master/Output_Images_for_writeup/before_distort.png)

For this the code is contained in the third cell of the IPython notebook located in "P4.ipynb" in the function created by me `cal_undistort_image()`
Below is the undistorted image we get.

![undistorted_image](https://github.com/rohanmaan/udacity-sdcnd-P4/blob/master/Output_Images_for_writeup/after_distort.png)

####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.
For creating a binary thresholded mask, I chose to apply various combinations of thresholds.
I converted the RGB image to HLS image, and then created a binary threshold mask which detected lane lien pixels with hue values(H channel in HLS image) < 100 and saturation values(S channel in HLS image)  > 100.

After this, I also created a mask which used a threshold on the absolute Sobel magnitude. I decided to go with a mask which applied different Sobel thresholds to the top and bottom halves of the image, as the top half of the image had smoother gradients due to the perspective transform. In the end, I selected any pixel with > 10 sobel magnitude at the top of the image, and > 35 at the bottom of the image.I found that the HLS threshold worked better closer to the car, and the Sobel threshold worked better further away. Thus, I chose the maximum values ie a bitwise OR of these two thresholds.
The binary thresholding code is in the function `hls_sobel_mask()` in cell no 3 in the same IPython notebook.




####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `perform_perspective_transform()`, which appears in the 3rd code cell of the IPython notebook).  The `perform_perspective_transform()` function takes as inputs an image (`img`), as well as the image shape(`img_size`).The source (`src`) and destination (`dst`) points are hardcoded in the following manner:

```python
 src = np.array([[585, 460], [203, 720], [1127, 720], [695, 460]]).astype(np.float32)
 dst = np.array([[320, 0], [320, 720], [960, 720], [960, 0]]).astype(np.float32)
```
This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 585, 460      | 320, 0        | 
| 203, 720      | 320, 720      |
| 1127, 720     | 960, 720      |
| 695, 460      | 960, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![original_warped_images](https://github.com/rohanmaan/udacity-sdcnd-P4/blob/master/Output_Images_for_writeup/warped.png)

####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

After creating the binary thresholded mask, the next task was to find and fit the lane lines. To achieve this first I created a histogram of the masked pixels in the bottom half of the image.The histogram helped in identifying the peaks in the image. In order to find where the lane line begins, biggest peak in both the left and right sides are selected as the starting point:

![lane_line_histogram](https://github.com/rohanmaan/udacity-sdcnd-P4/blob/master/Output_Images_for_writeup/histogram.png)

I then used numpy to split the image into a specified number of 10 strips or windows, and in each strip, repeated the following process:
1) Take the mean pixel x-position of the last strip (or histogram)
2) Select all masked pixels within 80 pixels of this value
3) Add the coordinates of these pixels to an array
This meant that I ended up with an array of (x, y) values for both the left and right
lane lines.
I used numpy.polyfit to then fit a quadratic curve to each lane line, which can
then be plotted on the input frame.
In order to smooth out the final curve, I took a weighted mean with the last frame’s
polynomial coefficients. (w=0.2)
The code for histogram peak detection is in the function `find_peaks()`, sliding window  is in the the fucntion `window_search()` and for lane fitting is in the function `plot_polygon()` in the code cell no 3 of the IPython notebook.

![mask](https://github.com/rohanmaan/udacity-sdcnd-P4/blob/master/Output_Images_for_writeup/hsl_sobel_mask.png) 

![fitted_curves](https://github.com/rohanmaan/udacity-sdcnd-P4/blob/master/Output_Images_for_writeup/poly_lines.png)

####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

In order to calculate the lane curvature radius, I scaled the x and y coordinates of my lane pixels and then fit a new polynomial to the real-world-sized data. Using this new polynomial, I could then use the radius of curvature formula below to calculate the curve radius in metres at the base of the image:

![formula_img](https://github.com/rohanmaan/udacity-sdcnd-P4/blob/master/WriteUp_Images/radius_curvature_formula.png)

I performed this on both the left and right lane lines, and then took the mean of these values for my final curvature value. The code for it can be found in the function `find_curvature()` in the code cell 3 of the IPython notebook.

Also ,In order to compute the position of the car in the lane, I used the lane polyfit to find the bottom position of the left and right lanes respectively. Assuming the width of the lane was 3.7 metres, I calculated the scale of the transformed image, and then used the distance between the centre of the image and the centre of the lane to calculate the offset of the car.The code for this can be found in the function `find_offset()` in the code cell 3 of the IPython notebook.

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.
The function `process_frame()` in the code cell 3 of the same IPython notebook returns the original image with the lane area between the two lanes in geen , along with the lane radius and offset shown in the top left corner.Below are some of the output images:
![out1](https://github.com/rohanmaan/udacity-sdcnd-P4/blob/master/Output_Images_for_writeup/out1.png) ![out2](https://github.com/rohanmaan/udacity-sdcnd-P4/blob/master/Output_Images_for_writeup/out2.png)
![out3](https://github.com/rohanmaan/udacity-sdcnd-P4/blob/master/Output_Images_for_writeup/out3.png) ![out4](https://github.com/rohanmaan/udacity-sdcnd-P4/blob/master/Output_Images_for_writeup/out4.png)

---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](https://www.youtube.com/watch?v=yRvr9ycVQQ4)

---

###Discussion
For this pipeline to work correctly, the binary thresholding or the masking of the lane lines should be well tuned.I found it difficult to get a good mask on the white lane lines, especially in the lighter coloured road, which meant that I had to use a combination of two masks in order to get a good result on the project video. Although if the lightning conditions varied erratically, the lane lines pixels wont be discovered effectively, leading to failing fitting of the lane lines.
In orderto make my algorith more robust I could have clustered outliers detection ie pixels in very small groups could have been discarded.
Also, normalizing the lightning conditions, or thresholding the L channel of the HLS image could have helped in achieving better lane detection.
Finally, for further improvement I could have implemented an iterative algorithm in which the lacation of the search window is based on the previous frames's search window location.This would have increased the speed of my algorithm and more robust to jittering problems.


  

