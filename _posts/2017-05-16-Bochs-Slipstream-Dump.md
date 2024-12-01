---
layout: post
author: Savoury SnaX
title: Bochs, slipstream and dumping success
---

This is going to be a large post, for what is essentially a small update :)

I decided the simplest way to write and test the program to communicate with the devkit, was probably to use the Slipstream emulator and debugger along with an IBM PC emulator (Bochs). I quickly hacked up a modification to the gameport driver in bochs to make it read/write from a memory mapped page. I already use this method to share memory between the Slipstream emulator and a simple .NET debugger, so I simply extended the Slipstream emulator in the same way. This means that when the Bochs emulated PC reads/writes to the 0x200 port, the Slipstream emulator will see the changes. The below shot shows Bochs running Turbo Debugger (bottom right), Slipstream video out (top right), and the Slipstream Remote Debugger (left). The Bochs emulator has just set the 0x82 command onto the host PC game port, and slipstream is about to consume the data (See GPIO3 in the ASIC view, at the bottom, left is output, right is input (showing 0x0282)).

![SlipAndBochs](/MSU_Blog/images/SlipAndBochs.png)

The protocol for communicating with the devkit is pretty simple. For reading from the devkit to the host, the devkit waits for bit 9 of port GPIO3 to be set, and captures the low 8bits (the byte being transferred). The devkit then sets bit 10 to the port, and waits for the host to clear bit 9, the devkit then writes 0 to the port to indicate all done. Writing from the host to the devkit is basically the same but the host now manipulates bit 10, and the devkit manipulates bit 9. Below is a dump of my simple host pc program that will dump the entire contents of RAM from the devkit.

<pre>
; Dump FFFFFF bytes from the MSU devkit starting at address 0
; Writes 1 byte at a time, so SLOW

.MODEL  SMALL
.STACK  100h
.DATA
.386

OUTPUTFILE              DB 'DUMP.BIN',0

; ADDRESS, LENGTH
TESTREADDATA            DD  0 , 0FFFFFFh

TDATA			DB 0
HANDLE			DW 0

.CODE
start:
	mov	ax,@data
	mov	ds,ax                   ;set DS to point to the data segment

	; Open file for write
	mov	cx,0
	mov	dx,OFFSET OUTPUTFILE
	mov	ah,03Ch
	int	21h
	mov	[HANDLE],ax

	; init communications - Guarantees we are in a good state for talking to the kit
	mov	dx,512
retry:
	in	ax,dx
	test	ax,0400h
	jz	good
	mov	ax,0000h
	out	dx,ax
	jmp	retry
good:

	; Port in correct state, lets try requesting some data
	mov	ax,0082h
	call	WriteIO
	; Now send the address and length we want to copy
	mov	si,OFFSET TESTREADDATA
	mov	cx,8
doAll:
	mov	al,[si]
	inc	si
	call	WriteIO
	loopw	doAll

	; Now read back all the data we requested
	mov	ecx,[TESTREADDATA+4]
doRead:
	call	ReadIO
	call	WriteToFile
	loopd	doRead

	mov	ah,03Eh
	mov	bx,[HANDLE]
	int	21h


	mov	ah,4ch                  ;DOS: terminate program
	mov	al,0                    ;return code will be 0
	int	21h                     ;terminate the program

WriteToFile:
	pusha

	mov	[TDATA],AL
	mov	bx,[HANDLE]
	mov	cx,1
	mov	DX,OFFSET TDATA
	mov	ah,040h
	int	21h
	popa
	ret

ReadIO:
.loopr:
	in	ax,dx
	test	ax,0200h
	je	.loopr
	mov	bh,al 
	mov	ax,0400h
	out	dx,ax
.loopr2:
	in	ax,dx
	test	ax,0200h
	jne	.loopr2
	xor	ax,ax
	out	dx,ax
	mov	al,bh
	ret

WriteIO:
	or	ax,0200h
	out	dx,ax
.loop:
	in	ax,dx
	test	ax,0400h
	je	.loop  
	xor	ax,ax
	out	dx,ax
.loop2:
	in	ax,dx
	test	ax,0400h
	jne	.loop2
	ret

END start
</pre>

So, using this approach, I successfully dumped the emulator RAM. I also transferred the program to my real PC and from there proceeded to dump the contents of the devkit ram, it took 21 minutes to dump all 16Mb (well technically 1 byte less than 16Mb - count was 0FFFFFFh). It took another hour to get the 16Mb file from the host pc to my modern pc (using a HXC Floppy Emulator, and HJSplit + rar), however I now have a complete dump of the memory and can cross reference it against the emulator and the original memory map. Note my devkit appears to only have 2 * 512K ram chips.

| First Address | Last Address | According to SS4 manual | Observations |
|:-------------:|:------------:|:-----------------------:|:------------:|
|0x000000 | 0x0FFFFF | DRAM0 | RAM |
|0x100000 | 0x7FFFFF | DRAM0 | Mirror of 0x000000-0x0FFFFF Every 1Mb |
|0x800000 | 0xEFFFFF | DRAM1 | Unconnected? -- floating bus possibly |
|0xF00000 | 0xF0FFFF | ROM | reads as 0F FF pairs - does not match any values in my rom |
|0xF10000 | 0xF103FF | PALETTE | only 18 bits of the 32 bits read back, rest are 0 - matches docs |
|0xF10400 | 0xF107FF | BLITTER REGS | 16bytes (00 84 01 00 F0 75 FF 00 00 00 00 00 00 00 00 66) mirrors across whole range |
|0xF10800 | 0xF13FFF | RESERVED | reads as 00 66 pairs |
|0xF14000 | 0xF17FFF | RESERVED | looks like it might be a mirror of a portion of ram |
|0xF18000 | 0xF181FF | DSP SIN/DATA | currently a sine table |
|0xF18200 | 0xF18219 | DSP CONSTANTS | Matches docs |
|0xF1821A | 0xF1827F | RESERVED | looks like there might be more constants in this space |
|0xF18280 | 0xF182FF | DSP REGISTERS | not checked, but look likely
|0xF18300 | 0xF18FFF | DSP DATA/SIN | not checked, but looks likely to be data ram |
|0xF19000 | 0xF1FFFF | NOT DOCUMENTED | Mirror of 0xF18000-0xF18FFF Every 64KB |
|0xF20000 | 0xFBFFFF | ROM | appears to be 0F FF (matches 0xF00000-0xF0FFFF) |
|0xFC0000 | 0xFDFFFF | ROM | 128k ROM appears here |
|0xFE0000 | 0xFFFFFF | ROM | 128k ROM appears here (mirror of 0xFC0000-0xFDFFFF) |


