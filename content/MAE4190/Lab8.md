+++
title = "Lab 8"
date = "2025-04-08"
weight = 4


+++


<div align = "center">

<table><tr><td><img src="/Lab8/close_finish_ref.JPG" width="250"></td><td><img src="/Lab8/close_finish.jpg" width="250"></td></tr></table>

</div>


# Implementing a Flip

The flip manuever begins by sending a bluetooth command to the robot, beginning the **STUNT** case. This is implemented as a finite state machine with three states:

---

#### 1. **Drive Toward Wall (State 1)**  
- The robot drives forward at maximum speed while continuously collecting distance data from the front TOF sensor.  
- My Kalman filter is used to estimate the robot’s distance from the wall using a system model (`Ad`, `Bd`, `C`) and measurement/model variance (`Sigma_z`, `Sigma_u`).  
- Once the robot reaches the `target` distance (based on Kalman data), it transitions to the next state. This only happens after it has first passed a `primed_distance` threshold - this priming prevents any false positives that might occur from noisy TOF data occuring at distances above ~2000mm. 

---

#### 2. **Execute Flip (State 2)**  
- The robot reverses at maximum speed for a fixed duration (`fliptime`) to perform a physical flip.  
- The `c` parameter, which adjusts drive motor asymmetry, is temporarily set to `flip_calibration_factor` - a different calibration is needed for reversal than for straight driving. This ensures the car is still facing the correct direction after the flip.

---

#### 3. **Drive Away (State 3)**  
- After flipping, the robot continues reversing for the remainder of the allowed time (`runtime`).  
- Distance measurements continue to be collected, and all control inputs are logged for post-run analysis.

---

All of these variables are sent over through bluetooth when the command is sent. This allowed me to try different values to tune the manuever easily.

I had previously defined functions for driving, collecting data, and predicting the next kalman step, so I utilized these as necessary.
<br>
<br>

Artemis stunt code:
```C


    unsigned long start_time = millis();
    while (stuntState == 1) //drive at wall state
    {
        if (millis() - start_time > runtime){
            drive(0);
            stuntState = 0; // stop if too much time has elapsed
            break;
        }

        // attempt to refresh sensor data
        if ( (tof1Counter < MAX_TIME_STAMPS) && (tof2Counter < MAX_TIME_STAMPS)  ) 
        {
        collect_tof();
        }

        // predict new distance value
        kalman(x, sigma, Ad, Bd, Sigma_u, Sigma_z, C, uf);
        float current_distance = tof_kalman[pidCounter];

        pid_control_input[pidCounter] = 100;
        drive(100);

        pid_time_stamps[pidCounter] = millis();
        pidCounter = pidCounter + 1; 
        if (current_distance >= primed_distance){
            primed = 1;
        }

        
        if (current_distance <= target && primed == 1 ){
            stuntState = 2;
        }     
    
    }

    unsigned long flip_start_time = millis();
    c = flip_calibration_factor;
    while (stuntState == 2) // execute flip state
    {

        if (millis() - start_time > runtime){
            
            drive(0);
            stuntState = 0; // stop if too much time has elapsed
            break;
        }

        pid_control_input[pidCounter] = -100;
        drive(-100);

        pid_time_stamps[pidCounter] = millis();
        pidCounter = pidCounter + 1; 

        if (millis() - flip_start_time > fliptime){ //next state after flipping for allotted time
            stuntState = 3;
        }

    }

    c = 0.8;
    while (stuntState == 3) //drive away from wall state
    {

        if (millis() - start_time > runtime){
            
            drive(0);
            stuntState = 0; // stop if too much time has elapsed
            break;
        }

        // attempt to refresh sensor data
        if ( (tof1Counter < MAX_TIME_STAMPS) && (tof2Counter < MAX_TIME_STAMPS)  ) 
        {
        collect_tof();
        }

        pid_control_input[pidCounter] = -100;
        drive(-100);

        pid_time_stamps[pidCounter] = millis();
        pidCounter = pidCounter + 1; 
    }
```      

<br>
<br>

Initially, the car would just skid instead of flipping. To fix this, I added some weight high up and towards the front of the car.
<div align = "center">

<img src="/Lab8/weight1.png" width="300"/> <img src="/Lab8/weight2.png" width="300"/>

#### Construction and attachment of the weight
</div>

<br>
<br>

# Results


<div align = "center">



<iframe width="700" height="315" 
    src="https://www.youtube.com/embed/G4j_Ngfp96A" 
    frameborder="1" 

    allowfullscreen>
</iframe>

#### time = 2.62

<iframe width="700" height="315" 
    src="https://www.youtube.com/embed/ws7I2RxpEwU" 
    frameborder="1" 

    allowfullscreen>
</iframe>

#### time = 2.63

<iframe width="700" height="315" 
    src="https://www.youtube.com/embed/VuU4sOZS4qc" 
    frameborder="1" 

    allowfullscreen>
</iframe>

#### time = 2.6




<img src="/Lab8/flip_graph.png" style="display:block ">

#### It is clear that the kalman filter follows the tof data pretty closely, and lets the car predict when it passes the threshold distance. After hitting the threshold, the input to the motors is reversed.


</div>



<br>
<br>


# Collaboration


I worked with Jack Long and Lucca Correia extensively. ChatGPT helped me make pretty plots.
