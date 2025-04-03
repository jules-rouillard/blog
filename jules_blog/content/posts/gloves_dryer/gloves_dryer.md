+++
title = 'Gloves Dryer'
tag = ["test"]
date = 2025-04-03
+++

Most days, I cycle to work, but I live in Ireland, and rain has become an inseparable part of the commute. 

While it is not a big issue as I carry spare clothes and shoes, my gloves and shoes do get wet and don’t dry over the workday. 
To reduce the time it takes to dry, one can use a few tricks. In this case, I went with a bit of ventilation. Having a small airflow can make a big difference especially since the air in the office is not saturated in water.

The goal of the small project was to build a fan that would be controlled by a humidity sensor placed inside gloves/shoes.

## Defining expected behaviour
The board will have 2 switches and one button. Which give us 4 possible states. The button is used to trigger a new read of the switches and change the state of the system.

There are 3 states that I want:

**Always On/Off**
Fans are always on or off. The states are toggled with a button press.

**Sensor control**
The button press set the reference for humidity (the sensor should be outside the humid area). Then, the sensor should be placed in the humid area. The humidity rises until it passes the threshold, after which the fans are turned on until the humidity decreases below the threshold. The humidity value is checked periodically when the timer interrupt trigger.

**Time control**
The button press starts a timer for the on state, lasting 3 hours. After the timer runs out, the system will go into an off state until the button is press again.
If the button is press while the system is in on state the timer will get reset and the on state will last a further 3 hours.

| SW2 | SW1 | Code | State | 
| - | - | - | -----| 
| 0 | 0 | 1 | **Always On/Off** | 
| 0 | 1 | 2 |  **Sensor control** | 
| 1 | 0 | 3 |  | 
| 1 | 1 | 4 | **Time control** | 

## Software 

### Firmware for communication with the humidity sensor

When using a sensor or any type of electronic component, it is important to understand how they behave and how to communicate with them.

**TLDR: READ THE DATA SHEET**[^1]

For standard components, you will be able to find someone online who took their time to write the firmware. This makes life easier.

In this case, I first found an example for PIC micro and reused it without looking much into it.
Nevertheless, it is a good exercise to try to rewrite it using the data sheet. So, here we go!

[^1]: Data sheet for DHT11 from mouser [link](https://www.mouser.com/datasheet/2/758/DHT11-Technical-Data-Sheet-Translated-Version-1143054.pdf)

The data sheet clearly explained the communication protocol. Since it's a single-wire two-way connection, understanding the timing for when the MCU/sensor is talking is crucial. To summarize:

1. The MCU send a start signal
2. The MCU wait for confirmation from the sensor that the message has been received
3. The sensor will start sending the data, which will be a 40-bit message with 16 bits for the humidity, 16 bits for the temperature and an 8-bit check sum.

The differentiation between 0b0 and 0b1 in the message depends on the length of time in which the voltage is high.

With that, I came up with my time diagram.
(I did not like the figure 2 "Overall Communication Process" in the data sheet as it lacks time information.)

{{< figure src="/posts/gloves_dryer/time_diagram_dhr11.png" title="title of image" width="400">}}

### MCU code


{{< highlight c >}}
Timer2_OverflowCallbackRegister(myTimer2ISR);
IO_RC5_SetInterruptHandler(get_ref_rh_and_start_timer);
{{< /highlight >}}


{{< highlight c >}}

void start_com(){
    IO_RC2_SetDigitalOutput();
    IO_RC2_SetLow();
    DELAY_milliseconds(18);
    IO_RC2_SetHigh();
    DELAY_microseconds(15);
    IO_RC2_SetDigitalInput();
}
{{< /highlight >}}


## Hardware

### Fan control

$$I_{E}=I_{B} +I_{C} $$
$$I_{B}=\frac{V_p-V_{BE}}{I_B} $$


### Testing

{{< figure src="/posts/gloves_dryer/test.jpg" title="title of image" width="400">}}

### PCB layout




## Compenent list