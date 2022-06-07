# AVR_Watchdog_Timer
Perhaps the most useful internal timer found in AVR-based microcontrollers and Arduino development boards.

Why? Because it can wake the microcontroller up from the deepest sleep modes. Also, it can help to rescue a system from errors that halt the progress of code execution. No other internal timer can deliver those two results. 

Similar to the other timers, the Watchdog can be used to establish non-blocking loops, that is, code that needs to repeat over and over at regular intervals without tying up the CPU. It cannot provide PWM, however.

The following are notes to myself as of June 2022, regarding the watchdog timer found in AVR microcontrollers including:
* ATmega328 (Arduino Uno, Nano)
* ATmega32u4 (Arduino Leonardo, Micro)
* ATtiny2313, ATtiny24,44,84, ATtine25,45,85
* No doubt many others; they all seem to work more or less the same way.

## The AVR Watchdog (WD)

* Has its own clock, running independently of other oscillators and timers.
* Can be used as a general timer in programs.
* WD clock continues to run during microcontroller (MCU) sleep modes. It can wake up the MCU.
* In System Reset Mode, it will reset the MCU upon WD counter overflow, enabling recovery from "hung" error conditions.

## The WDTON bit in the high fuse byte determines the default condition of the WD:

* "Programmed" condition, the WDTON bit = 0:
  * WD interrupt is not available.
  * WD is always on in System Reset mode, described below.
  * User code can regulate the WD in this mode:
    * determine the WD Counter overflow period by the choice of prescaler; and
    * reset the WD Counter to zero by issuing the WDR instruction; 
  * however, user code cannot stop the WD.
* "Unprogrammed" condition, the WDTON bit = 1:
  * WD interrupt mode can be selected in User code.
  * WD can be stopped by User code. 
  * User code can select WD prescaler and can reset the WD Counter. 
* The factory default state of the WDTON bit is 1, i.e., Unprogrammed.

## Modes of Operation
The WDE and WDIE bits in the register WDTCSR select one of several different WD modes of operation when WDTON fuse bit is Unprogrammed.

* Mode 0: Stopped
  * **How**: clear both bit WDIE and bit WDE in register WDTCSR
  * **Meaning**: WD counter overflows have no effect, if they occur at all. Maybe WD clock actually turns off and consumes no power? That would be nice.

* Mode 1: System Reset Mode
  * **How**: set bit WDIE to 0 and bit WDE to 1 in register WDTCSR
  * **Meaning**: Watchdog will reset the MCU (basically, the whole system.)
  * **When**: the Watchdog Counter overflows and rolls over to zero.

* Mode 2: Interrupt (Only) Mode
  * **How**: set bit WDIE to 1 and bit WDE to 0 in register WDTCSR
  * **Meaning**: Watchdog will raise an interrupt if bit I in the status register also is set to 1.
  * **When**: the Watchdog Counter overflows and rolls over to zero.

* Mode 3: Combined Interrupt and System Reset Mode
  * **How**: set both bit WDIE and bit WDE to 1 in register WDTCSR
  * **Meaning**: Watchdog will first raise an interrupt then switch to System Reset mode. 
  * **When**: In sequence:
    * initially, raise an interrupt upon WD counter overflow as in Mode 2
    * subsequently, reset the MCU upon the next WD counter overflow, as in Mode 1
  * **Application**: The ISR could save relevant data, e.g., to EEPROM, or emit a signal before resetting the system.

## Regulate the length of time before the WD counter overflows by selecting a pre-scaler.
Search for a table named something like, "Watchdog Timer Prescale Select", in the datasheet for the AVR part you are using. Most likely it will be under the section that describes the WDTCSR register. The table explains how to set the bits in WDTCSR that determine the choice of prescaler, and thus how long it takes the WD Counter to overflow.

## Prevent WD from interrupting or resetting the MCU in either of two ways:
* Way #1: stop the WD.
* Way #2: reset the WD Counter before it overflows, by issuing the WDR assembly language instruction.
* Example "inline" assembly statement for WDR in C++:
  * ```asm volatile ("wdr \n\t" :::);```
