+++
authors = ["Jasper Devreker"]
title = "Reverse engineering ESP32 Wi-Fi driver: the road ahead"
date = "2024-05-24"
description = "Estimating the effort needed to re-implement the Wi-Fi driver"
+++

First, a short recap of what this project is doing: the Wi-Fi stack for the ESP32 (a popular, cheap microcontroller) is provided through a binary blob. We're trying to reverse engineer the software and hardware to build our own open-source Wi-Fi stack. This will enable features that the current, closed source ESP32 Wi-Fi implementation does not have, for example 802.11s mesh networking. It will also improve the auditability of the code.

This is the fifth article in a series about reverse engineering the ESP32 Wi-Fi networking stack, with the goal of building our own open-source MAC layer. In the [previous articles]({{< ref "posts" >}}) in this series, we reverse engineered hardware registers for transmitting and receiving Wi-Fi packets. We wrote some code that implements association with an open access point and then connects the transmit and receive functions to the ESP32 IP networking stack. We also built a low-cost Faraday cage to be able to test our implementation without outside interference.

# Hardware initialization

One thing we haven't handled so far, is initialization of the Wi-Fi peripheral: our current approach is to let the binary blob initialize the hardware, after which we kill the FreeRTOS task that is responsible for handling everthing related to Wi-Fi and replace the Wi-Fi interrupt with our own. We will eventually want to also reverse engineer and reimplement this radio initialization; so let's look at the road ahead!

First of all, what happens in this hardware initialization? The main function responsible for initializing the hardware is `esp_phy_enable`.  This function mainly does RF calibration to compensate for imperfections of the hardware. According to the data sheet, this does, at least: I/Q phase matching; antenna matching; compensating carrier leakage, baseband nonlinearities, power amplifier nonlinearities and RF nonlinearities. It also sets the frequency and power that packets will be received/transmitted on.

On the ESP32, interacting with hardware peripherals happens through memory writes/reads: instead of writing to RAM, you write to the address space of a peripheral; each peripheral has their own memory space, and within those memory spaces, each memory address (register) has a different function. We patched Espressifs fork of QEMU to log all accesses to peripheral registers related to Wi-Fi; in addition to logging the addresss and value written/read, we also log the stacktrace, so we can see what functions are writing to what addresses, and what functions are calling those functions, and so on.

All in all, the complete radio initialization takes 53286 peripheral accesses; in comparison, sending a Wi-Fi packet takes about 10, so we still have a lot of work ahead to implement this. To visualize what this initialization consists of, we converted this list of peripheral memory accesses and stack traces to a flame graph:

[Link to flame graph visualisation](https://esp32-open-mac.be/coverage/branches/main/)

In the above flame graph, the width of each bar is linearly proportional to the amount of peripheral memory accesses; in order to not hide smaller functions, we rescaled each bar to `amount_of_accesses**0.4`:

Our current approach of reimplementing the radio initalization, is to one-by-one replace each function with our own implementation, then test that it can still connect to the AP and be pinged.

After browsing the visualisation for a bit, it appears that the hardware initialisation is very complex, and there don't appear to be any quick wins. It's likely that in the near future, we won't have a complete way to init the hardware.

# MAC stack implementation

In parallel with radio initialization, we can start to implement the Wi-Fi MAC stack, responsible for:

- scanning APs
- authenticating/associating with APs (also WPA2 protected APs)
- changing rate/TX power in response to for example dropped packets
- ...

We would like to avoid having to entirely write this from scratch ourselves as well, so we'll likely look at using FreeBSDs 802.11 code (Espressif, the manufacturer of the chip and developer of the binary blob, also based their MAC stack on the FreeBSD stack).

At the moment, we can already send/receive packets and configure the packet filter. What we can't yet do, is configure other parameters of the packets (for example rate, modulation, frequency or TX power): the parameters that we currently send packets with, are the default parameters after the radio has been initialized: channel 1 and the lowest rate there is. To have a performant Wi-Fi stack, we'll need to reverse the hardware a bit more to be able to set those parameters.
