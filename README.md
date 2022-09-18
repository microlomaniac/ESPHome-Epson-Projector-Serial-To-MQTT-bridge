# ESPHome-Epson-Projector-Serial-To-MQTT-bridge

(forked from ESPHome-Optoma-Projector-Serial-To-MQTT-bridge)
I adapted this repo for use with an ESP8266 connected to an EPSON projector using the ESC/VP21 protocol. Thanks very much rasclatt-dot-com for your great work!

This project uses an ESP8266 to provide a serial to MQTT gateway and associated configuration for Home Assistant control and automation of an Epson projector.

# The setup.

This project assumes you have installed the following;

- Home Assistant (tested on Home Assistant OS version, on a RPi4)
- MQTT broker (I use mosquitto)
- ESPHome, used to configure the ESP8266, and for Over The Air (OTA) updates to the code.

## Bill of materials

You will need the following components;

- 1 x ESP8266 module (ESP32 works, too, but a bit overkill for this purpose) (I use a Wemos D1 mini clone)
- 1 x Serial (TTL) adaptor

<img
  src="https://github.com/rasclatt-dot-com/ESPHome-Optoma-Projector-Serial-To-MQTT-bridge/blob/bcc0036cc4fa2783c63b56ebc33b174ee40c04e2/assets/Max232%20seriaal%20TTL%20adaptor.png"
  alt="RS232 TTL MAAX232 module"
  title="RS232 TTL MAX232 convertor"
  style="display: inline-block; margin: 0 auto; width: 250px">
  
A full blown implimentation of RS232 is not required for this to work, the Max232 (TTL) adaptor converts TX and RX (at a sutiable voltage to not fry the ESP!).

## Cabling

Setup is pretty straight forward, the MAX232 board can be powered from the 5V and GND ports of your ESP board, with TX of the module connected to RX on your ESP and RX on the serial board to TX on your ESP as shown below;


  
# ESPHome (ESP8266 configuration)

This documentation assumes you are familiar with using ESPHome to configure and manage ESP devices for Home Assistant.

Once your device is set up for OTA you need to copy the YAML, aqnd edit the following;

- device specific wifi/ and Static IP addressing (if required)
- make sure not to overwrite the ESP api and OTA keys so ESPHome can still manage the device.
- you may need to update the board details (this is based on using Wemos D1 mini) so yours may be different.
- You may need to update your GPIO pins (depending on your ESP8266) for TX and RX.
- Update your MQTT broker & credentials

ESPHome UART implimentation is geared towards writing to UART, in order to read and update the status this project uses and additional Uart Read Line Sensor library (uart_read_line_sensor.h included above).
The Uart Read Line Sensor library included in this fork is modified to work with the ESC/VP21 protocol as it will now also interpret colons as line endings, which works beautifully for this purpose.

Before the ESPHome will compile and upload this file needs to be copied to the ESPHome directory on your Home Assistant server that contains youe ESP configurations, (default is usually /config/esphome/), make sure that the "include: uart_read_line_sensor.h" line has been copied to your ESPHome configuration.

Once you are happy run the OTA process, once sucessful your device should de automatically detected by MQTT, (see below);

<img
  src="https://github.com/rasclatt-dot-com/ESPHome-Optoma-Projector-Serial-To-MQTT-bridge/blob/80feb1aeed47a8416ac781a16ca254c6eb693d56/assets/MQTT%20view.png"
  alt="Home Assistant Device Info"
  title="Home Assistant Device Info"
  style="display: inline-block; margin: 0 auto; width: 400px">

## RS232 commands

Epson provide documentation of their RS232/ serial command set, called the ESC/VP21 protocol - and this can be used to control pretty much anything the menu and remote control can do, however i have limited this project to power/standby, as the projector is plugged into an AV receiver and the rest of the features (i.e. inputs/ settings) dont need to be changed.

Adding additional sensors/switces/commands to the config should be straight forward - heres the Epson Rs232 guide if you want to go down that path:

https://epson.com/Support/wa00572 (Home)

https://files.support.epson.com/pdf/pl600p/pl600pcm.pdf (Business)

(the PWR commands we use here should work with both Home and Business projectors)

## connecting to the projector

Connection is pretty straight forward, plug the serial cable into the adaptor.

<img
  src="https://github.com/rasclatt-dot-com/ESPHome-Optoma-Projector-Serial-To-MQTT-bridge/blob/e54ede8d1789d1e28bcab44ca5ca86529715b698/assets/hd39hdrrear.jpg"
  alt="Serial adaptor to M5Stack Atom cabling"
  title="Serial adaptor to M5Stack Atom cabling"
  style="display: inline-block; margin: 0 auto; width: 400px">

# Power considerations

My projector is ceiling mounted and does not have a USB port, so I have to connect a 5V power supply. If yours has a USB port, read the following instructions for inspiration on how to deal with the USB port not supplying voltage when the projector is turned off.

This means once the projector switches off there is no power for the ESP unless you are able to provide a separate power supply to keep it on, i have 2 options to address this issue.

1. Battery backup implimentation
2. Separate smart power plug (for powering on controlls)

## Battery Backup (remember to change settings to ESP32 when going this route)

Testing out option one 1 changed the ESP32 module to an Adafruit Huzzah ESP32 feather https://learn.adafruit.com/adafruit-huzzah32-esp32-feather as these modules have an integrated power managment setup that allows a LiPo battery to be plugged in. When usb power is available the module runs on usb power source and charges the battery in the background, when the usb power goes off the power managment automatically switches to run from battery backup.

Even with a large capacity battery, there are other considderations to this option, the battery lifespan would be significantly reduced if the ESP32 was left running 100% of the time, LiPo batteries thaat are constantly drained and reharged will start to loose efficiency and need to be replaced often.

It would make sense to impliment Deep Sleep if you go this path, however there are trade offs in terms of responsivness if the device is only online for a brief time every minute of couple of minutes which may not be ideal for some people. When using Deep Sleep I also add an additional MQTT switch to turn off deep sleep cycles - with a pub/sub message thats read on wakeup to disable the deep sleep cycle, otherwise managing the device and performing OTA updates from ESPHome is almost impossiable!

I looked at a setup that used an IR PIR presence sensor to trigger a wake up on pin when motion is detected, however for the sake of simplicity - and avoid falling in to a rabbit hole of deep sleep and time based aautomation logic i chose to go the simpler path with option 2.

## Separate mains power 

Optoma projectors have a setting that allows projectors to automatically power on when the mains supply is switched on. This seemed the easiest way to close the gap on the ESP configuration (which already allows the projector to be powered down by serial.

After setting up this option in the Optoma menu, I added a smart power plug/ switch to the projector power supply, and configured home automation automations to switch the power off and then back on again to trigger the projector to power on, and use the esp32 setup to trigger power off by serial.

I have added some logic to the power on automation to prevent the power being switched (off/on) within a minute of the projector being switched off to prevent dammage to the projector that may happen if you interupt the cooldown cycle.

# Example Home Automation setup

Example dashboard and automation files have been included, these are based on dashboard buttons that switch all entertainment and living room systems on/ off (including AV receiver & selecting inputs, projector, media player and lighting. 

You will need to edit these to suit your setup - but have incuded these as a starting point.


