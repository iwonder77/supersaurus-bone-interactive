# Supersaurus Bone Interactive

https://github.com/user-attachments/assets/5193e17a-1725-41bf-98e3-4b2cfe211b8b

## Overview

The Supersaurus bone interactive at Thanksgiving Point's Museum of Ancient Life invites visitors to guess which dinosaur the Supersaurus is related to based solely on scapula bone graphics. Guests place the Supersaurus scapula into one of three slots, each corresponding to a different dinosaur with its own scapula graphic. When they're ready to check their answer, they press a button, and an LED light strip installed behind the frosted acrylic slots briefly lights up red or green, providing immediate feedback.

I thought this would be a great opportunity and challenge to design an interactive without using a microcontroller, which led me to a deep dive into 555 timers, MOSFETs, PCB design and manufacturing, and more. This repository documents that learning process, and includes schematics, design decisions, and links to relevant datasheets.

## Main Hardware Components

- T.I. NE555P Timer ([datasheet](https://www.ti.com/lit/ds/symlink/ne555.pdf?ts=1777883502652))
- IRF520 MOSFET ([datasheet](https://www.vishay.com/docs/91017/irf520.pdf))
- PWM RGB LED Strips [link](https://www.amazon.com/dp/B0FSWXXXD4?ref=fed_asin_title&th=1)
- Full BOM is included in the `docs/` folder

## Full Schematic

![full schematic](docs/full-schematic/bone-interactive-full.jpg)

## Main Hardware Architecture

Each 555 timer's TRIG pin is connected to a single reed switch (mounted behind the slot guests put the bone in), and each reed switch feeds into one "master" N.O. button. RC/pull up networks are configured for each TRIG pin to ensure smooth and predictable switching. This configuration allows the TRIG pin on each 555 timer to be pulled LOW only when its corresponding reed switch is closed **and** the button is pressed.

The three 555 timers are used in monostable mode to output a short pulse signal on pin Q when the TRIG pin is pulled low. More on monostable mode and the duration of the pulse below. The output signal (~10.3V when VCC=12V) is fed into the gate of a IRF520 N-channel MOSFET module, which switches the low side of an RGB LED Strip for as long as the gate is actuated.

## 555 Timer Notes

### Monostable Mode

Shorting the DISCHARGE and THRESHOLD pins of the 555 timer and routing them to an RC circuit operates the 555 timer in monostable mode. Ben Eater has a really great and informative series on the different 555 timer modes, I highly recommend watching them. Here's the one about [monostable mode](https://www.youtube.com/watch?v=81BgFhm2vz8).

In any case, the formula for the output signal duration is

$$
t_w = 1.1 * R_A * C
$$

Where R_A and C are the values of the resistor and capacitor in Ohms and Farads. For future boards I wanted to make this duration configurable, and my first idea to accomplish this is to have three open jumpers routed to resistors of different values (22k, 33k, 47k) which will give different duration lengths depending on which jumper is soldered.

Note: the trigger pulse must be shorter than $t_w$, if TRIG remains LOW beyond that point, the output will hold until TRIG rises back up to HIGH.

### Power-on Reset Circuit

The 0.22uF and 10k RC network on the 555's RESET pin form a POR (power-on reset) circuit. During initial power-up testing without it, the output (Q) pin produced occasional HIGH pulses due to undefined internal states of the IC, which turned on downstream LED strips. This RC network briefly holds RESET low during power-up, forcing Q in a known LOW state while internal circuitry stabilizes. As the 0.22uF capacitor charges, RESET rises to VCC and normal operation begins, preventing false triggering at startup.

### Output (Q) Voltage

When triggered, the output voltage on pin Q isn't necessarily equal to VCC. When powered by 5V, I measured outputs of ~3.3V. When powered by 12V, I measured outputs of ~10.3V. The datasheet confirms this slight drop in voltage, and it is important to know since we'll be using it to drive the gate of an N-Channel MOSFET.

## IRF520 N-Channel MOSFET Notes

### Electrical Characteristics

Below are some of the important electrical characteristics I'll be talking about. For a deeper dive, refer to the [datasheet](https://www.vishay.com/docs/91017/irf520.pdf)

- $V_{GS(th)}$ (gate-source threshold voltage): 2.0-4.0V @ $V_{DS} = V_{GS}$, $I_D = 200{µA}$
  - The point at which the MOSFET barely starts to conduct current from drain to source. It is highly recommended to drive the gate much higher than this value. This advice will be clarified further below.
- $R_{DS(on)}$ (drain-source on-state resistance): 0.27Ω @ $V_{GS} = 10V, I_D = 5.5A$
  - The effective _on_ resistance between the drain and source of the MOSFET. We want this to be as low as possible, since the power drawn by the MOSFET is given by $P = I^2 * R$
- $I_D$ vs. $V_{DS}$ graph at T = 25C (from datasheet)

<div align="center">
    <image src="docs/assets/ID-vs-VDS-curve.png" alt="id-vs-vds-curve">
</div>

In the ohmic region (linear region of the curves) the MOSFET acts like a voltage controlled resistor, where

$$
I_D = \frac{V_{DS}}{R_{DS(on)}}
$$

note that $R_{DS(on)}$ is not a fixed quantity, but decreases as $V_{GS}$ rises above $V_{GS(th)}$. The datasheet value of 0.27Ω only holds at $V_{GS} = 10V$. Near the threshold voltage, it can be orders of magnitude higher. Otherwise, in the saturation region, the MOSFET essentially acts as a current source, since the drain current $I_D$ stays fairly constant regardless of $V_{DS}$

### Driving the Gate

I want to discuss the case where the gate of the IRF520 MOSFET is being driven by the output (Q) pin of the 555 timer powered at VCC = 5V. The reason why this wasn't ideal for our use case stumped me for a bit and it'd be a good idea to explain why here.

From the datasheet, the output voltage of the 555 timer's Q pin is specified to measure in the range of 2.75V-3.3V when VCC = 5V. Let's assume that on a good day, it outputs 3.3V consistently. How would the MOSFET operate with a gate voltage of 3.3V?

The first red flag is that we're driving the gate right in the $V_{GS(th)}$ threshold voltage range, where the MOSFET barely starts to open and conduct current. This is a massive reliability concern, because:

- If the specific MOSFET we used had a $V_{GS(th)}$ = 2V: the MOSFET would slightly conduct
- If the MOSFET had a $V_{GS(th)}$ = 3.3V: the MOSFET would be right at threshold — essentially off
- If the MOSFET had a $V_{GS(th)}$ = 4V: the MOSFET wouldn't conduct at all

This should be enough of a reason to design for a gate-source voltage far past the threshold, but let's make some calculations to hammer down why this is incredibly inefficient. Before we do so let's make some safe assumptions about our load demand:

1. The 12V LED strip we're switching draws 1.67A at 12V (final installation strip draws much more). This makes its effective resistance $R_{load} = \frac{12V}{1.67A} = 7.2 Ω$
2. We set up the LED and MOSFET in a low-side switching configuration (negative input of LED connected to drain of MOSFET, positive input connected to positive power supply output, source grounded at power supply ground).

Take a look at the $I_D$ vs $V_{DS}$ graph. For $V_{GS} = 4.5V$, the device is in saturation at $I_D \approx 1A$ — well below the LED strip's rated 1.67A. The MOSFET becomes the current-limiting element: no matter how high $V_{DS}$ climbs, $I_D$ won't exceed 1A at this gate voltage. At $V_{GS} = 3.3V$, this current would be far smaller. How could we drive 12V LEDs with no more than 1A to spare? The obvious answer is we couldn't. Even so, let's work through exactly how bad it gets. For a device with $V_{GS(th)} = 2V$ — the most favorable end of the spec range — $I_D = 0.3A$ in saturation is a generous estimate.

In saturation, the MOSFET acts as a current source: $I_D$ is goverened by $V_{GS}$ and stays roughly constant regardless of $V_{DS}$. So when the gate is driven at 3.3V, the voltage drop across the LED strip is...

$$
V_{load} = I_D * R_{load} = 0.3 A \times 7.2 \Omega = 2.2V
$$

The LED strip only sees 2.2V! The voltage drop across the drain-to-source is

$$
V_{DS} = 12V - 2.2V = 9.8V
$$

We can confirm the saturation assumption holds: saturation requires $V_{DS} \geq V_{GS} - V_{GS(th)} = 3.3\,\text{V} - 2\,\text{V} = 1.3\,\text{V}$. With $V_{DS} = 9.8\,\text{V}$, this is satisfied by a wide margin — the operating point is self-consistent. The power dissipated by the MOSFET is a cool...

$$
P = V_{DS} * I_D = 9.8V * 0.3A = 2.9W
$$

That is actually not cool at all, the MOSFET is burning nearly 3W, and the LED strip isn't even seeing anything close to 12V.

Let's redo the calculations with $V_{GS} = 10V$ (closer to what the gate sees when the 555 timer is powered by VCC=12V). The curve for
