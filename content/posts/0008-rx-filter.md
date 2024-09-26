+++
authors = ["Jasper Devreker"]
title = "MAC RX filter on the  ESP32"
date = "2024-09-26"
description = "Filtering packets in hardware"
+++

This is a more technical blob post about a specific aspect of the ESP32 Wi-Fi hardware; see previous blog posts for a more general overview.

A problem we had, was that our MAC got a bit overwhelmed: we used to put it in promiscous mode, so that it received every single Wi-Fi packet in the channel it was listening to. To solve this, the designer of the Wi-Fi MAC in the ESP32 implemented hardware filtering based on MAC addresses: that way, the software only has to process packets it could be interested in, instead of every packet in the air. However, as with the rest of the Wi-Fi hardware, this is completely undocumented, so if we want to use it, we have to reverse engineer it.

## Approach

From reading the code, we've so far discovered 4 major ways to filter packets:

- based on receiver MAC address
- based on BSSID
- filter out probe request packets
- accept all packets (promiscous mode)

To find out how the filters work, we first looked at the code in Ghidra. This tells us the addresses of the registers and what the proprietary firmware writes to them, but does not tell us what the hardware register does. We can try to make an assumption based on the function name and based on what is written, but this only gets us so far. To check if our assumption is correct, we have to test it by sending a lot of different packets, and seeing which ones get accepted and which ones get dropped.

We use the Scapy Python library to craft the packets we want to send. We do this by first assuming a list of interesting fields (for example, the MAC addresses in a packet) and then generating all possible combinations. Since we can't brute force all MAC addresses, we chose these from a small list. The packets are then sent via a Wi-Fi dongle in managed mode. To make sure that there are no packets getting in from unrelated Wi-Fi stations/APs, we place the setup in the [faraday cage we built in a previous article]({{< ref "posts/0003-faraday-cage.md" >}}). After capturing the packets, we create a script in Python that models what we think the filter does, and compare the actual received packets to the expected received packets. We can then calculate a score based on how correct the model is compared to reality. Using this score, we can gradually improve the model, until we have a perfect score.

As a short recap, here's how the addresses in Wi-Fi frames work: there are 3 (or 4) address fields in a frame. The 'To DS' and 'From DS' bits determine what each field means.

| ![Meaning of addresses in 802.11 frames; CWAP Study Guide – Page 92; Used with permission from PLANET3 WIRELESS, INC.](/images/cwap-mac-address-01.png) | 
|:--:| 
| *Meaning of addresses in 802.11 frames; CWAP Study Guide – Page 92; Used with permission from PLANET3 WIRELESS, INC.* |


## Receiver MAC & BSSID

From reverse engineering, we found that there appear to be two "filter bank" peripherals, each with 2 filters:

- a filter to filter based on the RA (Receiver Address) in the 802.11 frame. This filter consists of:
    - a MAC address
    - a bitmask of which bit of the MAC address to consider in the filter (0 = ignore, 1 = consider)
    - a bit that enables the filter

   This filter can be used when the ESP32 is in station mode and connected to an AP. The filter accepts packets if the packet RA is equal to the filter MAC address (`RA & filter bitmask == filter address & filter bitmask`). If this filter matches, the hardware will automatically send back an ACK control frame.

- a filter to filter based on the BSSID. This filter also consists of:
    - a MAC address
    - a bitmask of which bit of the MAC address to consider in the filter (0 = ignore, 1 = consider)
    - a bit that enables the filter
   
   This filter matches if:
     - the RA matches either the broadcast all-FF address or the BSSID in the filter (using the bitmask in the same way as the MAC filter)
     - AND if there is a BSSID field (see the image above) in the packet, it matches either the broadcast all-FF address, or the BSSID in the filter (using the bitmask in the same way as the MAC filter)
   This filter could be used to filter for broadcast packets in a BSS.

The two different filter types in the filter bank slightly influence each other. If both are enabled at the same time, there is a special case in the BSSID filter: for packets with fromDS = 1 and toDS = 0, it will only accept if the third address (source address) does not match the RA filter.

In the same code that sets the filter banks, there are also three other bits that can be set. The meaning of those bits are not entirely clear, but from observing behaviour:

- two bits are always set/unset together, and seem to let probe requests through when enabled
- one bit (set in `ic_set_rx_policy_ubssid_check`) doesn't seem to do anything

The registers were documented in a machine readable patch file against the official Espressif SVD hardware description (see [this commit](https://github.com/esp32-open-mac/esp32-open-mac/commit/3405d26e4c64a3395ccfb91b8e9e3e65c4d299e0)). We reverse engineered the filter banks enough to implement station mode and access point mode.

