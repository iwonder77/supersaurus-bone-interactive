# Supersaurus Bone Interactive

https://github.com/user-attachments/assets/5193e17a-1725-41bf-98e3-4b2cfe211b8b

## Overview

The Supersaurus bone interactive at Thanksgiving Point's Museum of Ancient Life invites visitors to guess which dinosaur the Supersaurus is related to based solely on scapula bone graphics. Guests place the Supersaurus scapula into one of three slots, each corresponding to a different dinosaur with its own scapula graphic. When they're ready to check their answer, they press a button, and an LED light strip installed behind the frosted acrylic slots briefly lights up red or green, providing immediate feedback.

I thought this would be a great opportunity and challenge to design an interactive without using a microcontroller, which led me to a deep dive into 555 timers, MOSFETs, PCB design and manufacturing, and more. This repository documents that learning process, and includes schematics, design decisions, and links to relevant datasheets.

## Main Hardware

- NE555P Timer ([datasheet](https://www.ti.com/lit/ds/symlink/ne555.pdf?ts=1777883502652))
- IRF520 MOSFET ([datasheet](https://www.vishay.com/docs/91017/irf520.pdf))
- PWM RGB LED Strips [link](https://www.amazon.com/dp/B0FSWXXXD4?ref=fed_asin_title&th=1)

## Full Schematic

![full schematic](docs/full-schematic/Bone Interactive.jpg)
