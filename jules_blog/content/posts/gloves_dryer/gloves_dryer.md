+++
title = 'Gloves Dryer'
tag = ["test"]
date = 2025-03-14T15:24:36Z
+++

## Motivation

Most days, I cycle to work, but I live in Ireland, and rain has become an inseparable part of the commute. 

While it is not a big issue as I carry spare clothes and shoes, my gloves and shoes do get wet and don’t dry over the workday. 
To reduce the time it takes to dry, one can use a few tricks. In this case, I went with a bit of ventilation. Having a small airflow can make a big difference especially since the air in the office is not saturated in water.

The goal of the small project was to build a fan that would be controlled by a humidity sensor placed inside gloves/shoes.

## Software and component used

## Defining expected behaviour


### Firmware for communication with the humidity sensor

### MCU code

### Fan control

### Testing

### PCB layout

{{< highlight c >}}
Timer2_OverflowCallbackRegister(myTimer2ISR);
IO_RC5_SetInterruptHandler(get_ref_rh_and_start_timer);
{{< /highlight >}}

$$I_{E}=I_{B} +I_{C} $$
$$I_{B}=\frac{V_p-V_{BE}}{I_B} $$




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



{{< figure src="/posts/gloves_dryer/test.jpg" title="title of image" width="400">}}


