---
layout: post
author: Savoury SnaX
title: Hunting for IO ports
---

Since I haven't been able to succesfully locate the IO ports from the host PC, I took to trying to figure it out from the hardware. Thankfully the ISA bus pinout is fairly simple, and after a bit of tracing, I narrowed down the important area of the board :

![IO Area](/MSU_Blog/images/IO-Area.png)

Most of the above chips are latches, however the chip labelled U16 is the interesting one, this recieves A1/A4-A9/IORW/IORD/AEN and is almost certainly the address decode for the host IO port. Removing the label on the chip reveals it as a palce16v8h-15pc/4, which is an early form of programmable logic chip - this is unfortunate, as the documentation indicates we can't read back the program due to security fuses. An alternative approach that may work, is to electrically probe the pins looking for possible responses on one of the IO pins. The hope is, that if we can figure out the address that causes the chip to react, we will know the IO port for communicating from the host. I`ll have to try hooking it up to an Arduino and write a program to run through the combinations.

In the mean time, I also ordered a few ZIF sockets and some replacement EEPROMS (W27C512-45Z - which should work as a direct replacement for the EPROMS). This allows me to replace the EPROM pair (Odd/Even), the ZIF sockets allow me to replace the chips significantly more easily than the DIP sockets, and means I can replace the BIOS with my own code, if I fail to get the IO port communication working.

The MSUBIOS doesn't wait for joystick input, so at present I've replaced the DEVSYS5 BIOS on the board with that one. If I find the IO port, I'm pretty sure I can send it commands. The following commands are the most interesting :

*   0x80 = WriteToMemory - allows direct copying from host PC to devkit
*   0x81 = ExecuteMemory - allows us to set the execution address of the devkit (effectively, run the code we uploaded with 0x80)
*   0x82 = ReadFromMemory - Dump devkit memory back to the host.

