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

[image1]: ./output_images/calibration1_undist.png "Undistorted"
[image2]: ./test_images/test1.jpg "Road Transformed"
[test1]: ./test_images/test1.jpg "Road Transformed"
[image3]: ./examples/binary_combo_example.jpg "Binary Example"
[image4]: ./examples/warped_straight_lines.jpg "Warp Example"
[image5]: ./examples/color_fit_lines.jpg "Fit Visual"
[image6]: ./examples/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"
[image_area_peak]: ./other_images/histogram_area_instead_of_peak_point.png "histogram_area_instead_of_peak_point"
[image_sobel_combined]: ./output_images/image_combined_4.jpg "sobel find combined"
[perspective]: ./output_images/perspective/test6.jpg.png "perspective"
[final_lane_find]: ./output_images/lane/combine_straight_lines1.jpg.png "final lane find"
[gif_demo]: ./other_images/SelfDrvingCar_AdvancedLaneFind.gif "gif_demo"

##Before Start
The whole code structure has been designed for fast debug

Run **main.py**, it will create target video, there is a image index in very frame, if you see any frame not right
you can just pick up the image from **video_images** folder, put this file into 
**test_images**, run whole unit test in **/test/sobel_test.py**, it will create images for every step so that you can 
do a detailed check.

![alt text][gif_demo]

###Camera Calibration

The code for this step is contained in camera_calibrate.py.


I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners 
in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the 
object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, 
and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners 
in a test image.  
`imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane 
with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion 
coefficients using the `cv2.calibrateCamera()` function.  
I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

I choose calibration1.jpg as a test image to un-distortion, also saves camera matrix and distortion into a file 
called "camera_calibration_pickle.p" for later usage.

```python
img = cv2.imread('./camera_cal/calibration1.jpg')
img_size = (img.shape[1], img.shape[0])
camera_matrix, distortion = calibrateCamera(img_size)
save_camera_calibration(camera_matrix, distortion)
undistort(camera_matrix, distortion, img, './output_images/calibration1_undist.png')
```
![alt text][image1]

###Pipeline (single images)

####1. Binary image generation with color and gradient thresholds 
I used a combination of color and gradient thresholds to generate a binary image (thresholding steps at lines # through # in `thresholding.py`).  
Here's an example of my output for this step.
To generate below images, please run `test_threshold` method under `pipe_line_test.py` 
![alt text][test1]
![alt text][image_sobel_combined]

```python
def pipeline(img):
    img = np.copy(img)
    gray_image = cv2.cvtColor(img, cv2.COLOR_BGR2GRAY)
    hls_image = cv2.cvtColor(img, cv2.COLOR_BGR2HLS).astype(np.float)

    combined = combine_with_or(
        abs_sobel_thresh(gray_image, orient='x', sobel_kernel=25, thresh=(50, 150)),
        combine_with_or(
            *bgr_channel_threshold(img, b_thresh=(220, 255), g_thresh=(220, 255), r_thresh=(220, 255))
        ),
        combine_with_and(
            hls_channel_threshold(hls_image, s_thresh=(170, 255))[2],
            abs_sobel_thresh(gray_image, orient='x', sobel_kernel=5, thresh=(10, 100))
        )
    )
    return combined

```

####2. Perspective transform
All perspective transform method are located in `perspective_transform.py`
`def perspective_transform(img, perspective_transform_matrix)` will transform given image with transform matrix
`def inversion_perspective_transform(img, invent_perspective_transform_matrix)` will inversion that process 

![alt text][perspective]

```python
img = cv2.imread(fname)
matrix, invent_matrix = calculate_transform_matrices(img.shape[1], img.shape[0])
perspective_img = perspective_transform(img, matrix)
save_image(perspective_img, '../output_images/perspective/{}.png'.format(fname))
```

####3. Fit positions with a polynomial from lane-line pixels

method `find_line_with_slide_window` in `main.py` apply slide window into search area and return
all points found, we then fit positions with polynomial in method `process_image`

#####3.1 Cached Search Base Position
once we know roughly where are two lanes, we can cache it and next frame could search in the similar area. 
the code in main.py method named `_line_search_base_position`

If a last know fit has been found, the fit for next frame will only search between last fit +- 120

```python
@staticmethod
def _line_search_base_position(histogram,
                               last_know_leftx_base=None, last_know_rightx_base=None, peak_detect_offset=120):
    if last_know_leftx_base is None or last_know_rightx_base is None:
        midpoint = np.int(histogram.shape[0] // 2)
        leftx_base = np.argmax(histogram[:midpoint])
        rightx_base = np.argmax(histogram[midpoint:]) + midpoint
    else:
        left_start, left_end, right_start, right_end = LaneFinder._range(
            len(histogram), last_know_leftx_base, last_know_rightx_base, peak_detect_offset)
        leftx_base = np.argmax(histogram[left_start:left_end]) + left_start
        rightx_base = np.argmax(histogram[right_start:right_end]) + right_start

    return leftx_base, rightx_base
```
the test case will explain how this function works.
in test case 1, if there is no last know line position, the search will start from middle point of histogram, which is 
the index 1 and 4.

In test case 2, it will search across last know point +- peak_detect_offset
```python
def test_line_search_base_position_should_find_middle_point_if_no_last_knowledge(self):
        histogram = np.array([1, 2, 1, 3, 4, 3])
        left, right = LaneFinder._line_search_base_position(histogram, None, None)
        self.assertEqual(left, 1)
        self.assertEqual(right, 4)

def test_line_search_base_position_should_find_peak_point_near_last_know_position(self):
    histogram = np.array([1, 4, 1, 2, 1, 3, 4, 3, 5, 3])
    left, right = LaneFinder._line_search_base_position(histogram, None, None)
    self.assertEqual(left, 1)
    self.assertEqual(right, 8)
    left, right = LaneFinder._line_search_base_position(
        histogram, last_know_leftx_base=4, last_know_rightx_base=6, peak_detect_offset=1)
    self.assertEqual(left, 5)
    self.assertEqual(right, 6)
    left, right = LaneFinder._line_search_base_position(
        histogram, last_know_leftx_base=4, last_know_rightx_base=9, peak_detect_offset=2)
    self.assertEqual(left, 6)
    self.assertEqual(right, 8)
```
#####3.2 Abnormal detection and auto-correction
LandFinder class will save last 5 "normal" fit, they new_left_fit and new_right_fit will
compare with last normal fit, if they didn't off the center too much, we think the new fit
is "normal" and system will allow to use it to following process, otherwise new fit will discard
and last know "normal" fit will used for current frame.
Only 5 frames maximum allocated to replace the new fit, if 5 frames still consider as "abnormal"
this "abnormal" fit will still been used.

```python
def _compare_and_get_best_fit(self, new_left_fit, new_right_fit):
    if self._is_normal_fit(new_left_fit, new_right_fit):
        self.last_left_fits.append(new_left_fit)
        self.last_right_fits.append(new_right_fit)
        if len(self.last_left_fits) > self.MAX_REUSED_FRAME:
            self._drop_oldest_cached_fits()
        return new_left_fit, new_right_fit
    else:
        self._drop_oldest_cached_fits()
        left, right = self._recent_fits()
        print("fit abnormal, dropped. cached left{} cached right{} new left:{} new right:{}".format(
            left, right, new_left_fit, new_right_fit
        ))
        if left is None:
            print("Cache Fit exhausted, accepting new fit")
            return new_left_fit, new_right_fit
        return left, right
```

####4. Calculate the radius of curvature of the lane and the position of the vehicle

I did this in method `_calculate_radius` in `main.py`

####6.Plotted back down onto the road

this has been done in method `apply_fit_to_road` in `main.py`, please see image below

![alt text][final_lane_find]

---

###Pipeline (video)

Layout:

| Original Input                   | Perspective transform + Histogram + PloyFit |
| ---------------------------------|---------------------------------------------|
| Plotted Back Down Onto the Road  | Information                                 |
                                                                 
Video of my project output

<a href="http://www.youtube.com/watch?feature=player_embedded&v=y2jeJk-2F4I" target="_blank">
<img src="http://img.youtube.com/vi/y2jeJk-2F4I/0.jpg" alt="UDacity Sample Data" width="800" height="600" border="10" /></a>


---

###Discussion

####1. Historgram maybe shouldn't look at the highest point, look at the highest area. As show in below 
![alt text][image_area_peak]

####2. The implement here are very sensitive to lights
Maybe add auto exposure feature would help, for example analysis the image and increase exposure if it's too dark and vice versa

####3. Not able to handle change lane

####4. A yellow / white car in front would be very challenge
In this case the histogram will pick up that car instead of lane lines


