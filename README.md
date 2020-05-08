# cp210x-cfg
CLI utility for programming CP210x USB&lt;-&gt;UART bridges

Intended as a scriptable alternative to the official Silicon Labs CP21xx Customization Utility.

Tested with CP2102, CP2105 and the newer CP2102n chips.

Supported fields that can be programmed:
  - Vendor ID
  - Product ID
  - Product Name string
  - Serial string
  - Buffer flush bitmap
  - SCI/ECI gpio/modem mode
  - Toggle the device's USB power descriptor between the default 100mA and 500mA (experimental feature)

Additionally for the CP2102n:
  - Manufacturer string
  - Toggle between internal and external serial number
  - Enable TX/RX led outputs

# CAUTION

#### Write-once operation on some chips

This applies to the older CP210x chips (not the CP2102n)
Programming a CP210x config field is a write-once operation. It's perfectly possible to write bad values.
The consequences are yours, and yours alone.


Suffice to say that I've got a couple of modules which have a VID/PID of 0000:0000. Don't do that.
If you happen to screw up the VID/PID anyway, in Linux you can sort-of work around it by registering the
new VID/PID combo with the cp210x driver like so:

```
echo 0000 0000 | sudo tee /sys/bus/usb-serial/drivers/cp210x/new_id
```
Consider yourself forewarned, and forearmed.

Newer chips like the CP2102n can be reprogrammed indefinitely.

#### Experimental features

Enable these by using the -x flag.

The -I flag relies on editing a field described in the documentation
as "non user-modifiable". However, the "XpressConfigurator" tool from SiLabs seems to
be setting that field anyway. Also, its value corresponds to the standard USB
descriptor named "bMaxPower" - which enables the device to indicate 
its' maximum current draw - in units of "2mA".

While it's been proved to work properly in my case, make sure to test this feature 
thoroughly before using it in production.

## Examples
#### Showing the built-in help message (may differ from below)
```
$ ./cp210x-cfg -h
Syntax:
cp210x-cfg [-h ] |
           [-m vid:pid] [-d bus:dev]
           [ -l | [-V vid] [-P pid] [-F flush] [-M mode] [-N name] [-S serial]]

  -h            This help
  -m vid:pid    Find and use first device with vid:pid
  -d bus:dev    Find and use device at bus:dev
  -l            List all CP210x devices connected
  -V vid        Program the given Vendor ID
  -P pid        Program the given Product ID
  -F flush      Program the given buffer flush bitmap (CP2105 only)
  -M mode       Program the given SCI/ECI mode (CP2105 only)
  -N name       Program the given product name string
  -C manufact.  Program the given manufacturer name (CP2101n only)
  -S serial     Program the given serial string
  -t 0/1        Toggle between internal and user specified serial (CP2101n only)
  -L 0/1        Enable TX/RX led output (CP2101n only)
  -I 0/1        Toggle between 100mA and 500mA usb power descriptor. This flag works only in conjunction with -x flag.
  -H            Print a hexdump of the current device's config
  -x            Enable this tool's experimental features. See README.

Unless the -d option is used, the first found CP210x device is used.
If no programming options are used, the current values are printed.
```

#### Switching a CP2105 into modem mode
```
$ sudo ./cp210x-cfg -d 4.113 -M 0
ID 10c4:ea70 @ bus 004, dev 113: CP2105 Dual USB to UART Bridge Controller
Model: CP2105
Vendor ID: 10c4
Product ID: ea70
Name: CP2105 Dual USB to UART Bridge Controller
Serial: 007079CC
Flush buffers: 33
SCI/ECI mode: 0000

```

#### Tips for the future developers of this tool

I did some reverse engineering, using silabs' "Xpress Configurator", and came to conclusion,
that "Battery Charging" functionality is way too unreaiable to implement here, due to following
reasons:
 - In order to setup "Battery Charging", "Xpress Configurator" IDE modifies many of the devices'
 registers, many documented as "read-only", or even not marked at all.
 - The CP2102n IC doesn't consider some "dumb" wall plugs as proper charging ports, and will
 set CHREN pin to low.
 - By default, when plugged into my notebook, the IC set CHREN pin low after a couple seconds of
 inactivity. The walkaround I've found for this problem is to use a custom driver, which would prevent
 the device's entry into suspend mode.