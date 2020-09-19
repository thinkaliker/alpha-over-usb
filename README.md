# alpha-over-usb
RESEARCH ONLY: sniffing live imagery over USB for Sony Alpha cameras

## 2020-09-18 update
Apparently Sony published their webcam software a DAY after I posted this initially. [Link here](https://support.d-imaging.sony.co.jp/app/webcam/l/download/index.php) Oh well. The below is still up for historical purposes.

## Summary
I own a Sony Alpha DSLR (a6300) and want to use it as a USB webcam. I heard rumors of Sony providing a firmware update but can't find a single source that says this is true. In the meantime, Sony themselves recommend either using a capture card, or the third party solution of OBS capturing the Imaging Edge Desktop - Remote viewfinder and outputting that to a virtual webcam.

My thought is - if I can control/see the viewfinder in the Remote tool over USB, why can't I sniff/decode that viewfinder traffic myself?

## Findings (so far)

Here are my findings so far. Spoiler alert: I don't have any code for this.

### Wireshark
As it turns out, Wireshark is able to capture USB traffic if you install USBPcap on install time.

So, I installed Wireshark, and plugged in my camera in PC Remote mode. Then select the device in USBPcap and select start.

![https://i.imgur.com/07vLEQB.png](https://i.imgur.com/07vLEQB.png)

A lot of traffic immediately appears. The next step is to launch Imaging Edge Desktop and launch the Remote tool.

![https://i.imgur.com/4sevwPI.png](https://i.imgur.com/4sevwPI.png)

Once you start it, you will see more packets in Wireshark.

I applied this display filter to Wireshark to only filter out "large" packets. These packets contain our frames.
Filter: `usb.endpoint_address.number == 1 && usb.dst == host && usb.data_len > 20000`

![https://i.imgur.com/ARg9gyc.png](https://i.imgur.com/ARg9gyc.png)

As you can see, the bulk of the payload contains the image data. We can further decode this data.
Right click on the middle pane and select Show Packet Bytes. A window should appear and should show the image decoded. If it does not, make sure Decode As is set to "None" and Show As is set to "Image".

![https://i.imgur.com/2KXBXiX.png](https://i.imgur.com/2KXBXiX.png)
![https://i.imgur.com/QjPcWPf.png](https://i.imgur.com/QjPcWPf.png)

Great! There's only one problem though, some images have a large grey, almost bad color data in the bottom half of the screen. This is also visible in the above screenshot, the last few squares in the bottom row show this.

According to the Sony Camera Remote SDK API reference, the JPEG data starts with the SOI marker `FFD8` and ends with the EOI marker `FFD9`. As I can't seem to find the end marker in my string that I captured, my guess is that the data is cut off prematurely. This might be due to USBPcap or Wireshark.

### libusb/pyusb

I also (briefly) investigated accessing the data myself, using libusb. There's only one problem - I don't know anything about writing USB interfaces. But I tried anyway.

First, I installed libusb and PyUSB using pip. Then I realized that it doesn't work out the box like that on my Windows system. So, using Zadig, I overwrote the MTP driver that was currently in use with my camera with the libusb-win32 driver included with Zadig.

After running the following sample code, I got the output:

Sample code (slightly modified from pyusb):
```
import usb.core
import usb.util

if __name__ == "__main__":
    dev = usb.core.find()

    # was it found?
    if dev is None:
        raise ValueError('Device not found')

    # set the active configuration. With no arguments, the first
    # configuration will be the active one
    dev.set_configuration()

    # get an endpoint instance
    cfg = dev.get_active_configuration()
    intf = cfg[(0,0)]

    ep = usb.util.find_descriptor(
        intf,
        # match the first OUT endpoint
        custom_match = \
        lambda e: \
            usb.util.endpoint_direction(e.bEndpointAddress) == \
            usb.util.ENDPOINT_OUT)

    # assert ep is not None

    # write the data
    # ep.write('test')
    print(ep)
    dev.set_configuration()
    # print(cfg)
    print (dev)
    pass
```
    
Output:
```
DEVICE ID 054c:079c on Bus 000 Address 001 =================
 bLength                :   0x12 (18 bytes)
 bDescriptorType        :    0x1 Device
 bcdUSB                 :  0x200 USB 2.0
 bDeviceClass           :    0x0 Specified at interface
 bDeviceSubClass        :    0x0
 bDeviceProtocol        :    0x0
 bMaxPacketSize0        :   0x40 (64 bytes)
 idVendor               : 0x054c
 idProduct              : 0x079c
 bcdDevice              :  0x100 Device 1.0
 iManufacturer          :    0x1 Sony
 iProduct               :    0x2 ILCE-6300
 iSerialNumber          :    0x3 <redacted by me>
 bNumConfigurations     :    0x1
  CONFIGURATION 1: 500 mA ==================================
   bLength              :    0x9 (9 bytes)
   bDescriptorType      :    0x2 Configuration
   wTotalLength         :   0x27 (39 bytes)
   bNumInterfaces       :    0x1
   bConfigurationValue  :    0x1
   iConfiguration       :    0x0
   bmAttributes         :   0x80 Bus Powered
   bMaxPower            :   0xfa (500 mA)
    INTERFACE 0: Image =====================================
     bLength            :    0x9 (9 bytes)
     bDescriptorType    :    0x4 Interface
     bInterfaceNumber   :    0x0
     bAlternateSetting  :    0x0
     bNumEndpoints      :    0x3
     bInterfaceClass    :    0x6 Image
     bInterfaceSubClass :    0x1
     bInterfaceProtocol :    0x1
     iInterface         :    0x0
      ENDPOINT 0x81: Bulk IN ===============================
       bLength          :    0x7 (7 bytes)
       bDescriptorType  :    0x5 Endpoint
       bmAttributes     :    0x2 Bulk
       wMaxPacketSize   :  0x200 (512 bytes)
       bInterval        :    0x0
      ENDPOINT 0x2: Bulk OUT ===============================
       bLength          :    0x7 (7 bytes)
       bDescriptorType  :    0x5 Endpoint
       bEndpointAddress :    0x2 OUT
       bmAttributes     :    0x2 Bulk
       wMaxPacketSize   :  0x200 (512 bytes)
       bInterval        :    0x0
      ENDPOINT 0x83: Interrupt IN ==========================
       bLength          :    0x7 (7 bytes)
       bDescriptorType  :    0x5 Endpoint
       bEndpointAddress :   0x83 IN
       bmAttributes     :    0x3 Interrupt
       wMaxPacketSize   :   0x20 (32 bytes)
       bInterval        :    0x7
```

As you can see, the data should be coming over the Bulk interface `0x81`, according to Wireshark.

## Conclusion

As I don't know how to write USB drivers, and I have yet to look into using USBPcap to live convert the data, there are a few possible approaches to this moving forward.

- Using USBPcap, filter on the correct interface endpoint and decode the data as it comes in. Pipe this to an image decoder that assembles frames of a video and feeds it into a virtual video source for use.
- Using libusb/pyusb and the Wireshark USB capture, reverse engineer/clone the USB handshake required to communicate with the camera in PC Remote mode. Then listen in on the correct interface endpoint, decode, and feed into virtual video source (like above).
- Wait for Sony to update their Camera Remote SDK for more cameras and use that to interface with the camera.
- Sony releases firmware that turns the camera into a USB webcam source and this is all for nothing.

## Relevant links
- [USBPcap](https://desowin.org/usbpcap/index.html)
- [Wireshark](https://wireshark.org)
- [PyUSB tutorial](https://github.com/pyusb/pyusb/blob/master/docs/tutorial.rst)
- [Zadig](https://zadig.akeo.ie/)
- [Zadig Help](https://github.com/pbatard/libwdi/wiki/FAQ#Help_Zadig_replaced_the_driver_for_the_wrong_device_How_do_I_restore_it)
- [Sony Camera Remote SDK](https://support.d-imaging.sony.co.jp/app/sdk/en/index.html)
