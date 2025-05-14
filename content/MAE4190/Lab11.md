+++
title = "Lab 11: Localization (real)"
date = "2025-05-05"
weight = 6
+++


## Objective  
The goal of this lab was to implement the Bayes filter on the real robot using only the update step, and to compare the localization accuracy in simulation versus reality. Because of the noisy motion of the robot, only the update step was used with data collected from ToF sensors during a 360° rotation. The experiment was performed at four marked positions in the lab arena, and the results were analyzed based on how close the robot's belief was to ground truth.

---

## Simulation  

I verified that the Bayes filter update step worked correctly in simulation by running the `lab11_sim.ipynb` notebook. The belief distribution converged as expected, with clear distinction between odometry, ground truth, and belief.


<div align = "center">

<img src="/Lab11/im1.png" style="display:block ">


</div>

<br>

The simulation showed similar behavior to previous results in Lab 10. This confirmed that the Bayes filter code is functional before testing it on the real robot.

---

## Real Robot Implementation  

To enable the real robot to collect sensor data for the update step, I implemented the `perform_observation_loop()` function in the `RealRobot` class inside `lab11_real.ipynb`. This function commands the robot to rotate and collect ToF and IMU data at 20° increments. I also added a wait for a user input to allow me to tell the program to continue after the robot had finished scanning.

```python
def perform_observation_loop(self, rot_vel=120):
    ble.send_command(CMD.START_YPID, f"{kp}|{ki}|{kd}|{target_increment}|{df_alpha}|{turn_floor}|{time_increment}|{runtime}")

        print("Paused. Press Enter to continue...")

        # Wait for user input
        input()
        
        # Continue executing
        print("Continuing...")

        ble.send_command(CMD.GET_TOF, "")
        
                
        while (len(tof2_values) < 18):
            await asyncio.sleep(3)

            
        sensor_ranges = np.array(tof2_values)[np.newaxis].T
        sensor_bearings = np.array(tof2_orientation)[np.newaxis].T
        
        return sensor_ranges, sensor_bearings
```

The function returns angle and distance arrays as required by the update step of the Bayes filter. BLE communication is used to send commands to the robot and retrieve the sensor data.

---

## Localization Results  

I placed the robot at each of the four marked poses and ran the update step. The resulting belief was plotted and compared with the known ground truth. The green marker shows ground truth, and the blue marker shows the belief estimate.

---

### Pose 1: (-3 ft, -2 ft)


<div align = "center">

<img src="/Lab11/b1.png" style="display:block ">


</div>

<br>

See bad results discussion below.

---

### Pose 2: (0 ft, 3 ft)


<div align = "center">

<img src="/Lab11/b2.png" style="display:block ">


</div>

<br>

See bad results discussion below.

---

### Pose 3: (5 ft, -3 ft)


<div align = "center">

<img src="/Lab11/b3.png" style="display:block ">


</div>

<br>

See bad results discussion below.

---

### Pose 4: (5 ft, 3 ft)


<div align = "center">

<img src="/Lab11/b4.png" style="display:block ">


</div>

<br>

This is the one location where the localization is semi-accurate. Perhaps this is due to the close proximity to both the central box and the outer walls; this could lead to better tof data.

---

## Bad Results Discussion  

I spent a lot of time trying to debug why my localization was so innacurate. I ensured that all data points were taken at 20° intervals, and I also tried playing with the sensor uncertainty that the filter was using. At one point, I compared my data to the data of another student who was getting more accurate beliefs:


<div align = "center">

<img src="/Lab11/im2.png" style="display:block ">


</div>

<br>

Visually, the data looks pretty similar in shape and numerical values. I tried manually adjusting the more innacurate points where the tof sensor is trying to measure a greater distance (angles from ~0-60°). However this had no effect - this leads me to think that the way my sensor is slightly underestimating distances at *all* angles is leading to huge errors in my beliefs. I tried shimming the sensor to point upwards slightly, as well as adding solder to my connections to ensure they were solid. Despite this, the issues continued. Eventually, I had to just accept the error that I'm seeing :(

Because the simulation works, I know that this is an issue with my data. Theoretically, if the data is good, the bayes filter should work (parsing in the good data instead of mine confirms this). I'm not sure exactly what is wrong with my sensor data but it clearly is not what it should be.

---

## Video Demonstration  

<div align = "center">

<iframe width="500" height="315" 
    src="https://www.youtube.com/embed/w8AMTG4n4Uk" 
    frameborder="1" 

    allowfullscreen>
</iframe>

</div>

The car is clearly still when each data point is taken. The belief issue is likely with my TOF sensor - I have had issues with its accuracy in previously labs, and I think it is unfortunately on its last legs.

---

## Collaboration

I worked extensively with Lucca Correia and Jack Long. I also referenced Daria Kot's website for help utilizing the async keyword.

