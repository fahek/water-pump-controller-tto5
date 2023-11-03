# The Water Pump Controller

## What Is This?

  <p align="center">
  <img src=./drawings/wpc_system_sketch.png>
  </p>
  <p align = "center">
      <b>Figure 1:</b> Simplified drawing of the water system with the controller
  </p>

The controller was developed with a typical water system of a camper van in mind but
can also be used in other places where devices should not run for too long.
A typical camper van has a tiny bathroom and a kitchen which have water supply.
The sinks are oftentimes equipped with taps that have integrated switches and there
is a toilet which has a pushbutton for flushing. It might also have a shower in the
bathroom and/or one for taking a shower outside.

All switches are basically wired in parallel and turn on a single pump in the water
tank. There is also a main switch that cuts power to the whole pumping system.

This sounds like a reasonable and simple circuit that should not cause any trouble.

However, the manuals usually mention that the main switch should be turned off when no
water is needed to make sure that the pump is not running too long. This switch can be
hard to reach and it is therefore left on most of the time. Usually that does not cause
any problems (since the switches in the taps will be off when no water is requested). But
I know of cases where the pump was indeed running when no water was taken. It is then unclear
how long the pump was running. Luckily the pumps do not always break from that. But
having it running unintentionally is still not ideal and consumes a lot of power which
might be limited in many situations.

Another problem I've seen is that the switches in the taps might not be rated for the
current that the installed pump can draw. This can result in the early failure of tap's
switches and the pump can no longer be activated from that tap.

This controller can help to address these problems. It reads the states of the
switches (using low current to avoid overloading them) and turns on the pump
accordingly.

The controller should also turn off the pump if the switches are stuck for too long.
I believe that something in the range of two minutes is a reasonable time in the targeted
application. This would make sure that the pump is not running for too long while not being
in the way too much. Usually filling a pot for cooking or taking a shower (if one ever makes
use of that feature) does not require uninterrupted water supply for more than that. These
values are used as defaults when the circuit is reset, but can also be adjusted using
extra inputs.

There will also be support for a buzzer that can warn a bit before the water is turned
off and one could reset the timer by just shortly iterrupting the water flow.

The buzzer sound will change when the warning time is over and the pump was turned off.
The sequence used after the timeout can be used to tell which input is still active to
help with finding a tap that might not be fully closed.

The chip has more outputs than needed. Additional outputs were configured to drive LEDs
that can be used to indicate the state of the controller.

The controller has an input for a water pressure switch. Some systems do not use switches
in the taps but try to maintain a constant high pressure in the pipes, similar to what a
home installation would provide. One could also use such a switch as an additional way to
prevent that the pump keeps running when all taps are closed.

The timeout feature can be disabled.

## How Can It Be Used?

### Inputs

#### Taps With Switches

This is the main use case, basically as described above. The controller has six separate
inputs for switches. Having separate inputs can be used to help with finding problems with
stuck switches but is also helpful to keep certain circuits separated. In common systems
for example the toilet flush button activates the pump and also a solenoid valve. The valve
is releasing the water similar to what the classical part of the taps would do for the sinks.
Having a dedicated input at the controller can help with simplifying parts of the wiring.

#### Pressure Switch

A pressure switch can be used to keep a nearly constant pressure in the pipe system,
comparable to what one usually has at home.

One can see this as "turn on the pump if the pressure is too low" or "turn off the pump
when the pressure is too high". It would even be possible to use two switches, where the
high pressure switch is there only as a fallback and not meant to be triggered. Such an
additional switch is oftentimes used in systems for heating water.

This controller has a dedicated input for a high pressure switch to turn off the pump when
all taps are closed but a tap switch might still be active.

A low pressure switch for keeping the pressure in the system at a certain level could be
connected at a tap switch input and would (depending on the timeout enable input) benefit
from the timeout logic as well. If the low pressure signal is active for too long, it
either means that water is constantly taken from a tap or there is a leak. If water is
intentionally taken, the same considerations as for the tap switches regarding the duration
apply.

#### Timeout Enable

If the timeout feature is not useful, but the logic is still considered helpful, the timeout
can be disabled. Although the chip should not get into strange states when this pin is toggled
during use, the intended and tested use case is that this signal is fixed for a given system.

#### Duration Configuration

The I/O pins are used as additional inputs for configuring the warning and timeout periods.

The warning and timeout periods are stored in a register of five bits each. For the default time
scaling, the most significant bit is for 128s, the LSB is for 8s. With that the periods can be
selcted from 8s to 248s (approx 4 minutes) in steps of 8s. It is also possible to configure
a scaling factor for the warning and timeout periods, independant from changing the clock signal.
Changin the clock would also affect the pitch and sequence of the buzzer signal.

Five of the I/O pins are used as data pins fed into these registers. The other three pins are used
to store the data to the respective register. These are input/output pins. After a reset, these pins
are read and stored in a flip-flop each. If an input is high the related register is read from the
external configuration, otherwise the default value is kept. If the configuration should be read
from the external configuration, the I/O pins are set as a high output one by one bringing the data
inputs to a high state as configured for each value. The configurations should be combined using
diodes (and maybe pull-down resistors). In the [Wokwi drawing](https://wokwi.com/projects/380005495431181313)
this project is based on resistors are used instead (diodes not yet available in Wokwi).

If the timeout period is less or equal to the warning period, the warning is activated at the same
time as the timeout, effectively skipping the warning period.

The time scaling configuration divides the clock that is used for the warning and timeout comparison
by the configured value plus one. The configuration value defaults to three so the clock is divided
by four. The configured value can be in the range of 0 to 31, resulting in scaling factors of 1 to 32.
This allows timeout values in the range of 2s to 62s in the fastest and high resolution setting and
64s to 1984s (that's more than half an hour at roughly one minute resolution).

#### Clock

The chip is designed to run from a 32768Hz clock. That's a reasonably low frequency and easily
available as a crystal. Since the timing is not critical, any other clock in that range can
be used as well.

The clock could also be in a different range. That can be helpful, if the circuit for setting the
times should be avoided or if the available range is not suitable for some other use case. However,
this would also affect the pitch and durations of the buzzer sounds that are derived from the same
clock with fixed ratios.

#### Reset

The circuit is meant to be reset during power up. However, if all tap switches are off initially,
the circuit (except for the time period configuration registers) is reset as well. So in case any
switch is on when power is applied and external reset is not used, a wrong warning or timeout might
occur, including an already timed-out state.

### Outputs

#### Pump (OUT0)

This pin is meant to drive a transistor that can control the pump motor either directly or via
a relay.

It can also be used to drive an LED that could indicate a running pump. There are other pins for
LEDs with slightly different meaning as described below.

#### Buzzer (OUT6)

The controller can signal a long running pump with a buzzer. It will first give a reminder that
the water is running relatively long already. When the water is automatically turned off from the
timeout, the signal helps to identify the switch that is still active.

There is an additional output BuzzerHaltedOnly (OUT7) which only outputs the buzzer signal for the
timeout. This allows to have a silent warning period (LED only) but audible signal for the alarm. And
by connecting a buzzer between OUT6 and OUT7 the differential signal can be used to drive a buzzer
only for the warning period (or separate buzzers e.g. with different volumes for each phase).

#### LEDs

There are a couple of outputs that can indicate the state of the controller.

One output (OUT1) is similar to the pump control output, but would not be turned off if the pump gets
disabled because the high pressure switch input was activated. This helps to prevent that users
get the impression that the system is turned off when it is not.

This output can be combined with OUT5 (RunLong). It is indicating that the warning time or timeout
was reached. When using a dual color (green/red) LED, the green LED could be used for the active
pump signal and the red one for the RunLong signal. The LED would then appear green when the pump is
running, orange when the warning time is reached (both LEDs are on) and red when the timeout disabled
the pump (only RunLong stays active).

The outputs OUT2:OUT4 can be used together as an alternative to the combined LED described above. In
this variant there would be one dedicated LED for running normally, reaching the warning time and pump
being turned off from the timeout.

## How Does It Work?

  <p align="center">
  <img src=./drawings/wpc_diagram.png>
  </p>
  <p align = "center">
      <b>Figure 2:</b> The high level diagram of the water pump controller
  </p>

### Logic Blocks

#### Needs Water

A couple of OR-gates are used to determine if one of the tap inputs is active and water is needed or
requested. This gives the main signal for activating the pump.

#### LEDs

There are different LED outputs (see above for details). They are activated depending on the state of
the controller and a logical combination of water request and timer (warning phase or timed out).

#### Buzzer

The buzzer is activated during the warning phase and when the timeout is active with different signals.
Both signals are combined using an OR-gate.

The warning signal is generated using a group of AND-gates from the 1kHz signal, the warning state line
and the lines for 0.5Hz to 4Hz. This results in a beep that repeats every 2 seconds and is on for 125 ms.

The generation of the timed-out signal is more complex and described below since it contains more timing
parts.

### Timing Blocks

#### Frequency Divider

The main part of the frequency divider is built from a simple counter with D-Flip-Flops. It is held in reset
state when none of the tap inputs is active or when the external reset is active. It provides other blocks with
various timing signals, including the warning/timeout, buzzer frequency and sequence. The slowest signals are
clocked by a configurable counter (see below) which is clocked by the 2s signal generated in the fixed part.

The frequency divider has a configurable part to extend the range of possible timeout durations. This variable
part is only used for the timeout comparison but not for the buzzer sound generation. The variable part is also
built from D-Flip-Flops that are reset at the configured scaling value.

To make sure that the timing is stable, an additional Flip-Flop helps with signal forming. The comparison is done
at the falling edge of the input signal and keeps the reset signal applied for one cycle. The counter is at zero
for one and a half cycles (reset at the half cycle where the target counter value is reached, held at reset
until the next half-cycle and then increased to one at the next full cycle).

The output signal of this frequncy divider part is derived as a combination of the reset signal and the clock.

#### Warning and Timeout

The warning and timeout signals are each generated by a D-Flip-Flop. They are activated at the chosen
timer/counter value and stay active afterwards until they are reset by disabling all tap switches or an
external reset.

#### Timed-out Buzzer Sequence

The timed-out buzzer sequence depends on which tap switch is active. The number of beeps in that sequence
reflects the ID or position of the first active input.

One D-Flip-Flop is used to activate the sequence every 16 seconds. It is set high (data is tied to H) on
the rising edge of the 16s counter signal when the controller is in the timeout state. It is reset at the
end of the sequence or by the global reset.

A series of D-Flip-Flops form a sequence counter where with each clock tick an additional output is
activated. The signal that defines the beep duration and interval is used as the clock for this sequence
generator. This sequence generator is held in reset state by the signal that also sets the buzzer active
Flip-Flop mentioned above. Depending on which input is active, one of the sequence outputs is used to reset
the Flip-Flop that activates the buzzer. The signals are combined using AND-gates for selecting the right
sequence step and OR-gates to allow any of those selected signals to reset the buzzer active state.

The actual timed-out buzzer signal is then a combination of the 2kHz signal, the duration and interval line
(two beeps per second at 25% duty cycle), enabled by the already described active Flip-Flop. This timed-out
signal is then combined into the buzzer output signal using the OR-gate as described above.

## Testing

### Basic Tests

For testing the circuit, the outputs (including the pump output) can be connected to LEDs or as in the test board
to a 7-segment display. In the test board the pump output corresponds to the top segment. The inputs can be connected
to DIP switches. The clock should be set to 32768 Hz (2^15 Hz). The reset signal should provide a power-on reset and
optionally a manual reset that might be handy for testing. In the minimal setup, the last three bidirectional I/O pins should be
connected via separate resistors to GND. Connecting them directly to GND should be okay as well for a quick test. These pins can be
outputs that should only be driven to low in this case, but do not connect them directly to VCC. The other I/O pins can be left
open or connected to GND to avoid floating pins. The connection to GND can be done directly or via pull-down resistors to plan 
ahead for more tests with additional circuitry.

#### Timer Disabled

The test is about enabling the pump while not making use of the timer:

- Keep the input 6 (DIP 7) low to disable the timer
- Keep the input 7 (high pressure switch) low 
- Set any combination of tap switches (inputs 0 - 5) high
- The pump output (top segment) should be on
- The pump LED and ActiveNormal LED (right hand segments) should be on
- With all tap switches off all outputs should be off as well

#### High Pressure Switch

The test is to verify the high pressure switch:

- Set input 7 (the high pressure switch) high
- Set any combination of tap switches high
- Select any state for the timer enable pin (input 6)
- The pump output should be off
- The pump LED, ActiveLEDs and buzzer can be on, depending on the state of the controller

#### Timer With Default Settings

A simple test for the timer with default values:

- Set input 6 high to enable the timer feature
- Keep input 7 low to see the pump output
- Set any combination of tap switches (input 0 - 5) high
- Wait
- After 128s, the LED outputs should change from ActiveNormal to ActiveWarning
  (bottom right to bottom on the 7-segment display) and the RunLong LED (top left
  for 7-segment display) should be activated
- At the same time the buzzer should be activated every two seconds. LEDs would blink
  dim (center segment).
- After another 32s, the pump and pump LED should be turned off and the other LEDs should
  go from ActiveWarning to ActiveHalted (bottom to bottom left segment).
- The buzzer should emit a sequence corresponding to the first active tap input every 16
  seconds. Again for LEDs that would be a dim blinking sequence every 16s.
- Set all tap switches to low
- All outputs should be off
- Activate any tap switch 
- Pump (and related LEDs) should be on again, buzzer should be off

### More Advanced Tests

Testing the configuration feature requires additional external circuits. The last three
bidirectional I/O pins can be connected to VCC via a resistor to enable loading the
respective configuration value. Or they stay connected to GND via the resistor to skip loading
the associated value.

The other five data inputs can be connected directly to VCC or GND if a single configuration value
should be loaded. If more than one configuration should be set, the data lines should be connected
to a diode matrix. The cathodes of all diodes in a column (representing the bit positions
in the configuration values) are connected. The anodes of the diodes in a row are connected, each
row is for a specific configuration value.

The data pins would be connected to individual pull-down resistors and the cathodes of the diodes
in the corresponding column. The anodes of the diodes in a row would be connected to the
corresponding bidirectional I/O pin that is set high while the related configuration value is read in.

  <p align="center">
  <img src=./drawings/wpc_diagram.png>
  </p>
  <p align = "center">
      <b>Figure 3:</b> Setup for more complex tests
  </p>

The drawing shows the idea of the configuration matrix. Removing R_CFG for any of the rows skips
the storage of the associated configuration value. The diodes D_CFG would be removed or disconnected
according to the required configuration value. Removing a diode would result in a low configuration
bit at the related position, placing and connecting the diode as shown would give a high configuration
bit.

The tests would set one or more configurations and the timing would be observed.

## I/O Description

### Inputs

- 0 - TapA: Input for switch of first tap. Active tap should be high level.
- 1 - TapB: Input for switch of second tap. Active tap should be high level.
- 2 - TapC: Input for switch of third tap. Active tap should be high level.
- 3 - TapD: Input for switch of fourth tap. Active tap should be high level.
- 4 - TapE: Input for switch of fifth tap. Active tap should be high level.
- 5 - TapF: Input for switch of sixth tap. Active tap should be high level.
- 6 - EnableTimeout: Timeout is enabled when high level is applied.
- 7 - PressureHigh: Input for optional pressure sensor. Pump is disabled when signal has high level.

### Outputs

- 0 - Pump: Use to activate the pump
- 1 - PumpEnabled: LED for pump activity
- 2 - ActiveNormal: LED for normal operation
- 3 - ActiveWarning: LED signalling that the warning timeout is reached
- 4 - ActiveHalted: LED signalling that the pump was disabled after a timeout
- 5 - RunLong: LED showing that the warning or timeout periods were reached
- 6 - Buzzer: Controls a buzzer
- 7 - none

### Bidirectional I/O

- 0 - Configuration data input LSB
- 1 - Configuration data input
- 2 - Configuration data input
- 3 - Configuration data input
- 4 - Configuration data input MSB
- 5 - Timer scale configuration enable input and output for timer scale configuration reading
- 6 - Warning time configuration enable input and output for warning time configuration reading
- 7 - Timeout configuration enable input and output for timeout configuration reading
