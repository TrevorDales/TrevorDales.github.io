+++
title = "Lab 1: Artemis and Bluetooth"
date = "2025-01-30"
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

<img src="/Lab1/tempgraph.png" alt="Rising Temperature Graph" style="display:block ">

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


The above tests were performed using the following example sketches included in the Arduino IDE:
* Basics_blink
* Apollo3_serial
* Apollo3_analogRead
* PDM_microphoneOutput


<br>
<br>

## 1b
The next part of the lab focused on establishing a wireless connection to the Artemis through Bluetooth Low Energy (BLE). I began by activating a virtual environment and starting a Jupyter server.

<br>
<div align = "center">

<img src="/Lab1/VirtualEnvironmentSS.png" alt="Virtual Environment terminal screenshot" style="display:block ">

<img src="/Lab1/jupytertree.png" alt="Virtual Environment terminal screenshot" style="display:block ">

##### Activated VE and file tree from Jupyter server

</div>
<br>


After this, I had the Artemis print its MAC address, which I then inputted to the connections.yaml file in the Jupyter notebook. I also generated a Universally Unique Identifier (UUID) to ensure that my laptop only recognizes my board's services, and not any other boards. After this initial setup, I was able to establish a bluetooth connection between my laptop and my Artemis board.

<br>
<div align = "center">

<img src="/Lab1/mac.png" alt="Virtual Environment terminal screenshot" style="display:block ">

<img src="/Lab1/uuid.png" alt="Virtual Environment terminal screenshot" style="display:block ">

<img src="/Lab1/bluetoothconnect.png" alt="Virtual Environment terminal screenshot" style="display:block ">

##### Successful bluetooth connection!


</div>
<br>
<br>



# Task 1

This task involved writing a new command type for the artemis, called echo:

    case ECHO:
        char char_arr[MAX_MSG_SIZE];

        // Extract the next value from the command string as a character array
        success = robot_cmd.get_next_value(char_arr);
        if (!success)
            return;

        tx_estring_value.clear();
        tx_estring_value.append("Trevor's Robot Says: ");
        tx_estring_value.append(char_arr);
        tx_estring_value.append("!!!!");
        tx_characteristic_string.writeValue(tx_estring_value.c_str());

        break;

<br>
Using this command, I sent a string to my artemis, and my computer recieved and printed an augmented string:

<img src="/Lab1/echo.png" alt="Virtual Environment terminal screenshot" style="display:block ">

<br>
<br>

# Task 2

This task included another command type that received 3 floats, extracted them, and printed them to the Arduino serial monitor. 

    case SEND_THREE_FLOATS:
        float float_a, float_b, float_c;

        // Extract the next value from the command string as a float
        success = robot_cmd.get_next_value(float_a);
        if (!success)
            return;

        // Extract the next value from the command string as a float
        success = robot_cmd.get_next_value(float_b);
        if (!success)
            return;

        // Extract the next value from the command string as a float
        success = robot_cmd.get_next_value(float_c);
        if (!success)
            return;

        Serial.print("3 Floats: ");
        Serial.print(float_a);
        Serial.print(", ");
        Serial.print(float_b);
        Serial.print(", ");
        Serial.print(float_c);

        break;
<img src="/Lab1/3floats.png" alt="Virtual Environment terminal screenshot" style="display:block ">

<br>
<br>

# Task 3

This task utilized the millis() function in another new command type to get the time in milliseconds since boot. 

    case GET_TIME_MILLIS:
        double t;

        tx_estring_value.clear();
        tx_estring_value.append("T:");
        t = millis();
        tx_estring_value.append(t);
        tx_characteristic_string.writeValue(tx_estring_value.c_str());

        break;
<img src="/Lab1/millis.png" alt="Virtual Environment terminal screenshot" style="display:block ">


<br>
<br>

# Task 4

Task 4 was to set up a notification handler for the string GAAT characteristic. This notif handler function runs whenever the string characteristic is updating, making keeping track of the variable easier.

    def notif_handler(uuid, byteArray):
        s = ble.bytearray_to_string(byteArray)
        print(s)
<br>

    ble.start_notify(ble.uuid['RX_STRING'], notif_handler)

<br>
After running these, sending the command automatically results in the time getting printed: 
<img src="/Lab1/notif.png" alt="Virtual Environment terminal screenshot" style="display:block ">
<br>

# Task 5

For task 5, I read the current time and sent it to my computer as many times as possible in 5 seconds. I was able to send 139 messages - each message is 13 bytes, so this comes out to nearly 390 bytes per second.

    case GET_TIME_MILLIS_5s:
        double t1, initial_t;

        initial_t = millis();
        while (millis() - initial_t < 5000)
        {
            tx_estring_value.clear();
            tx_estring_value.append("Time:");
            t1 = millis();
            tx_estring_value.append(t1);
            tx_characteristic_string.writeValue(tx_estring_value.c_str());
        }
        break;

<img src="/Lab1/millis5s.png" alt="Virtual Environment terminal screenshot" style="display:block ">

<br>

# Task 6

Next, I added the timestamps to an array rather than writing each on eto the GAAT characteristic each loop. After the data is collected, I sent it all over at once.

    case SEND_TIME_DATA:
                
        for (int i=0; i<100; i++){
            currentMillis = millis();
            time_stamps[i] = currentMillis;
        }

        for (int i=0; i<100; i++){
        tx_estring_value.clear();
        tx_estring_value.append("Time:");
        tx_estring_value.append((double)time_stamps[i]);
        tx_characteristic_string.writeValue(tx_estring_value.c_str());
        }

        // Reset the array and index after sending data
        memset(time_stamps, 0, sizeof(time_stamps));  // Clear the array
        
        break;

<img src="/Lab1/sendtime.png" alt="Virtual Environment terminal screenshot" style="display:block ">


<br>

# Task 7

For task 7 I implemented tempurature logging in a method similar to the previous task. This time, I also included timestamps for the data.

    case GET_TEMP_READINGS:
        for (int i=0; i<30; i++){
            currentMillis = millis();
            time_stamps[i] = (int)currentMillis;
            temps[i] = (float)getTempDegF();
        }
        for (int i=0; i<20; i++){
            tx_estring_value.clear();
            tx_estring_value.append("Time:");
            tx_estring_value.append((double)time_stamps[i]);
            tx_estring_value.append(",");
            tx_estring_value.append("F:");
            tx_estring_value.append((double)temps[i]);
            tx_characteristic_string.writeValue(tx_estring_value.c_str());
        }
            
        break;

<img src="/Lab1/tempandtime.png" alt="Virtual Environment terminal screenshot" style="display:block ">


<br>

# Task 8

Method 1 records data much slower. The advantage it has is continuous communication with the computer, if you wanted to offload some of the compute for some reason. You also don't need any onboard storage for this. However, method 2 is much faster. The data comes in at way higher resolution, but the computer has to wait until all of it is collected to see any of it. This could also be useful if we wanted to measure values that changed really quickly over a short period of time. It is worth noting that this takes up precious space on the board.

<br>

I can see that my artemis is recording at about 4 times per millisecond, or 4000 times per second. Of the 350kB available, using 2 floats per message we can store 350000/8 = 43750 data pairs before running out of space.

<br>

## Collaboration


I worked with Jack Long and Lucca Correia extensively. I also referenced Daria's website from last year for writing to a GAAT characteristic.
