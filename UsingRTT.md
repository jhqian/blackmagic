# Using RTT

When debugging arm processors, there are three ways for the target to print debug messages on the host: Semihosting, Serial Wire Output SWO, and Real-Time Transfer RTT.

[Black Magic Probe](https://github.com/blacksphere/blackmagic) (BMP) is an open source debugger probe that already implements Semihosting and Single Wire Output. This patch adds Real-Time Transfer RTT output to usb serial port.

- RTT is implemented, not as a user program, but as a serial port device. To read RTT output, use a terminal emulator and connect to the serial port.

- A novel way to detect RTT automatically, fast and convenient.

## Use
This example uses linux as operating system. For Windows and MacOS see the *Operating Systems* section.

In one window open a terminal emulator (minicom, putty) and connect to the usb uart:
```
$ minicom -c on -D /dev/ttyBmpTarg
```

In another window open a debugger:
```
$ gdb
(gdb) target extended-remote /dev/ttyBmpGdb
(gdb) monitor swdp_scan
(gdb) attach 1
(gdb) monitor rtt
(gdb) run
^C
(gdb) monitor rtt status
rtt: on found: yes ident: off halt: off channels: auto 0 1 3
max poll ms: 256 min poll ms: 8 max errs: 10
```

The terminal emulator displays RTT output from the target,
and characters typed in the terminal emulator are sent via RTT to the target.


## gdb commands

The following new gdb commands are available:

- ``monitor rtt``

	switch rtt on

- ``monitor rtt enable``

	switch rtt on

- ``monitor rtt disable``

	switch rtt off

- ``monitor rtt poll `` max_poll_ms min_poll_ms max_errs

	sets maximum time between polls, minimum time between polls, and the maximum number of errors before RTT disconnects from the target. Times in milliseconds. It is best if max_poll_ms/min_poll_ms is a power of two. As an example, if you wish to check for RTT output between once per second to eight times per second: ``monitor rtt poll 1000 125 10``.

- ``monitor rtt status``

	show status.

rtt|found|state
---|---|---
rtt: off|found: no|rtt inactive
rtt: on|found: no|searching for rtt control block
rtt: on|found: yes|rtt active
rtt: off|found: yes|corrupt rtt control block, or target memory access error

A status of `rtt: on found: no` indicates bmp is still searching for the rtt control block in target ram, but has not found anything yet. A status of  `rtt: on found: yes` indicates the control block has been found and rtt is active.

- ``monitor rtt channel``

	enables the first two output channels, and the first input channel. (default)

- ``monitor rtt channel number...``

	enables the given RTT channel numbers. Channels are numbers from 0 to 15, inclusive. Eg. ``monitor rtt channel 0 1 4`` to enable channels 0, 1, and 4.

- ``monitor rtt ident string``

	sets RTT ident to *string*. If *string* contains a space, replace the space with an underscore _. Setting ident string is optional, RTT works fine without.

- ``monitor rtt ident``

	clears ident string. (default)

- ``monitor rtt cblock``

	shows rtt control block data, and which channels are enabled. This is an example control block:
	
```
(gdb) mon rtt cb
cbaddr: 0x200000a0
ch ena cfg i/o buf@        size head@      tail@      flg
 0   y   y out 0x20000148  1024 0x200000c4 0x200000c8   2
 1   y   n out 0x00000000     0 0x200000dc 0x200000e0   0
 2   n   n out 0x00000000     0 0x200000f4 0x200000f8   0
 3   y   y in  0x20000548    16 0x2000010c 0x20000110   0
 4   n   n in  0x00000000     0 0x20000124 0x20000128   0
 5   n   n in  0x00000000     0 0x2000013c 0x20000140   0
 6   n   n in  0x00000000     0 0x00000000 0x00000000   0
 7   n   n in  0x00000000     0 0x00000000 0x00000000   0
 8   n   n in  0x00000000     0 0x00000000 0x00000000   0
 9   n   n in  0x00000000     0 0x00000000 0x00000000   0
10   n   n in  0x00000000     0 0x00000000 0x00000000   0
11   n   n in  0x00000000     0 0x00000000 0x00000000   0
12   n   n in  0x00000000     0 0x00000000 0x00000000   0
13   n   n in  0x00000000     0 0x00000000 0x00000000   0
14   n   n in  0x00000000     0 0x00000000 0x00000000   0
15   n   n in  0x00000000     0 0x00000000 0x00000000   0
```

Channels are listed, one channel per line. The columns are: channel, enabled, configured, input/output, buffer address, buffer size, address of head pointer, address of tail pointer, flag. Each channel is a circular buffer with head and tail pointer.
	
Note the columns `ena` for enabled, `cfg` for configured.
	
Configured channels have a non-zero buffer address and non-zero size.  Configured channels are marked  yes `y` in the column `cfg` . What channels are configured depends upon target software.
	
Channels the user wants to see are marked yes `y` in the column enabled `ena`. The user can change which channels are shown with the `monitor rtt channel` command.

Output channels are displayed, and Input channels receive keyboard input, if they are marked yes in both *enabled* and *configured*.

The control block is cached for speed. In an interrupted program, `monitor rtt` will force a reload of the control block when the program continues.

## Identifier string
It is possible to set an RTT identifier string.
As an example, if the RTT identifier is "IDENT STR":
```
$ gdb
(gdb) target extended-remote /dev/ttyBmpGdb
(gdb) monitor swdp_scan
(gdb) attach 1
(gdb) monitor rtt ident IDENT_STR
(gdb) monitor rtt
(gdb) run
^C
(gdb) monitor rtt status
rtt: on found: yes ident: "IDENT STR" halt: off channels: auto 0 1 3
max poll ms: 256 min poll ms: 8 max errs: 10
```
Note replacing space with underscore _ in *monitor rtt ident*.

Setting an identifier string is optional. RTT gives the same output at the same speed, with or without specifying identifier string.

## Operating systems

[Configuration](https://github.com/blacksphere/blackmagic/wiki/Getting-Started) instructions for windows, linux and macos.

### Windows

After configuration, Black Magic Probe shows up in Windows as two _USB Serial (CDC)_ ports.

Connect arm-none-eabi-gdb, the gnu debugger for arm processors, to the lower numbered of the two COM ports. Connect an ansi terminal emulator to the higher numbered of the two COM ports.

Sample gdb session:
```
(gdb) target extended-remote COM3
(gdb) monitor swdp_scan
(gdb) attach 1
(gdb) monitor rtt
(gdb) run
```

For COM port COM10 and higher, add the prefix `\\.\`, e.g.
```
target extended-remote \\.\COM10
```

Target RTT output will appear in the terminal, and what you type in the terminal will be sent to the RTT input of the target.

### linux
On linux, install [udev rules](https://github.com/blacksphere/blackmagic/blob/master/driver/99-blackmagic.rules). Disconnect and re-connect the BMP. Check the device shows up in /dev/ :
```
$ ls -l /dev/ttyBmp*
lrwxrwxrwx 1 root root 7 Dec 13 07:29 /dev/ttyBmpGdb -> ttyACM0
lrwxrwxrwx 1 root root 7 Dec 13 07:29 /dev/ttyBmpTarg -> ttyACM2
```
Connect terminal emulator to /dev/ttyBmpTarg and gdb to /dev/ttyBmpGdb .

In one window:
```
minicom -c on -D /dev/ttyBmpTarg
```
In another window :
```
gdb
(gdb) target extended-remote /dev/ttyBmpGdb
(gdb) monitor swdp_scan
(gdb) attach 1
(gdb) monitor rtt
(gdb) run
```

### MacOS

On MacOS the tty devices have different names than on linux. On connecting blackmagic to the computer 4 devices are created, 2 'tty' and 2 'cu' devices. Gdb connects to the first cu device (e.g.: `target extended-remote /dev/cu.usbmodemDDCEC9EC1`), while RTT is connected to the second tty device (`minicom -c on -D /dev/tty.usbmodemDDCEC9EC3`). In full:

In one Terminal window, connect a terminal emulator to /dev/tty.usbmodemDDCEC9EC3 :

```
minicom -c on -D /dev/tty.usbmodemDDCEC9EC3
```
In another Terminal window, connect gdb to /dev/cu.usbmodemDDCEC9EC1 :
```
gdb
(gdb) target extended-remote /dev/cu.usbmodemDDCEC9EC1
(gdb) monitor swdp_scan
(gdb) attach 1
(gdb) monitor rtt
(gdb) run
```
RTT input/output is in the window running _minicom_.

## Notes

- Design goal was smallest, simplest implementation that has good practical use.

- RTT code size is 3.5 kbyte - the whole debugger 110 kbyte.

- Because RTT is implemented as a serial port device, there is no need to write and maintain software for different host operating systems. A serial port works everywhere - linux, windows and mac. You can even use an Android mobile phone as RTT terminal.

- Because polling occurs between debugger probe and target, the load on the host is small. There is no constant usb traffic, there are no real-time requirements on the host.

- RTT polling frequency is adaptive and goes up and down with RTT activity. Use *monitor rtt poll* to balance response speed and target load for your use.

- Detects RTT automatically, very convenient.

- When using RTT as a terminal, sending data from host to target, you may need to change local echo, carriage return and/or line feed settings in your terminal emulator.

- Architectures such as risc-v may not allow the debugger access to target memory while the target is running. As a workaround, on these architectures RTT briefly halts the target during polling. If the target is halted during polling, `monitor rtt status` shows `halt: on`.

- Measured RTT speed.

| debugger                  | char/s |
| ------------------------- | ------ |
| bmp stm32f723 stlinkv3    | 49811  |
| bmp stm32f411 black pill  | 50073  |
| bmp stm32f103 blue pill   | 50142  |

This is the speed at which characters can be sent from target to debugger probe, in reasonable circumstances. Test target is an stm32f103 blue pill running an [Arduino sketch](https://github.com/koendv/Arduino-RTTStream/blob/main/examples/SpeedTest/SpeedTest.ino). Default *monitor rtt poll* settings on debugger. Default RTT buffer size in target and debugger. Overhead for printf() calls included.

## Compiling firmware
To compile with RTT support, add *ENABLE_RTT=1*.

Eg. for STM32F103 blue pill:
```
make clean
make PROBE_HOST=stlink ENABLE_RTT=1
```
or for the STM32F411 *[Black Pill](https://www.aliexpress.com/item/1005001456186625.html)*:
```
make clean
make PROBE_HOST=f4discovery BLACKPILL=1 ENABLE_RTT=1
```
Setting an ident string is optional. But if you wish, you can set the default RTT ident at compile time.
For STM32F103 *Blue Pill*:
```
make clean
make PROBE_HOST=stlink ENABLE_RTT=1 "RTT_IDENT=IDENT\ STR"
```
or for STM32F411 *Black Pill*:
```
make clean
make PROBE_HOST=f4discovery BLACKPILL=1 ENABLE_RTT=1 "RTT_IDENT=IDENT\ STR"
```
Note the backslash \\ before the space.

## Links
 - [OpenOCD](https://openocd.org/doc/html/General-Commands.html#Real-Time-Transfer-_0028RTT_0029)
 - [probe-rs](https://probe.rs/) and [rtt-target](https://github.com/mvirkkunen/rtt-target) for the _rust_ programming language.
 - [RTT Stream](https://github.com/koendv/Arduino-RTTStream) for Arduino on arm processors
 - [\[WIP\] RTT support - PR from katyo](https://github.com/blacksphere/blackmagic/pull/833)
