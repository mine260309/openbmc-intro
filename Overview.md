- [OpenBMC Overview](#openbmc-overview)
  * [Overview](#overview)
  * [Systemd](#systemd)
  * [Components](#components)
    + [phosphor services](#phosphor-services)
  * [Layers](#layers)

---


# OpenBMC Overview

In this article it introduces the overview of OpenBMC and the components piece
by piece.


## Overview

OpenBMC is a free open source software management Linux distribution designed
for the embedded environment.

Based on [Yocto][1], it consists of: 
* [U-boot][2]
* [Linux Kernel][3]
* Services managed by systemd


## Systemd

All the services are managed by systemd in OpenBMC.
For details, check [openbmc-systemd.md][4]

Here is an example.

If you want to power on the host, you can run below commands either in BMC
shell or from remote via REST API:
* `systemctl start obmc-host-start@0.target`
* `curl -c cjar -b cjar -k -H "Content-Type: application/json" -d '{"data": "xyz.openbmc_project.State.Host.Transition.On"}' -X PUT https://${bmc}/xyz/openbmc_project/state/host0/attr/RequestedHostTransition`

Both do the same thing eventually, that is to start the
`obmc-host-start@0.target`, which depends on below services/targets:
```
systemctl list-dependencies obmc-host-start@0.target --no-pager
obmc-host-start@0.target
● ├─mapper-wait@-org-openbmc-control-chassis0.service
● ├─obmc-enable-host-watchdog@0.service
● ├─op-occ-enable@0.service
● ├─pcie-slot-detect@0.service
● ├─phosphor-gpio-monitor@checkstop.service
● ├─phosphor-watchdog@poweron.service
● ├─start_host@0.service
● ├─obmc-chassis-poweron@0.target
● │ ├─avsbus-disable@0.service
● │ ├─avsbus-enable@0.service
● │ ├─avsbus-workaround@0.service
● │ ├─cfam_override@0.service
● │ ├─fsi-bind@0.service
● │ ├─fsi-scan@0.service
● │ ├─...
● │ └─obmc-standby.target
● │   ├─mboxd.service
● │   ├─obmc-console@ttyVUART0.service
...
```
Systemd will start all the dependent servcies or targets, e.g.
1. It will start `obmc-chassis-poweron@0.target` to power on the chassis
2. It will start `phosphor-watchdog@poweron.service` to monitor the power on
progress
3. It will start `start_host@0.service` to start the host

Eventually the host is started when all the dependent services are started.


## Components

Let's walk through the code and get an overview of the components.

**Important Note**:
1. Below code structure reflects to OpenBMC tag v2.3
2. Starting from [194ff4f][5], the meta-machines are moved to top level,
   maintained in separated repo, and managed by subtree.
   So below code structure is out-of-date, but still reflect the whole picture
   of OpenBMC's components.

The major parts:

| Dir              | What it is    |
| -------------    | ------------- |
| import-layers    | The imported component from yocto and meta-openembedded |
| meta-openbmc-bsp | The bsp related components, mainly for aspeed, e.g. uboot/kernel configs, etc. |
| meta-phosphor    | Contains all the components of OpenBMC, a lot of stuff here, e.g. uboot/kernel, phosphor-xxx services. |
| meta-openbmc-machines | Machines specific configureations |
| meta-openbmc-machines/meta-openpower/common | Components specific to OpenPOWER |

### phosphor services

Most OpenBMC services are in `meta-phosphor/common/recipes-phosphor/`.

| Dir           | What it is    |
| ------------- | ------------- |
| chassis       | Chassis control, e.g. power on, off, reset button, etc |
| console       | The console server/client for VUART |
| datetime      | The time manager |
| dbus          | Dbus related services, e.g. mapper, dbus interfaces, etc |
| dump          | The debug collector, e.g. handler of checkstop |
| fans          | The fan presence, monitor and controller |
| flash         | The manager of BMC and Host flash, e.g. software manager |
| gpio          | The gpio monitor, e.g. monitor checkstop, or button press |
| host          | The services to control host, e.g. start host |
| images        | The build images, e.g. obmc-phosphor-image, initramfs, debug tarball |
| interfaces    | The rest server for dbug interfaces |
| inventory     | The inventory manager |
| ipmi          | Ipmi related services, e.g. host ipmid, net ipmid, configs of ipmi sensors, etc |
| leds          | The LED manager |
| logging       | The log manager, including ffdc |
| mboxd         | The mboxd service, implementing the mailbox protocol |
| mrw           | MRW related tools, e.g. the tool to parse MRW and generate yaml configs |
| network       | The network manager |
| packagegroups | Define which packages are included |
| sensors       | The hwmon service that maps hwmon sysfs to dbus |
| settings      | The settings manager |
| state         | The state manager to handle BMC, Host, Chassis states |
| users         | The user manager |
| watchdog      | The watchdog service |
| webui         | The reference WebUI for OpenBMC |

## Layers

OpenBMC is based on [Yocto][1], and the build has the concept of layers, where each
component is overrided or appended by the upper layer.

For example, you will see below output during building Romulus:
```
Build Configuration:
...
meta
meta-poky
meta-oe
meta-networking
meta-perl
meta-python
meta-virtualization
meta-phosphor
meta-aspeed
meta-ast2500
meta-openpower
meta-ibm
meta-romulus
```

Let's take linux kernel as example. 
The linux kernel recipe in OpenBMC is `linux-obmc`, and we can find it in
below layers:
1. `meta-phosphor/common/recipes-kernel/linux/`
2. `meta-openbmc-bsp/meta-aspeed/meta-ast2500/recipes-kernel/linux/`
3. `meta-openbmc-machines/meta-openpower/meta-ibm/meta-romulus/recipes-kernel/linux/`

So we know that linux-obmc is
1. Firstly defined in `meta-phosphor`, that specifies the linux kernel's URI
   and revision;
2. And then has a `bbappend` in `meta-ast2500`, that added the `defconfig` for
   the AST2500 chip;
3. At last has a `bbappend` in `meta-romulus`, that added the extra kernel
   configs specific to Romulus.

Putting the above together, `linux-obmc` is built with the specified source
URI, revision and configs.

It is similar for any other component for how it is defined and configured.


[1]: https://www.yoctoproject.org/
[2]: https://github.com/openbmc/u-boot
[3]: https://github.com/openbmc/linux
[4]: https://github.com/openbmc/docs/blob/master/openbmc-systemd.md
[5]: https://github.com/openbmc/openbmc/commit/194ff4f1f5d44b12e9cb06ddafa6adb20174a13c
