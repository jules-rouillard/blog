+++
title = 'Gloves Dryer'
tag = ["test"]
date = 2025-04-03
+++

Most days, I cycle to work, but I live in Ireland, and rain has become an inseparable part of the commute. 

While it is not a big issue as I carry spare clothes and shoes, my gloves and shoes do get wet and don’t dry over the workday. 
To reduce the time it takes to dry, one can use a few tricks. In this case, I went with a bit of ventilation. Having a small airflow can make a big difference especially since the air in the office is not saturated in water.

The goal of the small project was to build a fan that would be controlled by a humidity sensor placed inside gloves/shoes.

Project file : https://github.com/jules-rouillard/gloves_dryer

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


## Firmware for communication with the humidity sensor 
When using a sensor or any type of electronic component, it is important to understand how they behave and how to communicate with them.

**TLDR: READ THE DATA SHEET**[^1]

For standard components, you will be able to find someone online who took the time to write the firmware. This makes life easier.

In this case, I first found an example for PIC micro and reused it without looking much into it.
Nevertheless, it is a good exercise to try to rewrite it using the datasheet. So, here we go!

[^1]: Data sheet for DHT11 from mouser [link](https://www.mouser.com/datasheet/2/758/DHT11-Technical-Data-Sheet-Translated-Version-1143054.pdf)

The data sheet clearly explained the communication protocol. Since it's a single-wire two-way connection, understanding the timing for when the MCU/sensor is talking is crucial. To summarize:

1. The MCU send a start signal
2. The MCU wait for confirmation from the sensor that the message has been received
3. The sensor will start sending the data **(higher data bit)**, which will be a 40-bit message with 16 bits for the humidity, 16 bits for the temperature and an 8-bit checksum.

The differentiation between 0b0 and 0b1 in the message depends on the length of time in which the voltage is high.

With that, I came up with my time diagram.
(I did not like Figure 2, "Overall Communication Process" in the datasheet, as it lacks time information.)

{{< figure src="/posts/gloves_dryer/time_diagram_dhr11.png" title="Figure 1: Communication time diagram" width="400">}}

### MCU code

I will be using the PIC16F18126 an 8-bit microcontroller from Microchip for this project.

Microchip IDE, MPLAB, as a plugin call MPLAB Code Configurator, MCC for short. It is a graphical programming environment that will generate most of the configuration code for the microcontroller. We don't need to spend hours in the data sheet and manually set all the necessary registers.

In MCC, there are a few things we need to initialize. The pins we will use, the tmr2 and the delay module. As well as putting the button interruption EXT_INT at a higher priority than the tmr2 interrupt in the Interruption Manager.

|{{< figure src="/posts/gloves_dryer/pin_def.png" width="100">}} | {{< figure src="/posts/gloves_dryer/Project resource.png"  width="400">}}|
|:-------------------------:|:-------------------------:|

We will use the HFINTOSC clock, which runs at 32MHz, as the main clock for the MCU, but tmr 2 will use LFINTOSC clock, which runs at 31kHz. 
For tmr2 we need to set up the clock source, prescaler, postsclaer and callbackrate to achieve an interruption callback every 10min (600s). I used the maximum value for the pre/post scaler (128/16).
This give us a callbackrate of 9082.

$$IntTime = \frac{Prescaler * Postscaler * CallbackRate}{f_{LFINTOSC}} $$

$$CallbackRate = \frac{IntTime * f_{LFINTOSC}}{Prescaler * Postscaler} = \frac{ 600 * 31.10^{-6}}{128*16} =9082.03 $$

Now, we can move to writing our custom code. First, let's define some shortcuts and the variable we will use:

{{< highlight c >}}
#define SW1 RC4
#define SW2 RC3
#define F0 RC0
#define F1 RC1

unsigned char state, count_3h;
unsigned char Check, T_byte1, T_byte2, RH_byte1, RH_byte2, Ch;
unsigned Temp, RH, RH_ref, Sum;
{{< /highlight >}}

For the sensor communication, we will use the following to send the start signal:

{{< highlight c >}}
void send_start_signal() {
    IO_RC2_SetDigitalOutput();
    IO_RC2_SetLow();
    DELAY_milliseconds(18);
    IO_RC2_SetHigh();
    DELAY_microseconds(10);
    IO_RC2_SetDigitalInput();
}
{{< /highlight >}}

To verify that the start command was detected:

{{< highlight c >}}

void is_sensor_start_detected() {
    Check = 0;
    DELAY_microseconds(40);
    if (IO_RC2_GetValue() == 0) {
        DELAY_microseconds(30);
        if (IO_RC2_GetValue() == 1) {
            Check = 1;
            DELAY_microseconds(40);
        }
    }
}
{{< /highlight >}}

And to read the data sent by the sensor 8bit by 8bit. (If needed reminder on bit manipulation in c[^2]).

[^2]: C Bitwise Operators [link](https://www.tutorialspoint.com/cprogramming/c_bitwise_operators.htm)


{{< highlight c >}}
char ReadData() {
    char i, j;
    for (j = 0; j < 8; j++) {
        while (!IO_RC2_GetValue()); //Wait until input goes high
        DELAY_microseconds(30);
        
        //If input is low after 30us then the bit is a 0
        if (IO_RC2_GetValue() == 0)
            i &= ~(1 << (7 - j)); //Clear bit (7-b)
        else {
            i |= (1 << (7 - j)); //Set bit (7-b)
            while (IO_RC2_GetValue());
        } //Wait until PORTD.F0 goes LOW necessary since if a 1 still high after 30us
    }
    return i;
}
{{< /highlight >}}

Let's defined the timer interruption. We will also use this timer for state 4. 18 interruptions will equal 3 hours.
{{< highlight c >}}
void myTimer2ISR(void) {
    if (state == 2) {
        //Read sensor
        sensor_read();
        //Compare to reference value
        if (RH > (RH_ref + 5)) {
            F0 = 1;
            F1 = 1;
        } else {
            F0 = 0;
            F1 = 0;
        }
    } else {
        if (++count_3h >= 18) {
            //After 3 hours timer2 and fans are turned off
            count_3h = 0;
            F0 = 0;
            F1 = 0;
            Timer2_Stop();
        }
    }
}
{{< /highlight >}}

The button interruption is straightforward, it is just important to make sure we properly reset variables when changing states.
{{< highlight c >}}
void button_press(void) {
    Timer2_Stop();
    if (SW2 == 0 && SW1 == 0) {
        // Always on Code 1
        state = 1;
        IO_RC0_Toggle();
        IO_RC1_Toggle();
    } else if (SW2 == 0 && SW1 == 1) {
        // Sensor control Code 2
        //Set reference humidity value and start timer2
        state = 2;
        sensor_read();
        RH_ref = RH;
        //Turn on fan as the first timer2ISR won't be done until 10min 
        F0 = 1;
        F1 = 1;
        Timer2_Start();
    } else if (SW2 == 1 && SW1 == 0) {
        // Undefine Code 3
        state = 3;
        F0 = 0;
        F1 = 0;
    } else {
        //Time control Code 4
        state = 4;
        count_3h = 0;
        F0 = 1;
        F1 = 1;
        Timer2_Start();
    }
}
{{< /highlight >}}

Looking at the generated code by MCC we can find in timer header "tmr2.h" a function to link a custom call back to our timer interruption, we can also directly put custom code in tmr2.c for the default overflow callback.

I prefer to link the call-back to a custom function in "main.c". It is easier to differentiate custom code from generated MCC code that way.
{{< highlight c >}}
int main(void) {
    SYSTEM_Initialize();
    Timer2_OverflowCallbackRegister(myTimer2ISR);
    IO_RC5_SetInterruptHandler(button_press);
    F0 = 0;
    F1 = 0;
    count_3h = 0;

    // Enable the Global Interrupts 
    INTERRUPT_GlobalInterruptEnable();
    // Enable the Peripheral Interrupts 
    INTERRUPT_PeripheralInterruptEnable();

    while (1) {
        //Nothing to do here
    }
}
{{< /highlight >}}

With this last step, the firmware for the PIC16F18126 is ready.

## Hardware
### Fan control

I found 12V DC at a PC shop, rated for 80mA. I only had BJT PN2222 with me to control the fan. 
Using its data sheet, I verified that it is properly rated for our application.
We can see DC gain is specified for current from 0.1mA to 500mA, but only for a VCE of 1V and 10V.
From Figure 3, we can get the DC gain at 80mA for VCE = 1V is around 220. In reality, VCE will be much lower than 1V. To simplify, I will assume a DC gain of 100 and later measure it.

{{< figure src="/posts/gloves_dryer/dc_current_gain.png">}}



The following schematic summarises the Fan control circuit (Figure 4). We need to dimension RB to limit the current below 80mA.

$$I_{C} \approx \beta I_{B} $$

$$I_{B}=\frac{V_p-V_{BE}}{R_B} $$
$$R_{B}=\frac{V_p-V_{BE}}{I_B} $$
$$R_{B}=\frac{5-0.7}{800.10^{-6}}=5375 \Omega $$


{{< figure src="/posts/gloves_dryer/Diagram_bjt.png" title="Figure 4: Fan control circuit" height="400">}} | {{< figure src="/posts/gloves_dryer/Measured voltage.png" title="Figure 5: Measurement for DC gain estimation" height="400">}}
:-------------------------:|:-------------------------:

I have set up the circuit on a breadboard and added a series 10-ohm resistor to measure the current. I measured a 0.82V drop across the resistor, which means an 82mA current going through.

In Figure 5, I have annotated the voltage measurement without the 10-ohm resistor. From those, we can calculate the DC gain. I get a gain of 114, close enough to my first estimation. To be safer, I could increase the resistance to make sure we are below 80mA, but I did not bother doing it here. I have been using the fan for a while now with no issue. Also, the power supply I used can only deliver 12VDC 150mA, as such, the Fans are running below 80mA.

$$I_{B}=\frac{4.95-0.66}{6000} = 715 \mu A $$
$$\beta = \frac{I_{C}}{I_{B}} = \frac{82}{0.715} = 114 $$ 

### First prototype testing

{{< figure src="/posts/gloves_dryer/breadbord_testing.jpg" title="" >}} | {{< figure src="/posts/gloves_dryer/v1_testing.jpg" title="">}}
:-------------------------:|:-------------------------:

I used a solder board as the support for the prototype after testing on a breadboard. I tested the system with my gloves on a rainy day and it worked! 
I did some adjustments to the timers and threshold value for the humidity sensor (code and timer shown above are post-adjustment)

### PCB layout

To finalise the project, I designed a PCB on Kicad. And sent the Gerber files to a manufacturer.

{{< figure src="/posts/gloves_dryer/LAYOUTPCB.png" title="Layout" class="half">}}

{{< figure src="/posts/gloves_dryer/v2_vs_v1.jpg" title="Prototype vs V2" class="half">}}

<!-- <img src="/posts/gloves_dryer/LAYOUTPCB.png"> -->

{{< figure src="/posts/gloves_dryer/v2.jpg" title="Board V2">}}

## Compenent list

