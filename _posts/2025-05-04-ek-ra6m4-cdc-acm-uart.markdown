---
layout: post
title:  "Using the USB Interface as the Console"
date:   2025-05-04 20:16:00 -0800
---

Samples such as blinky use the console to output information about the state of the application. When using the EK-RA6M4 the console output is routed to UART0 by default and therefore a Serial-USB converter is required to view the information via a terminal emulator running on a PC:

{% highlight ruby %}
*** Booting Zephyr OS build v4.1.0-3496-g03ce34defdf4 ***
LED state: OFF
LED state: ON
LED state: OFF
{% endhighlight %}

In this blog post I’ll detail how to output the console information to the USB interface on the RA6M4, allowing it to be viewed on a PC without the use of a Serial-USB converter. I will use the USB Communication Device Class (CDC) Abstract Control Model (ACM) driver which is provided with Zephyr. I will also use the new Zephyr USB device driver.

Let’s use the blinky sample as a starting point and make the modifications required to send the console data to the USB interface instead of the UART. First we must modify the EK-RA6M4 [Board Device Tree Defintion][ek-ra6m4-dts] to enable the CDC ACM driver on the USB interface. We do this by modifying the USB Full-Speed node as follows:

{% highlight ruby %}
	pinctrl-0 = <&usbfs_default>;
	pinctrl-names = "default";
	maximum-speed = "full-speed";
	status = "okay";
	zephyr_udc0: udc {
		status = "okay";
		cdc_acm_uart0: cdc_acm_uart0 {
			compatible = "zephyr,cdc-acm-uart";
		};
	};
};
{% endhighlight %}

Next route the console output to the cdc-acm-uart interface by modifying the Zephyr chosen value as follows:

{% highlight ruby %}
zephyr,console = &cdc_acm_uart0;
{% endhighlight %}

Finally modify the blinkly sample [project configuration file][blinky-prj-conf] to tell the build system to include support for the Zephyr USB Device Driver by adding the following configuration options:

{% highlight ruby %}
CONFIG_CDC_ACM_SERIAL_INITIALIZE_AT_BOOT=y
CONFIG_CDC_ACM_SERIAL_PID=0x0004
CONFIG_CDC_ACM_SERIAL_PRODUCT_STRING="USBD console sample"
{% endhighlight %}

Now rebuild the blinky sample, program it into the EK-RA6M4 board and connect a cable to the USB full speed interface (J11). This will cause a new serial interface to appear on the PC and when you use a terminal emulator to view the output on this interface you should see something like the following:

{% highlight ruby %}
*** Booting Zephyr OS build v4.1.0-3496-g03ce34defdf4 ***
LED state: OFF
LED state: ON
LED state: OFF
{% endhighlight %}

Hopefully someone finds this useful, stay tuned for further posts regarding the use of Zephyr on RA Microcontrollers!

[ek-ra6m4-dts]: https://github.com/zephyrproject-rtos/zephyr/blob/main/boards/renesas/ek_ra6m4/ek_ra6m4.dts
[blinky-prj-conf]: https://github.com/zephyrproject-rtos/zephyr/blob/main/samples/basic/blinky/prj.conf
