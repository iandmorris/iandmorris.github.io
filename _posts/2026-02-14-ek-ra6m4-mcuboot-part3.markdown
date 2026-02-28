---
layout: post
title:  "Using MCUboot with the RA6M4 - Part 3"
date:   2026-02-14 15:25:00 -0800
---

In part 3 I'm going to detail how to load a new image into the RA6M4 and how to get MCUboot to switch to this new image. First, a couple of decisions have to be made:

1. Which physical interface will be used to transfer the new image from the host (PC) to the RA6M4 i.e. UART, Bluetooth, Ethernet etc.
2. Which protocol will be used to transfer the new image i.e. xmodem, FTP, SMP etc.

To keep things simple I'm going to use the UART interface as the physical transport. On top of this I'm going to use the SMP (Simple Management Protocol) as this is well supported by Zephyr and there are also a number of tools available which support SMP on the client (PC) side.

### MCUmgr subsystem

I'm going to modify the blinky sample from [part 2][mcuboot-part-2] to include support for the MCUmgr subsystem. To do this I just need to add the following to my `prj.conf` file:

{% highlight ruby %}
# Enable MCUmgr and dependencies.
CONFIG_NET_BUF=y
CONFIG_ZCBOR=y
CONFIG_CRC=y
CONFIG_MCUMGR=y
CONFIG_STREAM_FLASH=y
CONFIG_FLASH_MAP=y

# Some command handlers require a large stack.
CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=2304
CONFIG_MAIN_STACK_SIZE=2176

# Ensure an MCUboot-compatible binary is generated.
CONFIG_BOOTLOADER_MCUBOOT=y

# Enable flash operations.
CONFIG_FLASH=y

# Enable most core commands.
CONFIG_IMG_MANAGER=y
CONFIG_MCUMGR_GRP_IMG=y
CONFIG_MCUMGR_GRP_OS=y

# Enable the serial MCUmgr transport.
CONFIG_BASE64=y
CONFIG_UART_MCUMGR=y
CONFIG_MCUMGR_TRANSPORT_UART=y
CONFIG_CONSOLE=y

CONFIG_REBOOT=y
{% endhighlight %}

I then added the following to my board overlay file (`ek_ra6m4.overlay`) to tell MCUmgr to use the UART:

{% highlight ruby %}
/ {
	chosen {
		zephyr,uart-mcumgr = &uart0;
	};
};
{% endhighlight %}

With the MCUmgr subsystem enabled my application now includes an SMP server which listens for SMP packets on the UART. I'll rebuild it and flash it into the RA6M4 as follows:

{% highlight ruby %}
west build -b ek_ra6m4 --sysbuild samples/basic/blinky -p always -- -DSB_CONFIG_BOOTLOADER_MCUBOOT=y
west flash
{% endhighlight %}

### mcumgr-client

I'm going to use a tool called `mcumgr-client` on my PC. This will communicate with the RA6M4 via SMP over the UART and will allow me to send firmware images to the RA6M4. I followed the instructions on the [mcumgr-client][mcumgr-client] Github page to download and install the tool.

Once installed, I confirmed the tool was working and could communicate with the MCUmgr server running on my RA6M4, by sending the following command:

{% highlight ruby %}
mcumgr-client -d /dev/cu.usbserial-A904CZZ7 list
{% endhighlight %}

This command lists what images the RA6M4 contains and it returned the following:

{% highlight ruby %}
mcumgr-client 0.0.8, Copyright © 2024 Vouch.io LLC

23:11:43 [INFO] send image list request
response: {
  "images": [
    {
      "image": 0,
      "slot": 0,
      "version": "1.0.0",
      "hash": "e1ec4402a1afd35a014ec3cb181cc15d0225da38e70c377cb8d4e2c498a87943",
      "bootable": true,
      "pending": false,
      "confirmed": true,
      "active": true,
      "permanent": false
    }
  ],
  "splitStatus": "NotApplicable"
}%
{% endhighlight %}

Next I created a new image file to download. I wanted to be able to differentiate this image from the one currently programmed into the RA6M4 so I increased the version number in my `VERSION` file by setting it as follows:

{% highlight ruby %}
VERSION_MAJOR = 1
VERSION_MINOR = 1
PATCHLEVEL = 0
VERSION_TWEAK = 0
EXTRAVERSION =
{% endhighlight %}

I then rebuilt my application, generating a new image file, as follows:

{% highlight ruby %}
west build -b ek_ra6m4 --sysbuild samples/basic/blinky -p always -- -DSB_CONFIG_BOOTLOADER_MCUBOOT=y
{% endhighlight %}

Once the image was created I used to following command to get `mcumgr-client` to upload it via UART to the RA6M4:

{% highlight ruby %}
mcumgr-client -d /dev/cu.usbserial-A904CZZ7 upload ./build/blinky/zephyr/zephyr.signed.bin
mcumgr-client 0.0.8, Copyright © 2024 Vouch.io LLC

23:24:20 [INFO] upload file: ./build/blinky/zephyr/zephyr.signed.bin
23:24:20 [INFO] flashing to slot 1
23:24:20 [INFO] 59784 bytes to transfer
  [00:00:00] [=========================================================================================================================================================] 58.38 KiB/58.38 KiB (0s)23:24:21 [INFO] upload took 1s
{% endhighlight %}

Now, when I run the list images command again, I see the existing image and the new one I just uploaded:

{% highlight ruby %}
mcumgr-client -d /dev/cu.usbserial-A904CZZ7 list
mcumgr-client 0.0.8, Copyright © 2024 Vouch.io LLC

23:24:59 [INFO] send image list request
response: {
  "images": [
    {
      "image": 0,
      "slot": 0,
      "version": "1.0.0",
      "hash": "e1ec4402a1afd35a014ec3cb181cc15d0225da38e70c377cb8d4e2c498a87943",
      "bootable": true,
      "pending": false,
      "confirmed": true,
      "active": true,
      "permanent": false
    },
    {
      "image": 0,
      "slot": 1,
      "version": "1.1.0",
      "hash": "3b39e086ab70eb1926d3ac779e0467ea31fc7d71bc03ac3427043304d3fb49fa",
      "bootable": true,
      "pending": false,
      "confirmed": false,
      "active": false,
      "permanent": false
    }
  ],
  "splitStatus": "NotApplicable"
}%
{% endhighlight %}

To get MCUboot to use the new (version 1.1.0) image instead of the existing image I have to run the `test` command (with the image hash):

{% highlight ruby %}
mcumgr-client -d /dev/cu.usbserial-A904CZZ7 test 3b39e086ab70eb1926d3ac779e0467ea31fc7d71bc03ac3427043304d3fb49fa

mcumgr-client 0.0.8, Copyright © 2024 Vouch.io LLC

23:28:21 [INFO] set image pending request
{% endhighlight %}

Now when I run the list images command I see the new image marked as pending:

{% highlight ruby %}
mcumgr-client -d /dev/cu.usbserial-A904CZZ7 list

mcumgr-client 0.0.8, Copyright © 2024 Vouch.io LLC

23:29:50 [INFO] send image list request
response: {
  "images": [
    {
      "image": 0,
      "slot": 0,
      "version": "1.0.0",
      "hash": "e1ec4402a1afd35a014ec3cb181cc15d0225da38e70c377cb8d4e2c498a87943",
      "bootable": true,
      "pending": false,
      "confirmed": true,
      "active": true,
      "permanent": false
    },
    {
      "image": 0,
      "slot": 1,
      "version": "1.1.0",
      "hash": "3b39e086ab70eb1926d3ac779e0467ea31fc7d71bc03ac3427043304d3fb49fa",
      "bootable": true,
      "pending": true,
      "confirmed": false,
      "active": false,
      "permanent": false
    }
  ],
  "splitStatus": "NotApplicable"
}%
{% endhighlight %}

Finally, I issue the `reset` command to get MCUboot to boot this new image:

{% highlight ruby %}
mcumgr-client -d /dev/cu.usbserial-A904CZZ7 reset
{% endhighlight %}

When I monitor the output from the console on the RA6M4, I see that new new image (version 1.1.0) is now running:

{% highlight ruby %}
*** Booting MCUboot v2.3.0-44-g54732ebf76fa ***
*** Using Zephyr OS build v4.3.0-6613-g074a609436a3 ***
I: Starting bootloader
I: Image index: 0, Swap type: none
I: Image index: 0, Swap type: none
I: Primary image: magic=bad, swap_type=0x1, copy_done=0x2, image_ok=0x2
I: Secondary image: magic=unset, swap_type=0x1, copy_done=0x3, image_ok=0x3
I: Boot source: none
I: Image index: 0, Swap type: none
I: Image index: 0, Swap type: none
I: Image index: 0, Swap type: none
I: Image index: 0, Swap type: none
I: Bootloader chainload address offset: 0x10000
I: Image version: v1.1.0
I: Jumping to the first image slot?*** Booting Zephyr OS build v4.3.0-6613-g074a609436a3 ***
[00:00:00.000,000] <inf> blinky: Version: 1.1.0 Build Time: Feb 14 2026 15:20:50
[00:00:00.000,000] <inf> blinky: LED state: OFF
[00:00:01.000,000] <inf> blinky: LED state: ON
[00:00:02.000,000] <inf> blinky: LED state: OFF
{% endhighlight %}

Basically that's it. I've used MCUboot and MCUmgr to upload a new image to my RA6M4!

Hopefully someone finds this useful, stay tuned for further posts regarding the use of Zephyr on RA Microcontrollers!

[mcuboot-part-2]: https://iandmorris.com/2026/02/13/ek-ra6m4-mcuboot-part2.html
[mcumgr-client]: https://github.com/vouch-opensource/mcumgr-client
