# **Finding Lane Lines on the Road** 

[original]: ./test_images/challenge4.jpg "Original"
[2-white-and-yellow]: ./test_images_output/3_yellow_and_white_challenge4.jpg	"White and Yellow"
[3-gauss]: ./test_images_output/4_gauss_challenge4.jpg "Gauss Noise"
[4-canny]: ./test_images_output/5_canny_challenge4.jpg "Canny Edge Detection"
[5-roi]: ./test_images_output/6_roi_canny_challenge4.jpg	"Canny Image with Region of Interest"
[6-interpolated]: ./test_images_output/7_interpolated_challenge4.jpg "Interpolated"
[7-overlay]: ./test_images_output/8_overlay_challenge4.jpg "Overlay with Detected Lines"

### Overview

We find lane lines on a road by applying this set of operations in the pipeline:

1. Turn image from RGB to HSL (HLS in OpenCV) for easier filtering of colors.
2. Get only white and yellow colors.
3. Apply gaussian blurr on the image to remove false positive edges, (good pre-processing step before applying canny edge detection).
4. Do canny edge detection.
5. Apply region of interest.
6. Apply hough lines transform - detect lines.
7. Remove outliers from detected lines. If video, remember line slopes over time, filter out outliers.
8. Interpolate lane lines by applying least squares regression line.

---

### Pipeline Details

Let's go over some of them in more detail, given that we have this original image, that I took from the challenge video: 

![original][]

#### HSL Conversion and Color Filtering

We convert image to HSL (HLS in OpenCV) to define yellow and white color thresholds in a more intuitive way. I found HSL color model more intuitive, and it was quite easy to define yellow color thresholds. This is the result I got after applying white and yellow color masks on the original image: 

![2-white-and-yellow][]



#### Gaussian Blur

Gaussian blur, also known as Gaussian smoothing, helps us to reduce image noise and extra details, that could serve as false positives in detecting image edges. It's a sensible step to do before applying Canny edge detection. Let's see what we get: 

![3-gauss][]

As we can see, extra details got smoothed, which should help us to have more clear edges, specifically for lanes.

#### Canny Edge Detection

To clearly find lane lines, we should first find edges in our image. We apply Canny edge detection, which finds edges by calculating image gradients and using using low and high gradient thresholds, defined by us, to find edges. More info on the algorithm can be found [here](http://fourier.eng.hmc.edu/e161/lectures/canny/node1.html).

This is what we get: 

![4-canny][]



#### Region of Interest

We know that our lanes are going to be only in a defined area on an image, as camera is fixed. Let's apply it to our image: 

![5-roi][]

#### Hough Lines Transform, Outliers and Interpolation

**Hough Lines Transform**

Hough transform takes a pixel, which is defined by (x, y) in image space, and finds all possible lines that can go through this point in hough space. In order to avoid dividing by zero problem when slope is 0, hough space lines are defined in polar coordinates with theta and rho parameters.

So one points has a bunch of lines going through it. All these lines for one point form a single sinusoid in hough space. If we do this operation for all points, group of sinusoids that intersect with each other in a defined grid form a line in image space. 

**Differentiate right and left lane lines**

After we find all lines in our image, we have to understand which of them are on left lane vs right lane. Since our y axis goes down, all lines on the left lane should have negative slope, while all lines on the right lane should have positive slope. This is as simple as that. 

**Lines Outliers**

Our assumption is that most of the detected lines will be lane lines, but some lines would be noise, having slopes that are completely different from lane lines' slopes. To filter out noise, we find standard deviation and median of our left and right slopes, and filter out all slopes that are out of 2 standard deviation range.

**Filtering Noise with Memory in Video Lane Lines Detection**

In video, to reduce false positive lines even further, we can remember all previous slopes, and then apply 2 standard deviations filtering described above, where our slopes would be all slopes we observed over video so far.

**Interpolating**

We interpolate lines by applying least squares regression line. After finding regression line, OpenCV gives us back `xv, yv, x0, y0`. `xv and xy` is a unit vector, and `x0,y0` is a point on the fitted line. Our slope is `slope = vy/vx`. 

Given that line equation is `y = slope*x + b`, we find that `b = y0 - slope*x0` . As we know our region of interest y values, we easily find x by doing: `x = (y - b) / slope`. Now we just draw the line. This is. the final result:  

![6-interpolated][]

#### Overlay

The last step would be to overlay our detected lane line with original image, using openCV `addWeighted` function: 

![7-overlay][]



### Potential Shortcomings

* Using 2 stdev range for outliers detection was a wild guess, which works, but on the video I can see lines are still "shaking" a little bit meaning that some false positive lines "drag" the interpolated line a bit.
* Region of interest is hardcoded with percentages, but it might not work completely if a camera is fixed in another position, for example. 
* My global slopes code would never reach production, as I would quickly hit out of memory. 


### Possible Improvements

* For lines outliers detection, improvement would be to find optimal c*stdev range to filter by; c=2 was an intuitive guess, which turned out. to work "ok". Other improvement would be to experiment with other approaches for outliers detection.
* Rewrite global slopes state to be defined over a fixed time window, to avoid out-of-memory. Applying time window for remembering slopes might as well provide better filtering for false positive lines.
* Try on more videos. See what goes wrong. Improve based on findings.
* Tuning parameters for canny edge detection, gaussian noise kernel, hough lines transform could yield significant improvements, I guess. I would create a tool with parameter sliders, and quickly see the output based on changed parameters. 

