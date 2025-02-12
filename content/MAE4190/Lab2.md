+++
title = "Lab 2"
date = "2025-02-11"
weight = 4


+++

<br>

# Setting up the IMU

I began by connecting the IMU to the Artemis. After flashing the example code, I was able to display real time values from the accelerometer.I set AD0_VAL to 1 to define the I2C address of the IMU. In this case, without soldering any connections the board maintains the default address of 0x12.




<div align = "center">

<iframe width="175" height="315" 
    src="https://www.youtube.com/embed/8gO9bUrKclw" 
    frameborder="1" 

    allowfullscreen>
</iframe>

</div>
<br>
<br>
Next, I checked the calibration of the sensor. The values seemed to be quite consistent, so I don't think a two-point calibration is neccessary.
<br>
<br> 

<img src="/Lab2/pitchcal.png" alt="Rising Temperature Graph" style="display:block ">


<div align = "center">

##### Pitch Calibration (measured at 0, 90, and -90 degrees)

<br>
<img src="/Lab2/rollcal.png" alt="Rising Temperature Graph" style="display:block ">

##### Roll Calibration (measured at 0, 90, and -90 degrees)

</div>
<br>
<br>

# Accelerometer Data

I used the following code to calculate pitch and roll from accelerometer values:

    if(myICM.dataReady())
        {
        myICM.getAGMT();
        currentMillis = millis();
        time_stamps[i] = currentMillis;

        //accel pitch  
        pitch_a = atan2(myICM.accY(),myICM.accZ())*180/M_PI; 
        pitch_a_array[i] = pitch_a;

        //accel roll  
        roll_a = atan2(myICM.accX(),myICM.accZ())*180/M_PI;
        roll_a_array[i] = roll_a;
        }

I then appended the values to the string GAAT characteristic, utlilizing a notification handler to record the values in an array for plotting.
<br>
<br>


<img src="/Lab2/pitchstatic.png" alt="Rising Temperature Graph" style="display:block ">


<div align = "center">

##### Plotted Pitch Data (static noise)


</div>
<br>

The data looked very noisy on all axes. To address this, I took a fourier transform of the data to better understand the frequency spectrum that contained the noise. 
<br>
<br>


<img src="/Lab2/pitchfourier.png" alt="Rising Temperature Graph" style="display:block ">

<br>
<br>

In the frequency domain, most of the notable spikes appear between 0-5Hz. As such, I chose 5Hz as my cutoff frequency. I also calculated my data rate to be 129 messages/sec. With these numbers and the equations below, I calculated an alpha value of 0.15 for a low pass filter. 

This number is important - too high and you will be removing information with your filter. However, too low will be ineffective for removing noise.

<br>
<br>

<img src="/Lab2/equation.png" alt="Rising Temperature Graph" style="display:block ">


<div align = "center">

##### Where f_c is cutoff frequency and T is 1/sampling rate.

</div>
<br>

Using this calculated alpha value, I implemented a low pass filter for pitch and roll.

<br>



    //accel pitch low pass filter
    lpf_pitch_a_array[i] = alpha * pitch_a + (1 - alpha) * lpf_pitch_a_array[i - 1];
    lpf_pitch_a_array[i - 1] = lpf_pitch_a_array[i];

<br>


The resulting signal is less susceptible to noise!


<br>
<br>

<img src="/Lab2/lpfstatic.png" alt="Rising Temperature Graph" style="display:block ">


<div align = "center">

##### LPF, static noise

</div>
<br>


<br>

<img src="/Lab2/lpfslowcircle.png" alt="Rising Temperature Graph" style="display:block ">


<div align = "center">

##### LPF, isolated circle motion

</div>
<br>


<br>

<img src="/Lab2/lfpfullcircle.png" alt="Rising Temperature Graph" style="display:block ">


<div align = "center">

##### LPF, full circle motion. The raw signal is incredibly noisy, but the filter does a pretty good job at capturing the macro motion of the sensor.

</div>
<br>

# Gyroscope Data

Next, I implemented data collection for the gyro sensor. Note that the X and Y axes had to be flipped to match the orientation of the gyro and accel sensors on the board.

    //gyro pitch, roll, yaw  
    dt = (millis()-lastT)/1000.;
    lastT = millis();

    pitch_g_array[i] = pitch_g_array[i-1] + myICM.gyrX()*dt;
    roll_g_array[i] = roll_g_array[i-1] - myICM.gyrY()*dt;
    yaw_g_array[i] = yaw_g_array[i-1] + myICM.gyrZ()*dt;


<br>

<img src="/Lab2/gryodata.png" alt="Rising Temperature Graph" style="display:block ">


<div align = "center">

##### Gryo data compared to accelerometer. While the noise isn't as bad, there is drift due to integrating error over time. The accelerometer also seems to introduce lots of noise in an axis when measuring an acceleration in a different direction - the gyro is better at isolating measurements.

</div>
<br>

To deal with the drift, I implemented a complementary filter that used both sensors to achieve a more accurate measurement. I used the same alpha value from before, biasing the data towards the smooth gyro measurements. 

    //gyo complementary filter
    comp_pitch_g_array[i] = ( comp_pitch_g_array[i-1] + myICM.gyrX()*dt ) * (1 - alpha) + (alpha*lpf_pitch_a_array[i]);
    comp_roll_g_array[i] = ( comp_roll_g_array[i-1] - myICM.gyrY()*dt ) * (1 - alpha) + (alpha*lpf_roll_a_array[i]);



<br>

<img src="/Lab2/compfilter.png" alt="Rising Temperature Graph" style="display:block ">


<div align = "center">

##### Comparing signals for a slow rotation. Note the lack of drift in the complementary signal.

</div>
<br>


<br>

<img src="/Lab2/compfilter2.png" alt="Rising Temperature Graph" style="display:block ">


<div align = "center">

##### Sensor data in response to sudden vibrations. The complementary signal doesn't drift like the gryo data, and is also a bit less susceptible to vibration when compared to the accelerometer data. In many ways, this filter retains the best characteristics of each sensor.

</div>
<br>


# Data Sampling

After removing all print statements, I was able to get up to around 130 messages/sec. I used floats for all of the data, as the decimal value made sense for the type of data I was collecting. I didn't use doubles because they take up more space than a float, and I didn't need the extra precision. I used 10 arrays in total - 1 time array, 4 accelerometer data arrays (including filtered data), and 5 gyroscope data arrays. Consolidating this data into less arrays might be more memory efficient, but I found that splitting the data up this way made the parsing further down the line much more organized and readable.

With this setup, I was able to achieve 17 seconds of data by making each array 5000 data points long. I was able to increase this length all the way to 8000 before reaching memory problems, implying a max data collection time of almost 30 seconds. 


<br>
<br>


<img src="/Lab2/17s.png" alt="Rising Temperature Graph" style="display:block ">


<div align = "center">

##### 17 seconds of data.

</div>
<br>

# Car Stunt

This week, we received our cars! I was able to get it to flip by quickly decelerating the car. One thing to note is the absolute lack of fine control.


<div align = "center">

<iframe width="175" height="315" 
    src="https://www.youtube.com/embed/aTJ3zBcDTDw" 
    frameborder="1" 

    allowfullscreen>
</iframe>

</div>
<br>


# Collaboration


I worked with Jack Long and Lucca Correia extensively. I also referenced both Daria's website and Nila's website from last year for help implementing some of the filters. Lastly, I utilized ChatGPT for lots of debugging, and also to speed up writing plotting syntax.