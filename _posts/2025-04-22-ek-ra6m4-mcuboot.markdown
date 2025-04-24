---
layout: post
title:  "Using MCUboot with the RA6M4 - Part 1"
date:   2025-04-22 18:15:00 -0800
---

I wanted to get MCUboot running on the RA6M4 as the first step towards adding firmware upgrade capability to a project I’m working on. There is already a lot of great information available regarding the use of MCUboot with Zephyr, so I’m just going to provide a general overview of the steps I took and concentrate items that are specific to the RA6M4 (or RA Microcontrollers generally).

I started by reading the [Building and using MCUboot with Zephyr][mcuboot-zephyr] document and what follows is a description of the steps I took to get MCUboot running on the RA6M4.

### Creating a Partition Table

For development I’m using an EK-RA6M4 evaluation kit board and so I first needed to update the boards [device tree definition][ek-ra6m4-dts] to add the flash partitions required by MCUboot. Although the RA6M4 contains a dual bank flash I decided to keeps things simple and use it in linear mode, setting aside the first 64KB for the bootloader and dividing the remaining area into two equally sized image slots. To acheive this I added the following to the boards device tree:

{% highlight ruby %}
&flash0 {
	partitions {
		compatible = "fixed-partitions";
		#address-cells = <1>;
		#size-cells = <1>;

		boot_partition: partition@0 {
			label = "mcuboot";
			reg = <0x0 DT_SIZE_K(64)>;
		};
		slot0_partition: partition@10000 {
			label = "image-0";
			reg = <0x00010000 DT_SIZE_K(480)>;
		};
		slot1_partition: partition@80000 {
			label = "image-1";
			reg = <0x00080000 DT_SIZE_K(480)>;
		};
	};
};
{% endhighlight %}

I then had to tell Zephyr to locate any applications I build for this board in slot0. I did this by adding the following to the chosen node in the board device tree:

{% highlight ruby %}
chosen {
	zephyr,code-partition = &slot0_partition;
};
{% endhighlight %}

### Building MCUboot

I decided to configure MCUboot to overwrite the primary image with the upgrade image instead of performing a swap, just to keep things simple. To do this I simply created an EK-RA6M4 specific configuration file (`ek_ra6m4.conf`), containing the following, and placed it into the MCUboot boards folder (`/zephyrproject/bootloader/mcuboot/boot/zephyr/boards/`):

{% highlight ruby %}
CONFIG_BOOT_UPGRADE_ONLY=y
{% endhighlight %}

Next I built MCUboot and programmed it into my EK-RA6M4 board as follows as follows:

1.	To build the bootloader I executed the following command from the zephyr install root folder (zephyrproject):

{% highlight ruby %}
west build -b ek_ra6m4 bootloader/mcuboot/boot/zephyr -p always
{% endhighlight %}

2.	I used west to program the bootloader into the EK-RA6M4 board using the following command:

{% highlight ruby %}
west flash
{% endhighlight %}

When I viewed the output of the console of my EK-RA6M4 board I could see the following, telling me MCUboot is running successfully: 

{% highlight ruby %}
*** Booting MCUboot 6cbea0a24256 ***
*** Using Zephyr OS build v4.1.0-2923-gb469fc188b4f ***
I: Starting bootloader
I: Image index: 0, Swap type: none
E: Unable to find bootable image
{% endhighlight %}

### Building a Bootable Application Image

As can be seen from the console, MCUboot is unable to find a bootable image so the next step is to build one. 

I chose to use the blinky sample and to make it bootable I just added the following to the [project configuration file][blinky-prj-conf]:

{% highlight ruby %}
CONFIG_BOOTLOADER_MCUBOOT=y
{% endhighlight %}

By default MCUboot expects images to be signed and so to build blinky I used one of the signatures provided with MCUboot (these should obviously only be used for development!):

{% highlight ruby %}
west build -p always -b ek_ra6m4 samples/basic/blinky -DCONFIG_MCUBOOT_SIGNATURE_KEY_FILE=\"bootloader/mcuboot/root-rsa-2048.pem\"
{% endhighlight %}

Unfortunately this generated an error:

{% highlight ruby %}
Error: Invalid value for '--align': '128' is not one of '1', '2', '4', '8', '16', '32'.
{% endhighlight %}

The units of programming for the RA6M4 code flash are 128 bytes meaning that data can only be written in blocks of 128 bytes. However, the maximum alignment the tool used to create images (imgtool) can handle is 128 bytes (see `write-block-size` in [r7fa6m4af3cfb.dtsi][r7fa6m4af3cfb-dtsi]). As a quick work around I modified the imgtool to accept an alignment value of up to 128 bytes, these modification can be found in my [fork of the MCUboot repo][mcu-boot-fork].

With this fixes in place I tried the build again, this time getting a different error:

{% highlight ruby %}
Error: Image size (0xffa420) + trailer (0xc280) exceeds requested size 0x78000
{% endhighlight %}

The RA6M4 contains options setting memory, an area of flash that allows certain hardware features to be configured on start-up. This memory is located in the memory map at 0x0100_A100 to 0x0100_A2FF. The Zephyr build system takes information from the device tree and generates the data to be stored in this option setting memory. This is then added to the build output files (*.hex and *.bin). However, the addition of this data causes a problem when using the imgtool supplied with MCUboot. This tool is not happy that I told it (via the partition table) that the image slots are 480KB in size, but due to the addition of this option setting data, the hex/bin files generated when I build the application are much larger and won’t fit into the slot.

As a temporary workaround I decided to remove the option setting data from my build output file (and rely on the option setting data that is present when MCUboot was programmed into the flash). I did this by adding an EK-RA6M4 overlay file to the blinky example. I created a boards folder and placed an overlay file into this (`boards/ek_ra6m4.overlay`) that contained the following:

{% highlight ruby %}
/delete-node/ &option_setting_ofs;
/delete-node/ &option_setting_sas;
/delete-node/ &option_setting_s;
{% endhighlight %}

Now when I built again everything worked!

> NOTE: This is just a temporary solution as it relies on the option settings generated when MCUboot was built and programmed into the device. I’m not sure at this time what a longer term solution might look like….

The final step was to program the blinky application image into the flash (the west tool automatically selected the signed image file):

{% highlight ruby %}
west flash
{% endhighlight %}

Success! Now when I view the console output I see that MCUboot runs first, finds a bootable image and executes it!

{% highlight ruby %}
*** Booting MCUboot 6cbea0a24256 ***
*** Using Zephyr OS build v4.1.0-2837-g4680590fda0d ***
I: Starting bootloader
I: Image index: 0, Swap type: none
I: Bootloader chainload address offset: 0x10000
I: Image version: v0.0.0
I: Jumping to the first imag*** Booting Zephyr OS build v4.1.0-2837-g4680590fda0d ***
LED state: OFF
LED state: ON
LED state: OFF
LED state: ON
{% endhighlight %}

In part 2 I’ll detail how the process for uploading a new image and getting it to run in place of the original...

[mcuboot-zephyr]: https://docs.mcuboot.com/readme-zephyr
[ek-ra6m4-dts]: https://github.com/zephyrproject-rtos/zephyr/blob/main/boards/renesas/ek_ra6m4/ek_ra6m4.dts
[blinky-prj-conf]: https://github.com/zephyrproject-rtos/zephyr/blob/main/samples/basic/blinky/prj.conf
[mcu-boot-fork]: https://github.com/iandmorris/mcuboot/tree/ra_mcu_align_128
[r7fa6m4af3cfb-dtsi]: https://github.com/zephyrproject-rtos/zephyr/blob/main/dts/arm/renesas/ra/ra6/r7fa6m4af3cfb.dtsi

