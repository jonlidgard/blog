+++
title = "RF Socket Control with the Beaglebone Black"
date = 2017-11-22T09:14:50Z
description = "Controlling Mercury RC-5 433MHz sockets with a Beaglebone PRU"
categories = ["projects"]
tags = ["433", "home automation", "mercury rc5", "RF"]
thumbnail = "images/rc5-1.jpg"
type = "project"
draft =true
+++

As well as your Beaglebone you will need a cheap 433mhz transmitter. Ebay is a good
source.

![Transmitter](https://github.com/jonlidgard/rfsend/blob/master/images/tx.jpg)

Connections:
* Vcc to P9.07 (+5v SYS)
* Gnd to P9.1 (DGND)
* Data to P8.11.


The Beaglebones, apart from being completely open source, have one big advantage over their
rivals such as the Raspberry pi. The AM3358 Sitara processor used contains not only an ARM
Cortex-A8 but also 2 32 bit 200MHz microcontrollers, known as PRU's (Programmable Realtime Units).
rfsend uses one of these to create the pulse train for the 433mhz transmitter. They run independently of the main
processor and are completely deterministic - making them great for doing timing critical
tasks such as this.

## Configuring linux to use the PRU

rfsend uses the Linux Userspace I/O (UIO) method of communicating with the PRU's. Ensure
you have this configured properly on your linux distribution. I'm using Debian 9.1
with a 4.4.88-ti kernel. In order to use uio instead of the default rproc, modify your
/boot/uEnv.txt as shown:

```
###PRUSS OPTIONS
###pru_rproc (4.4.x-ti kernel)
#uboot_overlay_pru=/lib/firmware/AM335X-PRU-RPROC-4-4-TI-00A0.dtbo
###pru_uio (4.4.x-ti & mainline/bone kernel)
uboot_overlay_pru=/lib/firmware/AM335X-PRU-UIO-00A0.dtbo
```

Reboot & do a ```lsmod | grep uio``` Check you get:
```
uio_pruss               4629  0
uio_pdrv_genirq         4243  0
uio                    11100  2 uio_pruss,uio_pdrv_genirq
```

You will also need to load a cape to configure the Beaglebone pins.

```sudo sh -c "echo cape-universal > /sys/devices/platform/bone_capemgr/slots"```

or, if you want it loaded at boot, add it to '/etc/default/capemgr':

```
# Default settings for capemgr. This file is sourced by /bin/sh from
# /etc/init.d/capemgr.sh

# Options to pass to capemgr
# CAPE=BB-UART1,BB-UART2
CAPE=cape-universal
```

Now ```cat /sys/devices/platform/bone_capemgr/slots``` - You should now see something like:
```
0: PF----  -1
 1: PF----  -1
 2: PF----  -1
 3: PF----  -1
 4: P-O-L-   0 Override Board Name,00A0,Override Manuf,cape-universal
 ```

 Finally, connect the P8.11 pin to PRU0 with ```config-pin P8.11 pruout```
 If you don't have the config-pin script installed you can use this instead
 ```sudo sh -c "echo pruout > /sys/devices/platform/ocp/ocp\:P8_11_pinmux/state"```

You will probably want to add that as a boot script so it survives reboots so create
a text file with the following:

```
[Unit]
Description=Connect P8.11 to pru for rfsend command
After=capemgr.service

[Service]
Type=oneshot
ExecStart=/bin/sh -c "echo pruout > /sys/devices/platform/ocp/ocp:P8_11_pinmux/state"

[Install]
WantedBy=multi-user.target
```

Save it as /etc/systemd/system/config-pin.service
Enable with ```systemctl enable config-pin.service```

Alternatively you could create a custom overlay instead of using cape-universal.

## Sending the codes

{{< figure src="/images/rf_codes.png" caption="TriState codes output from Mercury Transmitter Fob" attr="" attrlink="" >}}

The code table above shows the binary code used to control my switches.
The code consists of 12 tri-state bits and a sync bit.
The first 5 tri-state bits are common to all sockets in the pack and form the group address.
The next 5 tri-state bits are the individual socket address.
Finally the last 2 tri-state bits are the socket state - ON or OFF.

Each tri-state bit is represented by 2 binary bits. So 24 binary bits
form each code.

I used an oscilloscope to decode the signals but you could also use an Arduino
with a cheap receiver module and the rc-switch library.

For more information google the datasheet for the PT2260 chip used in the
transmitter fob.

## Example use
The Mercury switches use a short pulse duration of 180us.
So, based on codes from the above table

```sudo rfsend -t180 -b 000100000101010100111100``` will turn off socket 1

```sudo rfsend -t180 107387``` will turn on socket 1




{{< figure src="/images/rf_codes.png" caption="TriState codes output from Mercury Transmitter Fob" attr="" attrlink="" >}}
Just need to send these using the triState function in rc-switch
Pulse length is set to 180us.

Wemos D1 / NodeMCU has 3.3v I/O so ok for connecting to BBB but
FS1000A transmitter uses 5v logic - however it seems to work ok with the 3.3v output.

Update:
Haven’t found a library that uses the RFM69 module to do the PT2260 protocol. Seems it will be easier to just use a simple bit banging transmitter - shame.

https://rayshobby.net/a-new-way-to-interface-with-remote-power-switch/

connect the beaglebone 3.3v to the Arduino VIN pin on the breadboard
connect the beaglebone ground to the Arduino ground on the breadboard
connect the beaglebone serial rxd to the arduino txd on the breadboard
connect the beaglebone serial txd to the arduino rxd on the breadboard

You should then be able to communicate with the Arduino on /dev/ttyO4, I can't recall which offhand.
/dev/ttyO4 would require an overlay to be loaded, as would any of the UART's. The only UART that is normally loaded at system up is /dev/ttyO0, which is the serial debug port.




Moteino is an Arduino already pared to a RFM69HW radio - expensive though

http://digitalmeans.co.uk/shop/moteino_usb-433mhz


5 x Mercury Sockets 10A - 433.9MHz  - 30m range - Uses RX-480R Module with RF83C Chipset

RF83C - Compatible transmission codes:
PT2260、PT2262、EV1527、PT2240
Uses ASK/OOK

Transmitter uses a HS2260A-R4 ic so looks like using PT2260 protocol
See here:
http://www.princeton.com.tw/Portals/0/Product/PT2260_4.pdf
Address of my tx fob:
A0 - gnd	|
A1 - f	| Solder Pads toast address
A2 - gnd	|
A3 - gnd	|
A4 - f	|

A5 - f
A6 - f
A7 - ?

RFM69H 433MHz Transceiver Tx/Rx board
Should be functional equivalent to RF12B so can use this
linux module:
https://github.com/gkaindl/rfm12b-linux

Description:
https://youtu.be/mRGY1KvX4bY

SO NEED A RFM69/RF12B Software driver for Beaglebone / Arduino that can talk
PT2260 protocol



If using an Arduino:
https://github.com/roccomuso/iot-433mhz

Recommendation to use Arduino with:
Super-heterodyne' RXB6 receiver but transmitter not required to be anything special.


Antenna length
Actually, many 433MHz circuit boards have a coil with a few windings between the circuitry and the solder pad marked ANT. The XD-RF-5V commonly available on the market has a three winding coil with a 5mm diameter. 5mm x 3 x PI accounts for almost 5cm, so the external part of the antenna should be around 12cm to come to a total length of quarter lambda.
I always find antenna's to be black magic, but for me 12cm seemed to work! Around the internet the most common antenna's length seems to be 17.3 cm.
You can even do some more spires if you want or ... if you don't want to make the antenna by yourself, just buy it somewhere.
