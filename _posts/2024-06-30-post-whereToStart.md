---
title: "Where To Start"
categories:
  - Overview
tags:
  - ECU
  - Jaguar
  - AJ27
---

![AJ27 ECU]({{ site.url }}{{ site.baseurl }}/assets/images/ECU1.jpg)

If you need to modify the operation of an older vehicle, how do you go about it? For example, the pedal position sensor or mass air flow sensor is obsolete, and you want to use an alternative part. Or you want to tweak the tuning, and the model is not supported by the common tuning companies. In this blog I'll describe the techniques I have used to analyze and modify the Engine Control Unit of a [Jaguar AJ27](https://en.wikipedia.org/wiki/Jaguar_AJ-V8_engine) engine. The AJ27 engine was fitted to the Jaguar X100 (XK8 and XKR) and X308 (XJ8 and XJR) for US model years 2000-03.

When getting started, some questions that need answering include:

* What CPU is used in the ECU?
* How are the engine sensors and actuators connected to the CPU?
* How can I get a copy of the firmware?
* How can I analyze the firmware?
* How do I modify and upload the firmware?

As a quick preview, some of the approaches I'll be using are as follows:

For identifying the CPU, the first port of call is internet searches of tuning company data, academic papers, on-line car forums and so on. Following on from that, examination of the ECU hardware and markings on the individual chips may be needed.

Determining connectivity between the CPU and engine components can be tackled a couple of ways. Tracing of the PCB is extremely tedious, if even possible, but if achievable it yields all the required information. An alternative approach involves decoding the [OBD2](https://en.wikipedia.org/wiki/OBD-II_PIDs) firmware routines of well known services (e.g. MAF reading or engine coolant temperature reading) and so identifying the variables/memory locations used to read and control these components.

A copy of firmware can usually be extracted from the ECU hardware, either by reading the memory chips if they are separate from the CPU (EPROMs) or downloading the memory of the CPU (on-board flash) using the CPU debugging/programming capabilities like [JATG](https://en.wikipedia.org/wiki/JTAG) or a manufacturer specific protocol. The car manufacturer software tools can also be examined, since they will typically include copies of the firmware for updating/reprogramming the ECU. For example, Jaguar uses a set of tools called SDD/IDS.

Firmware analysis is typically tackled using software analysis tools, with two of the most popular being IDA - developed by [hex-rays](https://hex-rays.com/), or the free and open source tool [Ghidra](https://ghidra-sre.org/) - developed by the US NSA.

Finally, installing modified firmware requires re-use of the car manufacturer developed software tools, or alternatively using the CPU debugging/programming capabilities.

In the next post, I'll kick things off by examining ECU hardware in more detail, using the AJ27 ECU as the example target.