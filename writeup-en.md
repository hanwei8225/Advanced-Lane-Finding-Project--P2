

**Advanced Lane Finding Project**

The project is divided into the following steps:
* The calibration matrix and distortion coefficient of lens are calculated by a set of chessboard images
* Use the calculated data to correct the original image
* Use gradient and color to change the image.
* Using ROI to crop images
* Use perspective principle to project image as aerial view
* Detect the pixels on the lane and fit them with polynomials to find the boundary
* Detect the curvature of the lane and the position of the vehicle center relative to the lane Center
* Color the interior of the lane and project it back to the original image direction
* Visual presentation and marking of curvature and head offset

[//]: # (Image References)

[image1]: ./output_images/usdistorted_image.png
[image2]: ./output_images/abs_sobel_x_y.png
[image3]: ./output_images/mag_and_dir.png
[image4]: ./output_images/combined_gradient.png
[image5]: ./output_images/hls_and_bgr.png
[image6]: ./output_images/combined_color.png
[image7]: ./output_images/combined.png
[image8]: ./output_images/masked.png
[image9]: ./output_images/warped.png
[image10]: ./output_images/field.png
[image11]: ./output_images/back_recitify.png
[image12]: ./output_images/mixed.png
[image13]: ./output_images/result.png
[image14]: ./output_images/fit.png


[video1]: ./project_video.mp4 
[video2]: ./output_images/project_video.mp4
[video3]: ./output_images/project_video_outing.mp4
[video4]: ./output_images/project_video_v2.mp4
[video5]: ./output_images/project_video_v2.mp4



## Here are the implementation ideas and results of each key point. All codes are in p2.ipynb, and there are comments and corresponding titles to distinguish them

---



### Lens calibration



#### 1. Principle

The main principle of lens calibration is to obtain the calibration matrix mit and the distortion coefficient dist of the lens through a set of chessboard diagrams (stored in / camera_cal). Through these two data, the images collected by the lens can be calibrated. The following points should be noted:

1.1 when reading, it is to read the whole group of pictures.

1.2 you need to set the number of black and white intersections of the chessboard, which is a tuple (row, column).

#### The results :

![alt text][image1]


### Image binarization

#### 1. Gradient

We use gradient to binarize the image. After the gray-scale image is used, the change rate of color depth in a certain area is used to detect the edge, so as to depict the edge of the object in the image. We use Sobel operator to monitor the gradient.

The binary image under each feature is obtained, and then a relatively clean edge image is obtained by Boolean operation.

Here are the results:

##### Absolute gradient in X and Y direction as threshold
![alt text][image2]
##### Composite gradient and angle as thresholds
![alt text][image3]
##### Gradient Boolean combination graph
![alt text][image4]



#### 2. Color space

In the  RGB color space, the Yellow Lane line can not be well detected, especially some stained and faded lane lines, which requires the use of other color space. Here I use HLS color space to extract the Yellow Lane line. After the test, it is found that the detection of the Yellow Lane in the s channel of HLS space is relatively clear, while in RGB color space, R channel can detect white Lane clearly, so we need to detect them separately and merge them


##### Detection results of s channel in HLS and R channel in RGB
![alt text][image5]
##### Two color space binary image combinations
![alt text][image6]
#### Boolean combination of gradient and binary image
![alt text][image7]


### Using Rio to crop images

For the whole image, we are only interested in one part of the image, and the rest are known useless images. In order to avoid the influence of this part of the image on our calculation, we use a mask to shield some useless parts, such as the sky, etc. Here I use a 7-edge shape for clipping, and the clipped image is as shown in the figure
![alt text][image8]


###  Perspective Transform
Because of road images are taken from our front-facing camera on a car,and we want to measure the curvature of the lines, to do that, we need to transform to a top-down view.
Here we need to obtain the coordinates of the four points on the original image and the corresponding four points after the projection transformation, so as to complete the projection.

I first used Hough space to fit the lane line, and intercepted the appropriate distance to obtain four points before projection:：
```python
src = np.float32([[ 205,720],[ 565, 470],[715,470],[1107,720]])
```
After several attempts, the positions of four points were obtained：
```python
dst = np.float32([[ 205,720],[ 205,235],[1100,235],[1100,720]])

```
The final coordinates are：

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 205, 720      | 300, 720        | 
| 565, 470      | 300, 235      |
| 715, 470     | 980, 235      |
| 1107, 720      | 980, 720        |

The following is the image after projection transformation：
![alt text][image9]




### Polynomial fitting of left and right lane lines
Using a  histogram statistics, you can get the point with the most pixels on the x-axis, which is the center line of the left and right lanes. Then take the center line as the benchmark, use a small window to scan up in turn, including the most non-0 pixel points, you can find the whole Lane line, and then use the pixel coordinates to fit a conic, that is the left and right center of lane：
![alt text][image14]


### Calculate the curvature of the lane and the distance of the head from the centerline
The curvature of the lane is calculated by the formula, which is explained here：https://www.intmath.com/applications-differentiation/8-radius-curvature.php，在python中的实现方法为：

```python
    road_left = (left_fit[0] * (y_eval ** 2)) + (left_fit[1] * y_eval) + left_fit[2]
    road_right = (right_fit[0] * (y_eval ** 2)) + (right_fit[1] * y_eval) + right_fit[2]
   
```

Considering the corresponding relationship between pixels and actual distance, the algorithm is changed as follows
```python
    ym_per_pix = 30/720
    xm_per_pix = 3.7/700
    left_curverad = ((1 + (2*left_fit_cr[0]*y_eval * ym_per_pix + left_fit_cr[1])**2)**1.5) / np.absolute(2*left_fit_cr[0])
    right_curverad = ((1 + (2*right_fit_cr[0]*y_eval*ym_per_pix + right_fit_cr[1])**2)**1.5) /np.absolute(2*right_fit_cr[0])

```
At the same time, you can calculate the difference between the center of the camera (the default camera is installed in the car center) and the center of the lane line, and then convert it to distance according to the scale

### Use color overlay for lanes

After finding the lane, we fill the whole area with green, as shown in the figure：
![alt text][image10]

### Invert the overlay to the original view
Because all our coverage is done on the aerial view, we need to reverse the aerial view to the normal perspective：
![alt text][image11]


### Merge overlay and original
Because the overlay is only the information we extract, in order to facilitate presentation, we need to cover the overlay with translucency on the original image：
![alt text][image12]

### Show curvature and distance
We need to show the curvature and the head off distance to the picture：
![alt text][image13]


At this point, the processing of a single picture is completed, and then the video will be processed. The processing of video can be equivalent to the processing of multiple pictures


## Video processing

### Pipeline (video)

The above image processing is integrated. See the code in p2.ipynb for details

### First processing
[video2]
It can be seen that there will be problems in the lane of complex terrain detection. After troubleshooting, it is found that the lane projection is too wide when the projection is done, resulting in one side of the lane line exceeding the screen when the curve is made. After adjusting the projection parameters, the second processing is carried out：

### Second processing
[video4]

The result of the second processing is not less than that of the first one, but in the case of some interference, the problem of individual frame detection failure will occur.

### Third processing
This time, some attempts have been made. The reason for the previous detection failure is that one side of the lane line is broken, and sometimes the fitting is not accurate if there are too few pixels.
But we know that the lane line is parallel, so I choose one side with more pixels detected as the reference, the other side is only obtained by translating the reference side, which is relatively good after testing, but there may be more unknown problems
[video5]

---

## Discussion
The places where the detection fails are mainly the wear of lane line, or the detection of too few pixels caused by lane line fracture, and some of them are due to the interference caused by some road color switching. Therefore, I think the lifting places mainly include the following aspects:
1. More reliable algorithm for detecting wear and lane lines in various light conditions
2. Eliminate the interference caused by road color switching
3. Using my third video processing method, as long as one lane line boundary is reliable, we can use translation to fit two lane lines.
