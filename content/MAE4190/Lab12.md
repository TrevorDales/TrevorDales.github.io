+++
title = "Lab 12: Planning and Execution"
date = "2025-05-14"
weight = 6
+++


## Introduction

I worked with Lucca and Jack to navigate our robot through a preset route of waypoints assigned to us in the lab (shown below). Overall, we were very successful in arriving and navigating through these waypoints completely autonomously with a combination of on-board PID control and off-board localization processing! The final result was incredibly rewarding as it required us to combine and implement various aspects of code and hardware debugging learned throughout the semester in addition to navigation and waypoint logic.

We began with complementary arrays of waypoints and localization booleans (i.e. a point, and a boolean to signal the script to localize at that waypoint). This allowed us to manually assign which points we localized at based on the accuracy of localization at each waypoint for the most accurate navigation. To navigate between points, we used sequential orientation and position PID using the DMP onboard our IMU, and Kalman-filtered TOF sensor, respectively.

We used Lucca's robot for this lab (his localization from lab 11 was the most reliable), and collaborated jointly on both Arduino and Python code components!


<div align = "center">

<img src="/Lab12/goal.png" style="display:block ">

##### Goal Waypoints
</div>

<br>
<br>

## Python Architecture

Below are the core functions and logic loop that controlled the robot for this lab:

```python
async def recieveData(localizing, current_waypoint):
# return current location, current orientation
    current_loc = [0, 0]
    print("localizing: ",localizing)
    if localizing:
        await loc.get_observation_data()
        
        # Run Update Step
        loc.update_step()
        belief = loc.plot_update_step_data(plot_data=True)
        
        current_loc[0] = 3.281*belief[0] # x
        current_loc[1] = 3.281*belief[1] # y

    else:
        #assume we are at the waypoint
        current_loc = [current_waypoint[0], current_waypoint[1]] #set current location to x,y of the current waypoint
        cmdr.plot_bel(current_waypoint[0]/3.281, current_waypoint[1]/3.281)
       
    return current_loc

def calculateTargetAngle(x, y, next_waypoint):
    # based on current location, calculate and return required angle
    x2 = next_waypoint[0]
    y2 = next_waypoint[1]
    
    angle_rad = math.atan2(y2 - y, x2 - x)
    target_angle_deg = math.degrees(angle_rad)
   
    return target_angle_deg

def calculateTargetDistance(x, y, current_tof, next_waypoint):
    

    x2 = next_waypoint[0]
    y2 = next_waypoint[1]
   
    distance_to_next_waypoint = math.sqrt((x2 - x)**2 + (y2 - y)**2)
    distance_to_next_waypoint = distance_to_next_waypoint*304.8 #convert from ft to mm
   
    target_distance_mm = current_tof - distance_to_next_waypoint
    return target_distance_mm

# Reset Plots
cmdr.reset_plotter()

# Init Uniform Belief
loc.init_grid_beliefs()

waypoint_list = [(-4, -3), (-2, -1), (1, -1), (2, -3), (5, -3), (5, -2), (5, 3), (0, 3),(0, 0)]
localize_flags = [False, True, False, False, False, True, True, True, True]  # encode whether to localize at a waypoint

current_loc = [-4,-3] #initialize starting loc as the first waypoint
current_tof = 0 # [mm]
cmdr.plot_bel(current_loc[0]/3.281, current_loc[1]/3.281)

while True: # main logic loop
   
    for i in range(len(waypoint_list)): # i represents current waypoint (starts at zero)

        target_angle = calculateTargetAngle(current_loc[0], current_loc[1], waypoint_list[i+1])
        print("Target angle:", target_angle)
   
        ble.send_command(CMD.PID_TURN_CONTROL, f"1.1|0|120|{target_angle}") # P|I|D

        await asyncio.sleep(5)

        ble.send_command(CMD.SEND_CURRENT_POSE, "")
        
        await asyncio.sleep(1.0)

        df = pd.read_csv("MappingData.csv")
        current_tof = df["TOF"].iloc[0]
        
        print("Current TOF:", current_tof)

        target_distance = calculateTargetDistance(current_loc[0], current_loc[1], current_tof, waypoint_list[i+1])

        print("Target Distance:", target_distance)
        
        ble.send_command(CMD.PID_CONTROL, f".14|0|40|{target_distance}") # P|I|D

        await asyncio.sleep(10.0)

        current_loc = await recieveData(localize_flags[i], waypoint_list[i+1])
        print("Current Location:", current_loc)

        with open("MappingData.csv", "w") as f:
            f.truncate()


        print("------------------------------------------------------------")

```
<br>

# Logic and Command Order

The navigation path is defined by a list of 2D waypoints waypoint_list, with each point representing a target location in feet. An accompanying list of boolean flags localize_flags determines whether the robot should re-localize its position at each waypoint using a grid-based localization system.

The script continuously loops through all waypoints in sequence, executing the following steps for each:

<br>

<b> 1. Turn Toward the Next Waypoint:</b>

- Calculates the angle between the robot’s current position and the next waypoint using the `atan2` function.

- Sends a PID control command over Bluetooth to rotate the robot to the calculated heading (`CMD.PID_TURN_CONTROL`, described in artemis section)

<b> 2. Read Current TOF Sensor Data:</b>

- Requests the current pose from the robot (`CMD.SEND_CURRENT_POSE`, described in artemis section).

- Extracts the distance reading from the CSV.

<b> 3. Move Toward the Waypoint: </b>

- Computes the expected distance to the next waypoint based on current position belief.

- Subtracts the distance to the next point from the current TOF reading to calculate the necessary target distance for the next motion.

- Sends a PID control command over Bluetooth to move the robot forward to the calculated target. (`CMD.PID_CONTROL`, described in artemis section)


<b> 4. Update Robot Location: </b>

- If localization is enabled for the current waypoint, the robot turns to zero degrees and then performs an observation scan (function `perform_observation_loop`) and sends the data through BLE. Then, an update step is performed to estimate the robot’s current position.
- If localization is disabled, the robot’s position is assumed to be exactly at the waypoint.


<br>
<br>

This dynamic approach meant we didn't have to hardcode numbers in; the parametric functions allow us to calculate targets and directions that work for any combination of waypoints. This was especially useful for correcting errors. If the robot missed a point slightly, after localizing it could calculate new targets to get back on track. 

<br>
<br>

## Artemis Commands

The following outlines the cases implemented in C on our artemis board. Note that data sent to the computer is logged in a CSV by a notification handler, where it can then be accessed as needed.

Initially, an arrived boolean flag (triggered when the robot arrived within a threshold of the target angle or distance) was used to end the PID control and move to the next step in our larger control loop. However, this meant that if we slightly overshot the target, the flag would flip true and disable the control required to return to the target.

We tried to severly overdampen both orientation and linear PID to fix this, however it left us with too much steady-state error given the variability of our deadband with battery voltage.

Our solution was implementing a simple 5 second cutoff timer before stopping the PID loop. While not the most elegant solution, this gave us plenty of time to approach angles/distances and eliminated the issue of early PID stoppage. getDMP(), RunPIDLin(), and RunPIDRot() are well-documented in previous labs, and handle the lowest level of data acquisition, filtering, and motor control. The clearVariables() function is seen in all cases used in this lab; it resets counters and recorded datapoints to ensure the Artemis doesn't run out of memory over long execution times.

<br>

For angular PI control, we implemented:
# `CMD.PI_TURN_CONTROL`:

```C++
// Get PID parameters over BLE
success = robot_cmd.get_next_value(Kp_turn);
if (!success) return;
// Same for Kd & Ki
...

success = robot_cmd.get_next_value(target_turn);
if (!success) return;
tx_estring_value.clear();

// Variables used for cutoff timer (5 s)
...

int turn_cutoff = 5000;
while ( (millis() - turn_starttime) < turn_cutoff ) { 
    getDMP();
    runPIDRot();
    counter_turn += 1;

}
stop();
pidTurn_start = 0;
clearVariables();
break;

```
<br>

And for linear PI control:
# `CMD.PI_TURN_CONTROL`:
```C++
distanceSensor1.startRanging();
pid_start = 0;
// Get PID parameters over BLE
success = robot_cmd.get_next_value(Kp);
if (!success) return;
// Same for Kd & Ki
...

success = robot_cmd.get_next_value(target_tof);
if (!success) return;
tx_estring_value.clear();

// Variables used for cutoff timer (7 s)
...
int linear_cutoff = 7000; 

while ((millis() - linear_starttime) < linear_cutoff) { // hard time cutoff
    runPIDLin();
    counter_lin = counter_lin + 1;
}
    stop();
    clearVariables();
    run = 0;

    break;
```
Where `runPIDLin()` is the PID logic function implemented in Lab 5.

<br>

To handle sending the current TOF reading, we implemented the following function:

# `CMD.SEND_CURRENT_POSE`:
```C++
distanceSensor1.startRanging();

while(!distanceSensor1.checkForDataReady()) {
    delay(10); // delay while not ready 
}

collectTOF();

// Send Data
tx_estring_value.clear();
tx_estring_value.append("T:");
tx_estring_value.append(TOF1[0]);
tx_characteristic_string.writeValue(tx_estring_value.c_str());

clearVariables();
break;
```
This function was used to inform the calculation for the target of linear PI control.

<br>

Lastly, the command responsible for performing a localization scan was reused from lab 11 (`CMD.SPIN`). This command is called from the `perform_observation_loop()` function on the laptop side.

<br>
<br>


## Results

Below are a series of trials from our lab! In the last two runs you can see a live belief map and logging in Jupyter Notebook, which ouputs critical information like whether we localize at that specific waypoint, our current belief pose, our calculated target heading, etc. To speed up our traversal, we chose to localize only at points that the robot was least consistent in reaching, and left other points entirely up to the lower level control.

<br>

# Early Trial:
<div align = "center">

<iframe width="880" height="495" 
    src="https://www.youtube.com/embed/_luMDGDUHVw" 
    frameborder="1" 

    allowfullscreen>
</iframe>


</div>

We saw a lot of success with the structure of our high-level planning on the first run, but ran into trouble with inconsistencies in lower-level control and angle targeting. When we localized, our planner output was correct based on position belief, but we struggled to tune the orientation PID loop in such a way that it responded (similarly) well to changes in angle between 10 and 180 degrees, while settling within a reasonable time. We changed our approach to add a time cutoff to both control loops, sacrificing a bit of accuracy for longer-term operation.

<br>

# Intermediate Trial:
<div align = "center">

<iframe width="880" height="495" 
    src="https://www.youtube.com/embed/ufMTsQO14IY" 
    frameborder="1" 

    allowfullscreen>
</iframe>


</div>
During this run we had to nudge robot at (-5,-3) due to a ToF underestimation. Also, the derivative term blew up at (5,3) while localizing so we had to manually face it at the temporary target angle. Otherwise, the pieces are starting to come together at this point!

<br>
<br>

# Final Result! : 
<div align = "center">

<iframe width="880" height="495" 
    src="https://www.youtube.com/embed/LJK30JyzM_U" 
    frameborder="1" 

    allowfullscreen>
</iframe>

</div>
Here is a full successful run! After extensive debugging, the robot successfully planned and executed a full autonomous traversal through all the waypoints. 
The advantage of our parameterized logic loop can be seen here; although not every point was reached with complete accuracy (see (5, 3)), the Bayes filter was able to identify any error in our position. Our python planner then calculated new targets to recover from this error. 

<br>
<br>
<br>

## Conclusion

Overall, it was super rewarding to see the different concepts that we've been implementing working together in unison. Big ups to [Lucca Correia](https://correial.github.io/) and [Jack Long](https://jack-d-long.github.io/) for being great group mates (in this lab as well as during the whole semester)!

Thank you so much to all of the TAs and to Professor Hellbling for your support this semester! This class taught me so much  :)

