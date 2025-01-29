+++
title = "Lab 1"
date = "2025-01-26"
weight = 4


+++

<br>

## 1a
In the first part of the lab, I established wired communication and control over the RedBoard Artemis Nano. I started with a blink test, and then tested the ability of the board to read and write using the serial monitor.
<br>
<br>

<div align = "center">

<iframe width="175" height="315" 
    src="https://www.youtube.com/embed/d6p6UubQxzs" 
    frameborder="1" 

    allowfullscreen>
</iframe>

##### Blink Test

</div>
<br>
<br>

<div align = "center">

<iframe width="175" height="315" 
    src="https://www.youtube.com/embed/jE8nBZ8M0LM" 
    frameborder="1" 

    allowfullscreen>
</iframe>

##### Serial Test

</div>
<br>




Next, I tested some of the onboard sensors - first the temperature sensor:
<br>

<img src="/tempgraph.png" alt="Rising Temperature Graph" style="display:block ">

<div align = "center">

##### The serial monitor visually shows an increase in temperature (&deg;F) when the board is pressed to my hand.

</div>
<br>


And then the microphone:

<br>
<br>



<div align = "center">

<iframe width="175" height="315" 
    src="https://www.youtube.com/embed/6gqOR4L8DFw" 
    frameborder="1" 

    allowfullscreen>
</iframe>

##### Whistling at the board, causing the frequency to spike.

</div>



<br>
<br>


The above tests were performed using the following example sketches included in the Arduino IDE:
* Basics_blink
* Apollo3_serial
* Apollo3_analogRead
* PDM_microphoneOutput


<br>
<br>

## 1b
The next part of the lab focused on establishing a wireless connection to the Artemis through BLE (Bluetooth Low Energy). I began by activating a virtual environment and starting a Jupyter server.

<br>
<img src="/VirtualEnvironmentSS.png" alt="Virtual Environment terminal screenshot" style="display:block ">


