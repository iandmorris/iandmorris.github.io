---
layout: post
title:  "Getting Started with Zephyr on the RA8M1"
date:   2024-07-04 15:46:24 -0700
---

Developing firmware for a Microcontroller typically requires a number of
components; an IDE, a toolchain and an SDK for the target MCU. A lot of
semiconductor vendors provide all of these in a single package, making
installation easy. For RA Microcontrollers, Renesas provides the FSP Platform
installer. This is a single file which once downloaded and installed provides
an IDE (the Eclipse based e2 Studio), a complete cross compilation toolchain
and a set of drivers and middleware (the Flexible Software Package or FSP).

By contrast, getting started with the Zephyr RTOS on Renesas RA
Microcontrollers requires the installation of a number of different components
and is therefore a multi-step process.

### Install the Zephyr Development Environment

The first step is to install and configure a Zephyr Development Environment.
This requires installing a number of dependencies (such as CMake, Python etc.),
the Zephyr RTOS itself and a Zephyr SDK (toolchain). Follow the steps in the
Zephyr Getting Started guide to get your development environment setup:

[Getting Started Guide — Zephyr Project Documentation][zephyr-getting-started]

**NOTE:** When you get to the step titled "Build the Blinky Sample" build for
whichever Renesas RA Evaluation Kit you have at hand. For example, if you have
an EK-RA6M4 then use the following commands to build and flash:
{% highlight ruby %}
west build samples\basic\blinky -b ek_ra6m4 -p always
west flash
{% endhighlight %}

After following these instructions, you should have a fully functioning Zephyr
Development Environment which you have just proved out by building and flashing
a simple blinky example into a Renesas Evaluation Kit board.

[zephyr-getting-started]: https://docs.zephyrproject.org/latest/develop/getting_started/index.html


