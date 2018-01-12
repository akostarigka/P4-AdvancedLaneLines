# **Advanced Lane Finding**

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

---

[//]: # (Image References)

[image1]: ./output_images/ChessboardUndistorted.png "Chessboard Undistorted Example"
[image11]: ./output_images/RoadUndistorted.png "Undistorted Camera Image"
[image2]: ./output_images/StraightLines1.jpg "Road Original Image"
[image3o1]: ./output_images/SobelOperator.png "Sobel Operator"
[image3a1]: ./output_images/SobelX.png "Sobel x-axis"
[image3a2]: ./output_images/SobelY.png "Sobel y-axis"
[image3a3]: ./output_images/SobelMag.png "Sobel Magnitude of Gradient"
[image3a4]: ./output_images/SobelDir.png "Sobel Direction of Gradient"
[image3]: ./output_images/CombinedGrad.png "Combined Gradient Binary Example"
[image31]:./output_images/AllThresholds.png "Combined Thressholds"
[image4]: ./output_images/Warped.png "Warped Image Example"
[image5]: ./output_images/WarpedWtLanelines.png "Warped Image Example wt Lane Lines"
[image6]: ./output_images/PipelineOnImages.png "Output Images"
[image7]: ./output_images/OutputVideo.png "Output Video"
[video1]: ./output_video.mp4 "Video"

The project includes the following files:
* **P4.ipynb** containing the project pipeline in jupyter notebook format
* **output_images** folder including example images from each stage of the pipeline
* **README.md** as a writeup report summarizing the results and providing description of each output image
* **output_video.mp4**

---

### Camera Calibration using chessboard images

A chessboard is the most suitable means of calibrating a camera because its regular high contrast pattern makes it easy to detect automatically. In this project, the calibration was performed using the chessboard images provided in the [Udacity Project Repository](https://github.com/udacity/CarND-Advanced-Lane-Lines). After retrieving the `objpoints` and `imgpoints` from these calibration images, this information would be used to calibrate our camera.

We started by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. We assumed that the chessboard is fixed on the (x, y) plane at z=0, such that the object points were the same for each calibration image.  Thus, `objp` was just a replicated array of coordinates, and `objpoints` would be appended with a copy of it every time all chessboard corners were successfully detected in a test image.  `imgpoints` would be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

The output `objpoints` and `imgpoints` were used to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  This distortion correction was applied on a chessboard test image using the `cv2.undistort()` function to obtain the result:

![alt text][image1]

---

### Pipeline Description ( function `Final_Pipeline()`)

* **Image Processing** ( function `Image_Processing()`)
    1. Undistort using camera calibration matrix ( function `cal_undistort()`)
    2. Apply thresholding functions combination
        - Gradient threshold (Sobel operator in x,y-directions, magnitude of the gradient, direction of the gradient)
        - RGB and HLS color space thresholds
    3. Apply region of interest mask
    4. Perform perspective transform ( function `warp_image()`)
* **Lane line detection** ( functions `find_lane_lines_ini(), find_lane_lines()`)
* **Lane curvature calculation** ( function `calc_curvature()`)
* **Lane area drawing** ( function `draw_lane_area()`)
* **Lane data printing**  ( function `print_lane_data()`)

### Image Processing ( function `Image_Processing()`)

#### 1. Correcting for Distortion ( function `cal_undistort()`)

The first step in analyzing camera images is to undo any distortion so that we can get correct and useful information out of it. To un-distort the camera image we used the object points (`objpoints`) and image points (`imgpoints`) calculated using the 20 camera calibration images discussed in the previous section.

Employing the function `cal_undistort()` we get the following result for a road test image.

![alt text][image11]

#### 2. Thresholding

To generate a binary image where the lane lines are clearly visible we tried various combinations of gradient and color thresholds.

##### Gradient threshold/Sobel operator

We initially employed the [Sobel Operator](https://en.wikipedia.org/wiki/Sobel_operator) to detect image edges.
This operator was introduced by Irwin Sobel and Gary Feldman in a talk on ["Isotropic 3x3 Image Gradient Operator"](https://www.researchgate.net/publication/239398674_An_Isotropic_3_3_Image_Gradient_Operator) presented at the Stanford Artificial Intelligence Project (SAIL) in 1968.

The operators for Sobel in x and y direction for a kernel size of 3 are expressed as follows:

![alt text][image3o1]

Applying the Sobel Operator in x,y-directions with threshold = (25, 255), we get on a test image:

* Absolute Sobel Operator on x-direction ( function `abs_sobel_thresh()`)
![alt text][image3a1]
* Absolute Sobel Operator on y-direction ( function `abs_sobel_thresh()`)
![alt text][image3a2]

If we apply a threshold = (25, 255) to the overall magnitude of the gradient, in both x and y, we get:

* Magnitude of the Gradient threshold ( function `mag_thresh()`)
![alt text][image3a3]

If we apply a threshold = (π/4, π/2) to the direction of the gradient (arctangent of the y gradient divided by the x gradient), we get:
* Direction of the Gradient threshold ( function `dir_threshold()`)
![alt text][image3a4]

Combining all of the above thresholds we get:
* Combined gradient thresholds
![alt text][image3]

##### RGB and HLS Color Spaces threshold

To further improve lane detection we also employed color thresholds. Such thresholds give us information about an image that we lose when converting for example to grayscale (e.g. we get to identify the yellow lines as well).

After trying various combinations we arrived at the following thresholds on RGB and HLS color spaces:

* RGB color space ( function `rgb_select()`):  we employed threshold > 150 to R-channel and G-channel.
* HLS color space ( function `hls_select()`): we employed S-channel threshold = (100, 255)  and L-channel threshold = (120, 255).  

##### Final gradient and color thresholds
To generate the binary image to be used for lane detection we finally used a combination of the aforementioned color and gradient thresholds ( function `all_thresholds()`). The result can be seen in the following figure, where noise has been minimized and all lines were distinctly depicted.

![alt text][image31]

#### 3. Region of interest mask

In order to focus on the portion of the image that is useful for the lane line detection, we defined a **region of interest** in each one and created a mask to exclude the rest of it. This was done by constructing a rectangular with the appropriate end points:

```python
region_of_interest_vertices = np.array([[0 , height-1],
        [width/2, height/2], [width-1, height-1]], dtype=np.int32)
```

where `height` and `width` are the height and width of the image respectively.

#### 4. Perspective transform ( function `warp_image()`)

To  have a view of the lane as from above and correctly calculate its curvature, we had to rectify each image to a "bird-eye view". For this reason we used a perspective transform through the function `warp_image()`.

The `warp_image()` function takes as inputs an image `img`, as well as source (`src`) and destination (`dst`) points and returns a warped image `warped_img`. It uses OpenCV functions `cv2.getPerspectiveTransform()` and `cv2.warpPerspective()` to calculate the transform that maps the points in the original image to the warped image with a different perspective and apply it to the original image `img`.

The source and destination points, that define the perspective transform, were chosen manually and hardcoded in the following manner:

```python
src = np.float32([(0.564*img.shape[1],0.65*img.shape[0]),
                (0.867*img.shape[1],img.shape[0]),
                (0.172*img.shape[1],img.shape[0]),                   
                (0.445*img.shape[1],0.65*img.shape[0])])
dst = np.float32(([(0.8*img.shape[1],0),
                (0.8*img.shape[1],img.shape[0]),
                (0.2*img.shape[1],img.shape[0]),
                (0.2*img.shape[1],0)]))
```

This resulted in the following source and destination points:

| Source        | Destination   |
|:-------------:|:-------------:|
| 721.9, 468    | 1024, 0       |
| 1109.7, 720   | 1024, 720     |
| 220.16, 720   | 256, 720      |
| 569.6, 468    | 256, 0        |

To verify that perspective transform worked as expected, we plotted the `src` and `dst` points onto a test image and its warped counterpart. It is obvious that the lines appear parallel in the warped image, as expected.

![alt text][image4]

### Lane line detection

#### Locate lane lines and fit a polynomial

After retrieving an undistorted, thresholded, warped binary image we used the following methods to detect the lane lines (code mainly taken from the [lecture notes](https://classroom.udacity.com/nanodegrees/nd013/parts/fbf77062-5703-404e-b60c-95b78b2f3f9e/modules/2b62a1c3-e151-4a0e-b6b6-e424fa46ceab/lessons/40ec78ee-fb7c-4b53-94a8-028c5c60b858/concepts/c41a4b6b-9e57-44e6-9df9-7e4e74a1a49a)):

* **Identify peaks in a Histogram and use Sliding Window ( function `find_lane_lines_ini()`):** this method was used as a starting point, to identify where the lane lines are and fit a polynomial. This why it is called `find_lane_lines_ini()`. Once we had the location of a valid lane line pair we could use the function `find_lane_lines()`.
* **Search in a margin around the previous line positions ( function `find_lane_lines()`):** this method was used only after we had identified the lane lines of a frame. In this case, instead of doing a "blind search", we skipped the sliding window step and searched in a margin around the previous line positions. This way we would improve speed and provide a more robust method for rejecting outliers.

Example of an original binary image, a warped image and a warped image with printed lane lines (yellow color) can be seen in the following figure:

![alt text][image5]

### Lane curvature calculation/data printing

Τhe curvature of the identified lane lines was calculated using the the function `calc_curvature()` that contains code based on the [lecture notes](https://classroom.udacity.com/nanodegrees/nd013/parts/fbf77062-5703-404e-b60c-95b78b2f3f9e/modules/2b62a1c3-e151-4a0e-b6b6-e424fa46ceab/lessons/40ec78ee-fb7c-4b53-94a8-028c5c60b858/concepts/2f928913-21f6-4611-9055-01744acc344f). Τo measure the radius of curvature of the polynomial curve f(y)=Ay^2 + By + C closest to the vehicle, we used the y curve values corresponding to the bottom of each image into the formula: R_curve = ( (1+(2 A y+B)^2)^3/2 ) / |2Α|.

Finally, to transform the derived values from pixels into real world space units we used the conversions:

```python
ym_per_pix = 30/720  # meters per pixel in y dimension
xm_per_pix = 3.7/700 # meters per pixel in x dimension
```

The curvature results vary from 1km to 3km in normal curvature lanes. However one will notice that in case of straight lines the curvature tends to get very high values. This is explained if one thinks of the curvature as the radius of a circle. Straight lines would correspond to line segments of a circle with really big radius.

Radius results as well as position of the vehicle with respect to the center of each frame are printed in each picture using the function `print_lane_data()` and can be seen in the example figures below.   

### Final pipeline tested on images

![alt text][image6]

### Final pipeline tested on video

Finally the pipeline was successfully tested on the project video. A link of the output video can be found [here](https://github.com/akostarigka/P4-AdvancedLaneLines/blob/master/output_video.mp4).

![alt text][image7]

#### Problems / Issues

During the implementation of the project, the following problems/issues were encountered:

* In the output video, a "jump around" of lines within frames was noticed in some instances. This could be avoided by taking the average over *n* past measurements and smoothening the results to obtain a cleaner picture.
* In case of straight lines the curvatures appear to be abnormally high. Even though this can be explained (straight lines correspond to line segments of a circle with really big radius), it provides distracting information and should be considered further.
* The implementation could be more robust with the use of a `Line()` class as suggested in the lecture notes. This way we could keep track of parameters needed from frame to frame without having to use global lists like `l_list` and `r_list` as implemented in the project pipeline.

### Conclusion/Discussion

In this project we have employed some advanced techniques to detect and plot lane lines on a figure and  subsequently on a video. To benefit the most out of each image we un-distorted them using a camera calibration matrix obtained from a set of chessboard calibration images, then we thresholded them using a combination of gradient (Sobel Operator) and color space thresholds and in the end we performed a perspective transform to warp them into a "bird-eye view". In the resulting thresholded warped image we were finally able to detect the lane lines, fit a polynomial and calculate their curvature. The pipeline was tested both on images and a video and could successfully demonstrate the effectiveness of the approach.     
