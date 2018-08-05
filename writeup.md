## Writeup 

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

[image1]: ./examples/undistort_out.png "Undistorted"
[image2]: ./examples/undistorted_example.png "Road Transformed"

[image3]: ./examples/grey_filter.png "Grey filtering example"
[image4]: ./examples/hls_filter.png "hls filtering"
[image5]: ./examples/hls_grey_combined.png "hls grey combination"
[image6]: ./examples/hls_grey_grad_combined.png "color-gradient combination"
[image7]: ./examples/warped.png "Warped image"
[image8]: ./examples/found_centers.png "Sliding widows"
[image9]: ./examples/polynom_fit.png "Polynm fitted"
[image10]: ./examples/line_decorated_example.png "Decorated example"
[video11]: ./result.mp4 "Output"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README


### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "./solution.ipynb".  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1.An example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (see lines Color & gradient filters).  
I use combination of filters for hls and hsv spaces (s and v channel respectively).
In combination with sobel operation threshold, magnitude and direction of the gradient together with the color spaces I get the optimal filtering to my mind. 
Here's an example of my output for this step. 

Greyscaing and filtering:

![grey filtering][image3]

Converting to HLS colorspace and filtering S-channel:
![hls filter][image4]

Combining greyscaling with HLS filtering:

![][image5]

Combination of color and graident-based filters:
![alt text][image6]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp(img, src, dst)`, which appears in lines of ipyton notebook under Perspective transform.  The `warp()` function takes as inputs an image (`img`), as well as source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
src_points = np.float32([[179, 720], [1140, 720], [740, 460], [550, 460]])
dest_points = np.float32([[350, 720], [977, 720], [977, 0], [350, 0]])

```



I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

The reverse function is called `unwarp(img, src, dst)` and uses iverted matrix:
```python
Minv = cv2.getPerspectiveTransform(dst, src)
```

![Warped image][image7]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

To identify the lines, I build the boxes around the window centroids with the maximum convolution, you can see it in the class `class WindowFinder` in the section Find sliding windows of example notebook.
I use the halfs, not the quarters of my transformed image, since it performs better, when there is another car close to the lane of the ego-car.
Then after identifying the boxes, I fit the polynomial of the second degree to its centers with 

```python
#left line
left_fit = np.polyfit(result_y, leftx, 2)
left_fitx = np.array(left_fit[0]*lane_y*lane_y + left_fit[1]*lane_y + left_fit[2], np.int32)
```
Here are the boxes drawn (note please, that left line is sick and you can't see all boxes, but I don't want to make the window width too big).

![Sliding Windows][image8]

For fitting polynom I use `np.mean` of centroids 2 previous frames and the current one. For this I use the class `class WindowFinder`and the function `def pipeline(init_image, right_line, left_line)`.
Initially right_line and left_line are empty lines.

![Polynom fit][image9]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in lines of function `get_final_image`:

```python
#calculate curvature radius by formular based on polynomial coefficients, take left turn
left_fit_x_r= np.array(left_fitx, np.float32)*curve_centers.meters_pp_x
left_fit_y_r= np.array(lane_y, np.float32)*curve_centers.meters_pp_y
polynomila_fit_real = np.polyfit(left_fit_y_r, left_fit_x_r, 2)
    left_curverad = ((1 + (2*polynomila_fit_real[0]*lane_y[-1]*curve_centers.meters_pp_y + polynomila_fit_real[1])**2)**1.5) / np.absolute(2*polynomila_fit_real[0])
```

I calculate the radius of the left line, since the curve in the video goes right.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines of `get_final_image` function.  Here is an example of my result on a test image:

![alt text][image10]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./result.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I had a lot of problems with the correct filtering, since if there was another car next to the ego car, it did produce some artefacts, that I couldn't get rid of. 
In my second submittion I changed the portion of the picture, where I take a histogram of, thus I could start from the right lower point. However, sometimes, there is no any points visible of the right line in the bottom of the frame.  
What could probably help is to start als from the top of the frame, or from both sides and try to "meet" in the middle.

Stil, I think the worst what can happen, is the long yellow truck at the right side of the ego car, that is very close to our lane.
I tried different color spaces and the best combination was to use hsl and lab color spaces. I tried different approaches to identify the yellow line, this combination was the best one for me.
I spent also a lot of time to find the corrrect number of prevoius frame to take a mean from. Then as I was told in my review, I discarded the lines, where the distance to the center is very high(> 70 cm) and just kept the median of prevoius two identified lines.
There was many points in time, when the performance was drastically droping while trying out new approaches.
