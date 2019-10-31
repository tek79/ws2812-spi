# ws2812-spi
This module contains python routines to program the WS2812 RGB LED chips on the raspberry,
using the hardware SPI MOSI (so no other hardware is needed)

As the WS2812 communication needs strict timing, the DIN line cannot be driven from
a normal GPIO line with python (an interrupt on the raspberry would screw things up).
Thats' why this module uses the hardware SPI MOSI line, this does confirm to the
timing requirements.

More info on the WS2812: https://wp.josh.com/2014/05/13/ws2812-neopixels-are-not-so-finicky-once-you-get-to-know-them/

# Raspberry Pi

## Wiring of WS2812-Raspberry
Connections from the Raspberry to the WS2812:
```
WS2812     Raspberry
GND   --   GND. At least one of pin 6, 9, 14, 20, 25
DIN   --   MOSI, Pin 19, GPIO 10
VCC   --   5V. At least one of pin 2 or 4
```

Of course the WS2812 can (should) be chained, the DOUT of the first
connected to the DIN of the next, and so on.

## Setup SPI on Raspberry
First, enable the SPI hardware module on the SPI, using raspi-config, in
Advanced Options / SPI, and enabling the SPI interface and the module loading:
    sudo raspi-config


Then, get the python spidev module:
```
git clone https://github.com/doceme/py-spidev.git
cd py-spidev
make
make install
```

## Testing this ws2812.py module
This module can be tested using:
    python ws2812.py


Sample program that uses the module:
```
import spidev
import ws2812
spi = spidev.SpiDev()
spi.open(0,0)

#write 4 WS2812's, with the following colors: red, green, blue, yellow
ws2812.write2812(spi, [[10,0,0], [0,10,0], [0,0,10], [10,10,0]])
```

# Orange Pi
Orange Pi Zero, Armbian_5.91_Orangepizero_Debian_buster_next_4.19.59, 31.10.2019

## Install
sudo armbian-config
System -> Toggle hardware configuration -> spi-spidev
(didn't specify SPI1, there is no pinout for SPI0, SPI0 is for flash)
edit /boot/armbianEnv.txt and add "param_spidev_spi_bus=1" (skip with Orange Pi PC):
```
overlays=spi-spidev usbhost2 usbhost3
param_spidev_spi_bus=1
```
Now you get /dev/spidev1.0 (or /dev/spidev0.0 with Orange Pi PC).

DIN to pin19/SPI1_MOSI/GPIO15/PA15, GND->GND, VCC->5V.

/etc/udev/rules.d/50-spi.rules:
```
SUBSYSTEM=="spidev", GROUP="spiuser", MODE="0660"
```

```
sudo groupadd spiuser
sudo adduser "$USER" spiuser
sudo udevadm control --reload-rules
sudo modprobe -r spidev; sudo modprobe spidev
# logout out and login to update user group
```

## Python and modules
```
sudo apt install python3-pip
sudo pip3 install setuptools
sudo pip3 install wheel
sudo pip3 install numpy
sudo apt install libpython3.7-dev
# (or sudo apt install python3-setuptools python3-dev python3-wheel python3-numpy)
sudo pip3 install spidev
sudo pip3 install git+https://github.com/joosteto/ws2812-spi
sudo sed -i 's/str(err)/(str(err))/g' /usr/local/lib/python3.7/dist-packages/ws2812.py
```

## Test
```
#!/usr/bin/env python3
import spidev
import ws2812
spi = spidev.SpiDev()
spi.open(1,0) # use spi.open(0,0) with Orange Pi PC

#write 4 WS2812's, with the following colors: red, green, blue, yellow (GRB)
ws2812.write2812(spi, [[0,10,0], [10,0,0], [0,0,10], [10,10,0]])
```

## Problems

## Flickering and first led is always green
/usr/local/lib/python3.7/dist-packages/ws2812.py

modify write2812_numpy4(spi,data)-function:
```
    tx = numpy.insert(tx, 0, 0x00) # fix first green led
    spi.max_speed_hz = int(4/1.05e-6) # fix flickering
    spi.writebytes(tx.tolist()) # fix flickering
```
or use https://github.com/mcgurk/ws2812-spi/blob/master/ws2812.py

# Notes #
Note: this module tries to use numpy, if available.
Without numpy it still works, but is *really* slow (more than a second
to update 300 LED's on a Raspberry Pi Zero).
So, if possible, do:
```
sudo apt install python-numpy
```
