# Connecting X10 With Alexa
Our current home installed X10, the old, first generation smart home system.

It uses power line to send the signals. From Wiki,
Household electrical wiring which powers lights and appliances is used to send digital data between X10 devices. This data is encoded onto a 120 kHz carrier which is transmitted as bursts during the relatively quiet zero crossings of the 50 or 60 Hz AC alternating current waveform. One bit is transmitted at each zero crossing.

It is a pretty old system, but there are ways to connect it with Alexa.

https://www.instructables.com/id/How-to-Control-X10-Devices-With-Amazon-Echo-or-Goo/
https://coreyswrite.com/uncategorized/amazon-echo-x10-home-control-updated/comment-page-2/

Before starting, make sure you have X10 system installed!
Hardwares:

RPI 4
Firecracker: Send signals to X10 transreceiver, and turn the lights on or off.
transreceiver: Receive the signals from firecracker, and then send the signals to the controllers via powerline. To achieve this, you have to put it in your poweroutlet.
remote controller: Optional. Useful when you are testing the new system.

# Software
Bottlerocket:
ha-bridge: device discovery

