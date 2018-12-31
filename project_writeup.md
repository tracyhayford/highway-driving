# **Highway Driving** 

**Project Goals**

The goals / steps of this project are the following:
In this project, your goal is to design a path planner that is able to create smooth, safe paths for the car to follow along a 3 lane highway with traffic. A successful path planner will be able to keep inside its lane, avoid hitting other cars, and pass slower moving traffic all by using localization, sensor fusion, and map data.

### Compilation
Code compiles without errors with cmake and make.

### Valid Trajectories
* Drives more than 4.32 miles without incident
* Obeys speed limit - never exceeds 50 mph
* Max acceleration and jerk are not exceeded
* No collisions occur
* Car stays in a lane except when switching lanes
* Able to change lanes

### Reflection
The code was developed in a number of steps as suggested by the project Q&A video.

#### Step 1 - Drive straight moving a specified distance each step
    // TODO: define a path made up of (x,y) points that the car will visit sequentially every .02 seconds
    double dist_inc = 0.5;
    for(int i = 0; i < 50; i++)
    {
          next_x_vals.push_back(car_x+(dist_inc*i)*cos(deg2rad(car_yaw)));
          next_y_vals.push_back(car_y+(dist_inc*i)*sin(deg2rad(car_yaw)));
    }
#### Step 2 - Stay in a lane moving a specified distance each step
    // TODO: define a path made up of (x,y) points that the car will visit sequentially every .02 seconds
    double dist_inc = 0.5;
    for(int i = 0; i < 50; i++)
    {
    // move the car along the read
    double next_s = car_s+(i+1)*dist_inc;
    // 1.5 lanes (middle of the middle lane) - 4 meters wide 
    double next_d = (1.5*4);
    vector<double> xy = getXY(next_s, next_d, map_waypoints_s, map_waypoints_x, map_waypoints_y);
    next_x_vals.push_back(xy[0]);
    next_y_vals.push_back(xy[1]);
    }
#### Step 3 - Smoothing (limit acceleration and jerk)
##### Process:
* Started with two points that are tangent to current position
* Added sparsely spaced waypoints out 30, 60 and 90 meters
* Used these 5 points to define a spline function
* Created the new next x,y path by copying any points not “used” (passed) from the previous path
* Filled in the remaining points (up to 50) using the spline to provide smooth transitions (limited acceleration and jerk) from the current position to a point 30 meters along the path

#### Step 4 - Added sensor fusion to avoid collisions
##### Process:
* Check cars in our lane and when one is within 30 meters, reduce speed
* Continue reducing speed until no car is in our lane within 30 meters
* Increase speed when no car is within 30 meters (up to speed limit - 49.5 mph)

#### Step 5 - Added finite state machine to manage lane changes
##### States:
* Keep lane (KL) – stay in the current lane
    > Next states – PLCL, PLCR, KL
* Prepare to change lanes left (PLCL) – perform necessary checks before changing lanes, wait for lane change
    > Next states – KL, PLCL, LCL
* Change lane left (LCL) – check for traffic and if clear, request lane change
    > Next states – KL, LCL
* Prepare to change lanes right (PLCR) – perform necessary checks before changing lanes, wait for lane change
    > Next status – KL, PLCL, LCR
* Change lane right (LCR) – check for traffic and if clear, request lane change
    > Next status – KL, LCR

##### State control (use a cost function to pick a lane):
* Created a helper routine (successor_states) to generate a list of successor states for a given state
    > Code checks to make sure that any lane changes are valid.  If not, the successor states won't allow the "Prepare to change...." state.
* Created a structure (car_prox) to hold car proximity information (flag if it's close, s position, speed, distance away from a specified car - Ours)
* Created a routine (check_prox) that checks a given car's proximity to another car (ours)
* Created a routine (calculate_avg_velocity) to calculate average speed of cars in the selected lane using those within a specified proximity
    > Uses check_prox helper
* Created a routine (calculate_proximity_cost) to calculate cost based on proximity of traffic in the selected lane using those within a specified proximity
    > Uses check_prox helper

##### Loop to handle the state machine
* Calculate cost of each potential next state
    > KL - Cost is based on the how close we are to the target speed.  Zero when there isn't any traffic in our lane.
    > PLCL – Cost is based on the speed of the traffic in the left lane vs target speed.  Zero when traffic is at or above target speed
    > PLCR - Cost is based on the speed of the traffic in the right lane vs target speed.  Zero when traffic is at or above target speed
    > LCL - Cost is based on traffic proximity in left lane.  Zero when no traffic within 30 meters
    > LCR – Cost is based on traffic proximity in right lane.  Zero when no traffic within 30 meters
* Select next state based on lowest cost
* Process next state
    > KL – Code from step 4 already calculates a reference speed.  No other processing necessary.
    > PLCL, PLCR – Code from step 4 already calculates a reference speed.  No other processing necessary.  Wait for LCL or LCR cost to be lowest so lane change can be executed.
    > LCL - Check to see if the lane to the left is clear (using check_prox function).  If it's clear, set lane to lane + 1.  Code from step 3 handles the lane and provides a smooth lane change.
    > LCR - Check to see if the lane to the right is clear (using check_prox function).  If it's clear, set lane to lane - 1.  Code from step 3 handles the lane and provides a smooth lane change.

### Future Optimization Ideas
* Optimize speed change with different slow-down and speed up
* Optimize lane changes to use different distances for cars ahead versus behind
* Add speed matching for lane changes
* Package the code into a class for reuse