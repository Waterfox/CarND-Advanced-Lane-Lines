## Udacity CarND P4 Advanced Lane Lines Project

The main file for this project is P4_0.4.ipynb. This notebook loads in pickle files and has functions copied in from other notebooks where the processes for each step of the project were developed and tuned. Tweaks to each step may have been added in the final P4_0.4 file. 

The final project output file is proj6.mp4
and is also on youtube:
https://www.youtube.com/watch?v=-94d1Z6pmCs

##Camera Calibration

Camera_calibrate_tune0.1.ipynb uses the chessboard calibration images to calculate the maxtrix and coefficients used to undistort the camera images. The variables are saved into camera_cal.p. An example of orginal and undistored images of a chessboard are included in the notebook and shown below:

![alt tag](/results_imgs/uncalibrated.png)
![alt tag](/results_imgs/calibrated.png)

##Color and Gradient Thresholding


Pipeline_tune_0.1.ipynb illustrates the process used to come up with the pipeline function used for color and gradient thresholding of the images. Different channels of color transforms are vizualized to determine effectiveness at highlighting features of lane lines. Ultimately the following channels were used:

x-gradient of the S channel of the HLS color space
threshold of the S channel of the HLS color space
threshold of the U channel of the LUV color space
threshold of the Z channel of the XYZ color space

These values were ORed or ANDed to create the final output image.
Throughout the process, many parameters were changed before settling on this approach. 

An example of the output of this stage is included below

![alt tag](/results_imgs/pipeline_output.png)

##Perspective Transform

Perspective_tune_0.1.ipynb is where the process for transforming an image into a bird's eye view was developed. It was assumed that the file /test_images/solidWhiteRight.jpg contained straight lines where the vehicle was roughly in the center of the image. Four transformation points were manually selected on the image and used to calculate the perspective transform. This image was a different size than the other camera data and so the point locations were scaled based on the image size. Points were selected so that the lane lines would show up as vertical and straight in the transformed image. This transformation depends highly on the assumption that the lanes in the solidWhiteRight.jpg file were centered and straight.

The perspective transform was calculated for an image with the same 1280x720 dimensions as the camera data. The transform matrix was saved into M_Minv.p

Examples of the input transform points, shown as red dots are included here:

![alt tag](/results_imgs/perspective_points.png)

The output points were selected as a rectangle. 100pixel padding was used and the output image size was selected to be 800x600pixels

Examples of the transformed perspective and transformed back perspective for curved lanes are shown in:

![alt tag](/results_imgs/perspective_transformed.png)
![alt tag](/results_imgs/perspective_transformed_back.png)


##Curve Fitting

The algorithm for finding points on the thresholded and warped image was developed in Curve_fit_tune_0.1.ipynb.

Similar to the histogram method suggested, a matched filter process was used to detected line locations. To do this: a square, 20 pixel wide window function was convolved across the x-axis lines of the flat image. This was done over 8 (parameter: win_y) lines and summed together. Peaks were recorded using np.argmax for outputs that met a detection threshold: peak_thresh = 10. The process was then advanced down the y axis of the image in steps win_y/2 until the end of the image was reached.

The window function used for convolution and an example output line taken across row 380 of the input image are shown here:

![alt tag](/results_imgs/matched_filter_img.png)
![alt tag](/results_imgs/window.png)
![alt tag](/results_imgs/match_filter_line.png)

A second order polynomial was fit to each of these points based on which side of the image they fell on. Perhaps something better could have been done here, such as adding a point to a group (left or right lane) based on proximity to other points. The output points,fitted curves and coresponding polygones are given in the following images:

![alt tag](/results_imgs/matched_filter_points.png)
![alt tag](/results_imgs/fitted_curves.png)
![alt tag](/results_imgs/fitted_curves2.png)
![alt tag](/results_imgs/lines_poly.png)

At this point the polygon output was transformed back to camera space and combined with the camera image.


## Tracking lines in the videos

In P4.04.ipynb, a Line class was created to track a moving average of the line fit, the x,y-points of the line and the curvature as well as the confidence in the state of detection of the line. Objects were created for both the Left and Right lines and updated each time a line was successfully detected.

The general process for tracking lines from a video file involved processing the video frame by frame and updating the global Line objects based on curves fitted from each frame.

The steps of the process are:

1. Undistort the camera image
2. Apply the gradient and color thresholds to give a binary image
3. Transform the perspective to birds eye view
4. Detect points and fit curves to the lines
5. Calculate the curvature of the lines with the curvature function
6. Check if the points and lines make sense with the check_lines function
	a) Are there sufficient points
	b) Are the distances between the top points of the line reasonable in warped space?
	c) Are the distances between the bottom points of the lines reasonable in warped space?
	d) Is the curvature of each line similar to previous values withing X%
	e) Are the x-points of each line similar to previous values within X%
7. If so, update the line objects with the latest points, keep an n-point moving average of parameters
8. If the lines have been detected for a sufficient number of frames, consider the lines "locked on" and apply a polygon mask around the search area in successive frames based on previous lines. 
	a) This mask is applied with the mask_warped function and draws a polygon around +/- 30 pixels of each line.
	b) See the example masks below
	 
9. Draw the lines and polygones and transform them back to image space.
10. Combine the Polygons to the image and add information such as curvature, confidence and distance from the lane center using the add_info function.

![alt tag](/results_imgs/mask_left.png)
![alt tag](/results_imgs/mask_right.png)

Some of the key parameters in this process were:
the moving average size, n=6
the maximum confidence the filter could acheive (successive frames detect and frames lost before reseting the moving averages. max_confidence = 24, close to 1s of data for 30FPS video.
the threshold for confidence where the region mask would begin to be applied: thresh_confidence = 5

##Performance:

This filter performed well on the project_video.mp4, tracking the left and right lines through all of the video. Having the max_confidence set to 24 in a 30FPS video allowed a fraction of a second of undetected lines to pass before tracking was lost and the previous average line position ceased being used to draw lines. This could have been set higher to allow artificial tracking of the lines longer into periods where detection was difficult. 

Evidence of the moving average values of the lines were apparent in some cases, where if a faulty curve was fit, it would take some time, a fraction of a second, to recover back to displaying the proper line. This was partially resolved by adding more checks on the fitted curve before updating the line objects. 

Outputs for 2 confidence levels are found in: proj6.mp4 and proj5.mp4
An output video for an untracked line (frame by frame line detection) is found in proj.mp4

![alt tag](/results_imgs/output_img.png)

In retrospect, I wish I scaled the confidence value shown rather than putting the current max value "24" on the video.

#Challenge videos

The filter is useless when applied towards the challenge videos as seen in proj_challenge.mp4. Detailed debugging would be required and this would involve creating video output of each stage of the detection process. I had technical difficulties getting moviepy to output data such as the warped image with line fits applied in attempt to debug. I tried saving the video into memory by appending frames to a global list using the moviepy fl_image function but processing these and saving them back into a video required too much memory. This output would have been highly useful for visualizing where the filter is failing.

I suspect succeeding at these videos would mostly involve detailed tuning and small incremental improvements of the current process based on visual movie feedback. The current system seems to fail at identifying the lines at all, most of the improvement would likely involve improving line identification. There are not any obvious additional methods I'm aware of that could augment the process besides additional tuning.



#Improvements

The confidence based moving average approach to tracking the lanes was made up. It could be effectively replaced by a method founded in actual statistics such as the Kalman filter or particle filter. It may have been quite difficult to relate the characteristics or state of the line, such as the polynomial coefficients to a state transition model to be used in the prediction steps of the new filter. How do polynomial coefficients change as the car moves left or right or when the lane begins to curve? Does this process follow a gaussian error distribution? Implementing such a filter is likely beyond the scope of this project for now. 

