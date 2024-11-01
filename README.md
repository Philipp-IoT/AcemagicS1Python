# AcemagicS1Python
Interfacing the Acemagic S1 display from Python on Linux.

# Introduction
This repository collects additional information on how to interface the Acemagic S1 display from Python on Linux. The aim is not to provide a complete library, but to address the issues encountered during the setup of the display. Additionally, an example script is included. The general description of how to interface the display, as well a Node.js tool, is already provided by [AceMagic-S1-LED-TFT-Linux](https://github.com/tjaworski/AceMagic-S1-LED-TFT-Linux) project. 

# Issue
The Acemagic S1 display is connected to the PC using USB. The device implements three USB interfaces, one HID Boot Interface and two custom HID interfaces. The first custom HID interface is used for updating the screen.

```
~$ lsusb -vd 04d9:fd01 | awk '
  /Interface Descriptor/
  /bInterfaceNumber/ { print $0 }
  /bInterfaceClass/ { print $0 }
  /bInterfaceSubClass/ { print $0 }'

Interface Descriptor:
      bInterfaceNumber        0
      bInterfaceClass         3 Human Interface Device
      bInterfaceSubClass      1 Boot Interface Subclass
    Interface Descriptor:
      bInterfaceNumber        1
      bInterfaceClass         3 Human Interface Device
      bInterfaceSubClass      0 
    Interface Descriptor:
      bInterfaceNumber        2
      bInterfaceClass         3 Human Interface Device
      bInterfaceSubClass      0 
```

Unfortunately, the HID descriptor of the second interface cannot be parsed correctly by the kernel. Access using the HIDRAW API is not possible. 

```
~# dmesg

[82116.166307] hid-generic 0003:04D9:FD01.0008: input,hidraw1: USB HID v1.10 Device [HID 04d9:fd01] on usb-0000:07:1b.0-1/input0
[82116.166373] usbhid 5-1:1.1: couldn't find an input interrupt endpoint
[82116.169106] hid-generic 0003:04D9:FD01.0009: hiddev0,hidraw2: USB HID v1.10 Device [HID 04d9:fd01] on usb-0000:07:1b.0-1/input2
```

The [hidapi](https://pypi.org/project/hidapi/) library is based on `libusb` and supports both, HIDRAW and libusb backends. The used backend need to be specified during compilation. The HIDRAW API is used by default.

The [hid](https://github.com/apmorton/pyhidapi) library is also based on `hidapi` and supports both, HIDRAW and libusb APIs a. [If both backends are available, `libhidapi-hidraw` uses HIDRAW](https://github.com/apmorton/pyhidapi/blob/master/hid/__init__.py#L10-L25). 

Therefore, both libraries are not ideal, as HIDRAW API is used by default and access using HIDRAW API is not possible due to the corrupt interface description.

# Interfacing the Display from Python

Fortunately, direct access via libusb is possible using the Python extension libusb1. This library allows raw access to USB devices. Interfacing a custom HID class via raw access is relatively simple, as no special protocol needs to be followed.

HId uses interrupt transfers for communication. The endpoint is included in the USB descriptor. For the AceMagic S1 display controller it is 0x02.

```python
import usb1

# open device
device = usb1.USBContext().openByVendorIDAndProductID(self.vendor_id, self.product_id)
device.claimInterface(1)

# send raw data to the interrupt endpoint
device.interruptWrite(0x02, bytes(data))

# close device
device.releaseInterface(1)
device.close()
```

The custom protocol expects the preamble 0x55 as first byte. No report ID is send as first byte. HID libraries usually expect a report ID as parameter, which is omitted for custom HID classes. Furthermore, the device expects a fixed report size of 4104 bytes.

The following example shows how to set the orientation of the display:
```python
# set orientation to landscape
data = [0x55, 0xA1, 0xF1, 0x01]
# fill the rest of the data with 0x00 to make it 4104 bytes
data.extend([0x00] * (4104 - len(data)))
```

# Debugging 

The `tshark` tool can be used to capture USB packets using `Wireshark` and `libpcap`. This is helpful if no desktop environment is available to directly use `Wireshark`.   

```
~# tashark -D

~# tshark -i 10 -c 100 -w /tmp/usbmon5.pcap
```

# Credits

This project was inspired by and builds upon the great work done by the [AceMagic-S1-LED-TFT-Linux](https://github.com/tjaworski/AceMagic-S1-LED-TFT-Linux) project by tjaworski and contibuters matthiasch and SBRK. Thank you for your great work!
