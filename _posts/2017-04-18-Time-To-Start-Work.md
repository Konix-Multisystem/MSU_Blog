---
layout: post
author: Savoury SnaX
title: Time to start work
---

Finally got the motivation/time to start looking at the MSU devkit I acquired. My old 386 motherboard decided to pack in, so I nabbed a new (read old) replacement from ebay. The devkit is ISA slot based, so needs an old PC in order to function.

![Devkit and Board](/MSU_Blog/images/Board_And_Devkit.jpg)

The oscilloscope was used to verify that the devit produces some sort of display from the VGA port. The Horizontal and Vertical syncs look to be PAL style (seperate syncs though), unfortunately I don't currently have a display capable of syncing to a 15Khz signal.

I've dumped the EPROMs from the board, they don't match the version I have (taken from a different devkit IIRC). This means I have at least 3 different BIOS revisions that all come from some point during the MSU lifetime. 

I`m going to spend some time reverse engineering the BIOS and get it running under the slipstream emulator.
