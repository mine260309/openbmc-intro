# Porting Guide

In this article, it introduces the guide of how to port OpenBMC to a new
machine, including changes to openbmc layers, linux kernel, and several
components related to IPMI, gpio, hwmon sensor, LED, etc.

And it also introduce a tip about how to modify code locally to build OpenBMC
with local changes without changing the recipe.

## Porting to a new machine

To port OpenBMC to a new machine, usually it includes:
1. Add new machine's layer in meta-openbmc
2. Add new machine's kernel changes, e.g. configs, dts
3. Add new machine's Workbook
4. Various changes specific for the new machine, e.g. hwmon sensor, LED, etc.

### Add machine layer

Let's take an example of adding machine `machine-name-2` with manufacture
`manufacturer-1`.

1. Create `meta-manufacturer-1` under `meta-openbmc-machines/meta-openpower`
   as a directory of `manufacturer-1`;
2. Create `meta-machine-name-2` under `meta_manufacturer-1` as a directory for
   the machine
3. Use machine name `machine-name-2` instead of `manufacturer-1` in config
   files
4. Create a conf dir in `meta_manufacturer-1`, following `meta-ibm/conf`
5. So the final directory tree looks like below:
   ```
   meta-manufacturer-1/
   ├── conf
   │   └── layer.conf
   └── meta-machine-name-2
       ├── conf
       │   ├── bblayers.conf.sample
       │   ├── conf-notes.txt
       │   ├── layer.conf
       │   ├── local.conf.sample
       │   └── machine
       │       └── machine-name-2.conf
       ├── recipes-kernel
       │   └── linux
       │       ├── linux-obmc
       │       │   └── machine-name-2.cfg  # Machine specific kernel configs
       │       └── linux-obmc_%.bbappend
       └── recipes-phosphor
           ...
           ├── images
           │   └── obmc-phosphor-image.bbappend  # Machine specfic apps/services to include
           └── workbook
               ├── machine-name-2-config
               │   └── Machine-name-2.py  # Machine specific workbook (see below)
               └── machine-name-2-config.bb
   ```

The above creates a new layer for the machine, so you can build with
```
TEMPLATECONF=meta-openbmc-machines/meta-openpower/meta-manufacturer-1/meta-machine-name-2/conf . oe-init-build-env
bitbake obmc-phosphor-image
```

### Kernel changes

The device tree is in https://github.com/openbmc/linux/arch/arm/boot/dts,
follow [aspeed-bmc-opp-romulus.dts][1] or similar machine:
1. Add the new machine device tree:
   * Describe the GPIOs, e.g. LED, FSI, gpio-keys, etc. You should get such
     info from schematic.
   * Describe the i2c buses and devices, which usually include various hwmon
     sensors.
   * Describe the other devices, e.g. uarts, mac.
   * Usually the flash layout does not need change, just include
     `openbmc-flash-layout.dtsi` is OK.
2. Modify Makefile to build the device tree.

Note:
* In `dev-4.10`, there is common and machine-specific init code in
  `arch/arm/mach-aspeed/aspeed.c` which is used to do common inits and perform
  specific settings for each machine.
* From `dev-4.13`, there is no such code anymore, most of the inits are done
  with the upstreamed clock and reset driver.
* So if the machine really needs specific settings (e.g. uart routing), please
  send mail to [mailing list][2] for discussion.

### Workbook

In legacy OpenBMC, there is a "workbook" to describe the machine's services,
sensors, FRUs, etc.
It is a python config and is used by other services in [skeleton][3].
In latest OpenBMC, they are mostly replaced by phosphor-xxx services and thus
skeleton is deprecated.

But workbook is still needed for now to make the build.

An example is [meta-quanta][4], that defines its own config in OpenBMC tree,
so that it does not rely on skeleton repo, although it is kind of dummy.

For OpenPOWER machines, several configs are still used in this config, e.g.
in [Romulus.py][5]:
```python
GPIO_CONFIG['BMC_POWER_UP'] = \
        {'gpio_pin': 'D1', 'direction': 'out'}
GPIO_CONFIG['SYS_PWROK_BUFF'] = \
        {'gpio_pin': 'D2', 'direction': 'in'}

GPIO_CONFIGS = {
    'power_config' : {
        'power_good_in' : 'SYS_PWROK_BUFF',
        'power_up_outs' : [
            ('BMC_POWER_UP', True),
        ],
        'reset_outs' : [
        ],
    },
}
```
The PowerUp and PowerOK GPIOs are needed for the build to power on the chassis
and check the power state.

### Miscs

Different machines have different devices, e.g. hwmon sensors, leds, fans.

OpenBMC is designed to be configurable, so you could describe such devices in
different config files in the machine layer.

#### Hwmon Sensors

Hwmon sensors include sensors on board (e.g. temperature sensors, fans) and
OCC sensors.
The config files path and name shall match the devices in device tree.

There is detailed document in openbmc [doc/sensor-architecture][6].

Here let's take Romulus as exmaple.
The config files are in [meta-romulus/recipes-phosphor/sensors][7] which
includes sensors on board and sensors of OCC, where on board sensors are via
i2c and occ sensors are via FSI.

* [w83773g@4c.conf][8] defines the `w83773` temperature sensor containing 3
temperatures:
   ```
   LABEL_temp1 = "outlet"
   ...
   LABEL_temp2 = "inlet_cpu"
   ...
   LABEL_temp3 = "inlet_io"
   ```
   This device is defined in its device tree as [w83773g@4c][9].
   When BMC starts, the udev rule will start `phosphor-hwmon` and it will create
   temperature sensors on below DBus objects based on its sysfs attributes.
   ```
   /xyz/openbmc_project/sensors/temperature/outlet
   /xyz/openbmc_project/sensors/temperature/inlet_cpu
   /xyz/openbmc_project/sensors/temperature/inlet_io
   ```
* [pwm-tacho-controller@1e786000.conf][10] defines the fans and the config is
   similar as above, the difference is that it creates `fan_tach` sensors.
* [occ-hwmon.1.conf][11] defines the occ hwmon sensor for master CPU.
   This config is a bit different, that it shall tell `phosphor-hwmon` to read
   the label instead of directly getting the index of the sensor, because CPU
   cores and DIMMs could be dynamic, e.g. CPU cores could be disabled, DIMMs
   could be pulled out.
   ```
   MODE_temp1 = "label"
   MODE_temp2 = "label"
   ...
   MODE_temp31 = "label"
   MODE_temp32 = "label"
   LABEL_temp91 = "p0_core0_temp"
   LABEL_temp92 = "p0_core1_temp"
   ...
   LABEL_temp33 = "dimm6_temp"
   LABEL_temp34 = "dimm7_temp"
   LABEL_power2 = "p0_power"
   ...
   ```
   * The `MODE_temp* = "label"` tells that if it sees `tempX`, it shall read
      the label which is the sensor id.
   * And `LABEL_temp* = "xxx"` tells the sensor name for the corresponding
      sensor id.
   * For example, if `temp1_input` is 37000 and `temp1_label` is 91 in sysfs,
      `phosphor-hwmon` knows `temp1_input` is for sensor id 91, which is
      `p0_core0_temp`, so it creates
      `/xyz/openbmc_project/sensors/temperature/p0_core0_temp` with sensor
      value 37000.
   * The power sensors do not need to read label since all powers are
      available on a system.

#### LEDs

#### GPIOs

#### IPMI

#### Fans


[1]: https://github.com/openbmc/linux/blob/dev-4.13/arch/arm/boot/dts/aspeed-bmc-opp-romulus.dts
[2]: https://lists.ozlabs.org/listinfo/openbmc
[3]: https://github.com/openbmc/skeleton
[4]: https://github.com/openbmc/openbmc/tree/master/meta-openbmc-machines/meta-x86/meta-quanta/meta-q71l/recipes-phosphor/workbook 
[5]: https://github.com/openbmc/skeleton/blob/master/configs/Romulus.py
[6]: https://github.com/openbmc/docs/blob/master/sensor-architecture.md
[7]: https://github.com/openbmc/openbmc/tree/master/meta-openbmc-machines/meta-openpower/meta-ibm/meta-romulus/recipes-phosphor/sensors
[8]: https://github.com/openbmc/openbmc/blob/master/meta-openbmc-machines/meta-openpower/meta-ibm/meta-romulus/recipes-phosphor/sensors/phosphor-hwmon%25/obmc/hwmon/ahb/apb/i2c%401e78a000/i2c-bus%40440/w83773g%404c.conf
[9]: https://github.com/openbmc/linux/blob/dev-4.13/arch/arm/boot/dts/aspeed-bmc-opp-romulus.dts#L194
[10]: https://github.com/openbmc/openbmc/blob/master/meta-openbmc-machines/meta-openpower/meta-ibm/meta-romulus/recipes-phosphor/sensors/phosphor-hwmon%25/obmc/hwmon/ahb/apb/pwm-tacho-controller%401e786000.conf
[11]: https://github.com/openbmc/openbmc/blob/master/meta-openbmc-machines/meta-openpower/meta-ibm/meta-romulus/recipes-phosphor/sensors/phosphor-hwmon%25/obmc/hwmon/devices/platform/gpio-fsi/fsi0/slave%4000--00/00--00--00--06/sbefifo1-dev0/occ-hwmon.1.conf
