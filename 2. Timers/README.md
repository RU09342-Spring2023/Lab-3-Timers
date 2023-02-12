# Timers and Timer Modules
Up to this point, timing has been, imprecise. We have been using delay cycles, random loops, and more, but all of the options **keep the processor occupied**. Loops are dependent on checking a condition constantly, taking up clock cycles, and they are not exact, especially if there are branching conditions within the loop. In our most basic example of blinking and LED initially:
```c
P1OUT ^= BIT0;
__delay_cycles(100000);
P1OUT ^= BIT0;
__delay_cycles(100000);
```
Or something like:
```c
i = 0;
while (i<=100000)
{
  i++;
}
P1OUT ^= BIT0;
```

We are spending over *200,000* clock cycles just to manage blinking this LED. If we want things to be precise or be able to let the processor do things while waiting to blink the LED, why don't we leverage another peripheral to do this: **TIMERS**

## What would we want in a timer module
Before we actually take a look what is in the Timer module, what would we want from it? Looking at the previous examples:
- We are trying to space out something happening by a certain amount of time. (Setting a timer)
- We are waiting for that amount of time to trigger moving on to the next instruction. (Setting an Alarm/Interrupt)
- We would like to time how long something is taking. (Capturing or recording an amount of time).

In a pseudocode fashion, we would really like to maybe be able to setup our code more like:
```c
void main(){
  gpioInit();
  setTimerClock();    // Set how fast our clock should run
  setTimerAlertFor(); // Tell the timer to alert us after a certain amount of time
  while(1){
    ...
  }
}

void timer_interrupt(){
  turnOffAlarm();
  doTaskWhenAlarmGoesOff();
}
```

This actually sort of flows pretty naturally, and when we get into it, it makes a lot of our main function simpler to write. To do this, we need some simple blocks to build off of:
- A clock, or something which can generate a clock signal of a known frequency.
- A counter, which can increment based on the clock signal.
- A comparator, which can let us tell if a certain amount of time has passed.

## TIMER B Modules
Let's start taking a look at the Timer B Module.
### Timer Block
This is the part of the timer that is actually keeping time. It is based on a clock generator, which can be controlled with dividers to be slower or quicker. This increments a Counter which is effectively counting how many of the clock cycles have gone by.
![Timer B Timer Block](https://i.gyazo.com/41bde011330cc3404d67aca1dcb7809e.png)

The register which controls this is **TBXCTL**. There are several fields within the register which is important in basic configuration:
- **TBSSEL:** Selects the clock source. For most of our early examples, we will be using the *SMCLK* which is a 1MHz clock.
- **ID:** Internal divider, which can be set to 00 (/1), 01 (/2), 10 (/4), 11 (/8). This can be further divided using the **IDEX** register.
- **MC:** Determines if the counter should not count (00), count up to a user set value (01), count up to a module set value (10), count up to a number then back down to zero (11).

To use these configurations, there are several Macros which have been defined to access these specific fields within the **TBXCTL** register.
```c
TBXCTL = ID_0 | TBSSEL_2 | MC_UP;
```
These definitions can be found in the MSP430FR2355.h file, and you can search for the following header.

![TB0 Registers](https://i.gyazo.com/df9136af276a8e0284bf5a7b2a808bdc.png)

### Capture and Compare Block
This seems more complex, but it at its core allows you to *Capture* the current time in the counter, and also *Compare* the current time to another value. Towards the bottom, we also have a circuit which can **control a pin directly**, but more on that later.

![TB0 CCR](https://i.gyazo.com/f2f668a54df03189981ba6b275e4dbef.png)
