---
layout: post
title:  "Using MCUboot with the RA6M4 - Part 2"
date:   2026-02-13 09:18:00 -0800
---

In [Part 1][mcuboot-part-1] I built the mcuboot binary and then the application binary and then programmed them into the RA6M4 one at a time. There is a easier was to do this by using sysbuild.

### Sysbuild

Sysbuild (System build) basically allows you to build both mcuboot and your application with one command and then program them both into the RA6M4 with another. There are also other uses for sysbuild but I won't cover those here, see the official [sysbuild][zephyr-sysbuild] documentation if you are interested. To build the blinky example and mcuboot and then program them into the RA6M4 use the following commands:

{% highlight ruby %}
west build -b ek_ra6m4 --sysbuild samples/basic/blinky -p always -- -DSB_CONFIG_BOOTLOADER_MCUBOOT=y
west flash
{% endhighlight %}

### Adding a Build Date & Time

When working on firmware upgrades it is useful to include the build date and time in the application binary. This ensures a new binary is produced on each compile and makes it easy to identify if the correct application binary has been loaded (if this build date and time is output via the console).

To embed the build date and time in the blinky sample, and output it via the console, I first added the following to the `main.c` file to add support for logging:

{% highlight ruby %}
#define LOG_LEVEL LOG_LEVEL_DBG
#include <zephyr/logging/log.h>
LOG_MODULE_REGISTER(blinky);
{% endhighlight %}

I then added the following to the main function (just before the `while(1)` loop) to output the build time & date via the console:

{% highlight ruby %}
LOG_INF("Build Time: %s %s", __DATE__, __TIME__);
{% endhighlight %}

Finally, I enabled logging by adding the following to the blinky project configuration file:

{% highlight ruby %}
# Enable logging
CONFIG_LOG=y
{% endhighlight %}

This results in the following console output:

{% highlight ruby %}
*** Booting MCUboot v2.3.0-44-g54732ebf76fa ***
*** Using Zephyr OS build v4.3.0-6613-g2bde9ff2943d ***
I: Starting bootloader
I: Image index: 0, Swap type: none
I: Image index: 0, Swap type: none
I: Primary image: magic=unset, swap_type=0x1, copy_done=0x3, image_ok=0x3
I: Secondary image: magic=unset, swap_type=0x1, copy_done=0x3, image_ok=0x3
I: Boot source: none
I: Image index: 0, Swap type: none
I: Image index: 0, Swap type: none
I: Image index: 0, Swap type: none
I: Image index: 0, Swap type: none
I: Bootloader chainload address offset: 0x10000
I: Image version: v0.0.0
I: Jumping to the first image sl?LED state: OFF
*** Booting Zephyr OS build v4.3.0-6613-g2bde9ff2943d ***
[00:00:00.000,000] <inf> blinky: Build Time: Feb 13 2026 08:50:07
LED state: ON
LED state: OFF
{% endhighlight %}

### Adding a Version Number

As can be seen in the console output above, the image version number is currently set to v0.0.0:

{% highlight ruby %}
I: Image version: v0.0.0
{% endhighlight %}

Changing this is easy, I simply added a file named `VERSION` to the root of the blinky sample folder i.e. at the same level as the `prj.conf` file and then set the contents of this file to the following:

{% highlight ruby %}
VERSION_MAJOR = 1
VERSION_MINOR = 0
PATCHLEVEL = 0
VERSION_TWEAK = 0
EXTRAVERSION =
{% endhighlight %}

Now when I rebuild the application and program it into the RA6M4 I see the following output on the console:

{% highlight ruby %}
*** Booting MCUboot v2.3.0-44-g54732ebf76fa ***
*** Using Zephyr OS build v4.3.0-6613-g2bde9ff2943d ***
I: Starting bootloader
I: Image index: 0, Swap type: none
I: Primary image: magic=unset, swap_type=0x1, copy_done=0x3, image_ok=0x3
I: Secondary image: magic=unset, swap_type=0x1, copy_done=0x3, image_ok=0x3
I: Boot source: none
I: Image index: 0, Swap type: none
I: Image index: 0, Swap type: none
I: Image index: 0, Swap type: none
I: Bootloader chainload address offset: 0x10000
I: Image version: v1.0.0
I: Jumping to the first image sl?*** Booting Zephyr OS build v4.3.0-6613-g2bde9ff2943d ***
[00:00:00.000,000] <inf> blinky: Build Time: Feb 13 2026 09:11:49
{% endhighlight %}

Next I updated the blinky sample to print out the version number along with the build date and time. To do this I added the following include file to `main.c`:

{% highlight ruby %}
#include <zephyr/app_version.h>
{% endhighlight %}

Next, I updated the statement that prints out the build date and time to also print the application version number, as follows:

{% highlight ruby %}
LOG_INF("Version: %s Build Time: %s %s", APP_VERSION_STRING, __DATE__, __TIME__);
{% endhighlight %}

Finally, I updated the statement that prints the LED state information to use the logging mechanism instead of printk, just to keep things consistent:

{% highlight ruby %}
LOG_INF("LED state: %s", led_state ? "ON" : "OFF");
{% endhighlight %}

Now when I rebuild the application and program it into the RA6M4 I see the following output on the console:

{% highlight ruby %}
*** Booting MCUboot v2.3.0-44-g54732ebf76fa ***
*** Using Zephyr OS build v4.3.0-6613-g2bde9ff2943d ***
I: Starting bootloader
I: Image index: 0, Swap type: none
I: Image index: 0, Swap type: none
I: Primary image: magic=unset, swap_type=0x1, copy_done=0x3, image_ok=0x3
I: Secondary image: magic=unset, swap_type=0x1, copy_done=0x3, image_ok=0x3
I: Boot source: none
I: Image index: 0, Swap type: none
I: Image index: 0, Swap type: none
I: Image index: 0, Swap type: none
I: Image index: 0, Swap type: none
I: Bootloader chainload address offset: 0x10000
I: Image version: v1.0.0
I: Jumping to the first image sl?*** Booting Zephyr OS build v4.3.0-6613-g2bde9ff2943d ***
[00:00:00.000,000] <inf> blinky: Version: 1.0.0 Build Time: Feb 13 2026 12:43:26
[00:00:00.000,000] <inf> blinky: LED state: OFF
[00:00:01.000,000] <inf> blinky: LED state: ON
[00:00:02.000,000] <inf> blinky: LED state: OFF
{% endhighlight %}

Now I'm pretty much ready to make the changes required to upload a new image and get it to run in place of the original. I'll detail how I do that in part 3...

[mcuboot-part-1]: https://iandmorris.com/2025/04/23/ek-ra6m4-mcuboot.html
[zephyr-sysbuild]: https://docs.zephyrproject.org/latest/build/sysbuild/index.html


