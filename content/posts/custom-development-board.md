---
title: "Custom Development Board"
date: 2020-10-31T12:32:21+02:00
draft: false
---

![IOTE.AI dev board v1 top](/images/devboard/iote_ai_dev_v1_top.jpg)

# Making PCBs

Long gone are the days when manufacturing your own PCB was either expensive or
tedious. A long time ago I wanted to make my own PCBs and acquired the materials and tooling for it.
I needed to print the traces on a transparent film paper, buy photoresist PCBs, UV exposure, etching with
Ferric Chloride (with a heater), cleaning, drilling and so on. I did a few PCBs but the end result wasn't that pretty.

I realized that I actually still have the (1-sided) UV exposure box
somewhere in the storage room. They are still selling that thing for 340eur in the same shop
where I bought it from.

From my PCB factory tools the Dremel drill is probably the only thing that I still use sometimes.

# Ordering PCBs

Nowadays there are of course dozens of places where you can order your PCBs with very low cost. For me, it wouldn't 
be a hard decision if I now had to choose from finding all the stuff needed to create my own PCB versus sending the Gerber 
files with a few clicks and paying for the service. Don't get me wrong though; some people like to create things with their own
hands even though they could just buy it. That's great for them, most likely they have their own workshop with plenty of storage space.

# Tracing embedded software

I have been ordering a bunch of different kinds of development boards but they all lack one thing what I want; a dedicated debug header with trace pins. For some development boards the trace pins are not buried and are available in the pin headers but then it requires jumper wires and it's not ideal. In some boards the trace pins are multiplexed for example as SD card interface pins and could be soldered in.

Hardware tracing embedded software is invaluable. On ARM Cortex-M chips it is the CoreSight on-chip debug and trace technology that you want to have access to. You should have ETM, ITM and DWT cells covered in your debugging toolbox. ITM can be used (via the SWO pin) with a Serial Wire Viewer (SWV) for debugging, but it needs instrumentation in the firmware. DWT, Data Watchpoint and Trace unit, is needed for instance for variable tracing. The king of the hill in the ARM Cortex-M debugging is the Embedded Trace Macrocell (ETM) which gives you a non-intrusive x-ray vision to what is happening in your controller.

Since these development boards don't have what I want, the MIPI20T debug connector with trace pins, I decided to create my own.

# Ordering assembled PCBs

This is the part that blew my mind, how it has changed over time, how easy and affordable it is. I mean ordering assembled low volume PCBs as a hobbyist. The factory that I went for is [JLCPCB](https://jlcpcb.com/). I had to make some compromises on the electronics based on what components are available for the SMT assembly, but all in all it was a great joy to design my own Cortex-M7 development board. A couple of weeks later after sending the manufacturing files a nice little box with the 50x35mm assembled boards were at my local post office.

In the picture in the bottom left corner you can see the pads for the most important feature of this development board, a 2x10 1.27mm pitch pin header for the trace and SWD pins. The other component that I need to hand solder was the USB connector. I've heard that JLCPCB is going to include an option for assembled USB connector soon as well.

![IOTE.AI dev board v1 bottom](/images/devboard/iote_ai_dev_v1_bottom.jpg)


-Petri

