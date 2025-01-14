+++
authors = ["Jasper Devreker"]
title = "Entirely eliminating the FreeRTOS dependency"
date = "2025-01-14"
description = "Entirely eliminating the FreeRTOS dependency"
+++

We're writing an open source 802.11 (Wi-Fi) stack for the ESP32. We can currently already connect to an open network, and we can even set the ESP32 to be an open network itself. The biggest hurdle in having an entirely open source stack, is the radio calibration: this is very complex on the ESP32. To get the hardware initialized, we used to call the proprietary API, which would then internally start a FreeRTOS task that handles setting up the Wi-Fi peripheral. After the calibration is done, we kill the task and from then on only run open source code.

This approach was not ideal, since it has a dependency on FreeRTOS; the task also allocates a lot of buffers, which will never be free'd and just take up memory. To combat this, we've gone through the Wi-Fi task, looked at what was needed to properly initialize the hardware and pulled that code out of the proprietary task into our hardware initialization function.

# Results

While pulling out the functions, we also tried to remove as much binary blobs as possible from the build process; we're now down to 3 blobs we still need (libphy.a and librtc.a for the hardware calibration; libpp.a for some hardware initialization functions we haven't ported yet).

This also saved quite a lot of memory: the proprietary task allocates buffers to receive packets in, which are then never used, since we allocate our own buffers in the open source code. We save a total of 18828 bytes (18 kB) by not doing this. In the regular world, it would be insane to spend time on just getting back 18 kB of RAM, but remember, this is embedded: there is only 320 kB of DRAM on the ESP32; so we save ~5% of RAM doing this.

This should also make porting to other ESP32 variants easier, since it's now clearer what needs to be ported.
