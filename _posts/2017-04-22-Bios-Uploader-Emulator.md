---
layout: post
author: Savoury SnaX
title: Bios, Uploader and Emulator
---

I've spent some time disassembling and documenting the newly recovered BIOS. I've termed the 3 bios files as DEVSYS5 (from my board), MSUBIOS (the one I recieved when starting work on konix emulation) and EASY (which came from an TXC development directory). I believe the order of age from earliest to latest is DEVSYS5->MSUBIOS->EASY. From the disassembly, it appears MSUBIOS is a hacked variant of the DEVSYS5, designed to save time when developing for the board. EASY seems to rely on a number of additional components (NAND FLASH & MODEM), neither of which I have.

The Slipstream emulator now shows the following (first shot is from the MSUBIOS, second is from DEVSYS5), the colours may not be correct :

![MSUBIOS](/MSU/images/Bios-CP-1.png)

![DEVSYS5](/MSU/images/BIOS-DevSys5.png)

The MSUBIOS basically does the following :

*   Setup GDT/IDT tables and switch to protected mode
*   Initialise Display hardware
*   Copy an image to the display
*   Wiat for communication on GPIO3

So the BIOS in theory waits for the host PC to issue it commands, this is mostly done via the input output ports on the host. Looking at the uploader from the EASY bios, it used port 0x300 & 0x302. However my PC only reads back 0xFFFF from these addresses, so the board I have may be mapped differently. I wrote a quick asm routine to try and scan the IO ports looking for possible locations, however I didn't have any luck - It's possible the board is faulty or uses a port mapping that conflicts with something else in my PC.

