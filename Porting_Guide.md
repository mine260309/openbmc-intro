# Porting Guide

In this article, it introduces the guide of how to port OpenBMC to a new
machine, including changes to openbmc layers, linux kernel, and several
conponents related to IPMI, gpio, hwmon sensor, LED, etc.

And it also introduce a tip about how to modify code locally to build OpenBMC
with local changes without chaning the recipe.

## Porting to a new machine

To port OpenBMC to a new machine, usually it includes:
1. Add new machine's layer in meta-openbmc
2. Add new machine's kernel changes, e.g. configs, dts
3. Add new machine's Workbook
4. Various changes specific for the new machine, e.g. hwmon sensor, LED, etc.

### Add machine layer

### Kernel changes

### Workbook

### Miscs

#### Hwmon Sensors

#### LEDs

#### GPIOs

#### IPMI

#### Fans
