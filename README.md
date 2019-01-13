# blink_bluepill_rust

Trying to blink an LED on a 1.35€ "blue pill" STM32F103C8 board.
I guess things won't work on first try so I take notes in this file.

# Chapter 1 and 2 of The Embedded Rust Book. (Install to "Hello, world!")

## Installing

I already hade Rustup installed. I removed the unsued toolchains and then followed instructions in chapter 1.3 and 1.3.3 (I used Windows, don't judge) of [The Embedded Rust Book](https://rust-embedded.github.io/book/intro/index.html).
I also wanted to add cargo-generate as told in chapter 1.2 (`cargo install cargo-generate`), but at some point I required to install msvc which is a really 1.1Gb download just a C compiler on windows. Since it is an optionnal step I skept it.

I upgraded y aliexpress clone ST-LINK V2 firmware to latest version using `ST-LinkUpgrade.exe` found in "ST Link utility" by ST.

## UNEXPECTED idcode: 0x2ba01477

After following chapters 1.3 and 1.3.3, I tryed starting OpenOCD to check that if found my STLink-V2-1 programmer and my Blue Pill board. The Book says to type : `openocd -f interface/stlink-v2-1.cfg -f target/stm32f3x.cfg`but since my board has an stm32f103, I used `openocd -f interface/stlink-v2-1.cfg -f target/stm32f1x.cfg`:

```
D:\code\OpenOCD\bin>openocd -f interface/stlink-v2-1.cfg -f target/stm32f1x.cfg
GNU MCU Eclipse 64-bit Open On-Chip Debugger 0.10.0+dev-00352-gaa6c7e9b (2018-10-20-06:24)
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
WARNING: interface/stlink-v2-1.cfg is deprecated, please switch to interface/stlink.cfg
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
none separate
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : clock speed 950 kHz
Info : STLINK v2 JTAG v32 API v2 SWIM v7 VID 0x0483 PID 0x3748
Info : using stlink api v2
Info : Target voltage: 3.204230
Warn : UNEXPECTED idcode: 0x2ba01477
Error: expected 1 of 1: 0x1ba01477
in procedure 'init'
in procedure 'ocd_bouncer'
```

That does not work, the interesting lines are:
```
Warn : UNEXPECTED idcode: 0x2ba01477
Error: expected 1 of 1: 0x1ba01477
```

The idcode returned by the CPU does is not the expected one. That is not completely surprising: I bought the chipest board from aliexpress, and thought advertised havinf an stm32f103 chip from ST Micro, it comes with an advertised-as-perfect-replacement cs32f103c8t6 by CKS. It supposed to be a perfect clone (they do not even provide a datasheet for it), but this part returns a slightly different idcode.

The `idcode` is if cheap identifier. It is part of the [JTAG](https://en.wikipedia.org/wiki/JTAG) protocol. (We do not use JTAG here but the STLink protocol, which IIUC adds the possibility to use a simpler/cheaper connection between some ST Micro chips and the computer). At address (0x0) the protocol allow the chip to expose an identifier called `DPIDR` (for 'Debug Port Identification register', see chapter 2.2.5 of [ARM Debugger Interface Architecture Specification](https://static.docs.arm.com/ihi0031/d/debug_interface_v5_2_architecture_specification_IHI0031D.pdf). The documentation says that bits 28 to 31 contains `Revision code. The meaning of this field is IMPLEMENTATIONDEFINED.`.
Since only bits 28 and 29 are different, we can expect that the chip is still compatible, and create a new configuration file for OpenOCD tu just tell him to expect the actually received idcode.

I copied the `openocd\scripts\target\stm32f1x.cfg` file, naming the copy `cs32f1x.cfg` and changed:
 * the name of the chip:
```
if { [info exists CHIPNAME] } {
   set _CHIPNAME $CHIPNAME
} else {
   set _CHIPNAME cs32f1x
}

```
 * the idcode:
```
#jtag scan chain
if { [info exists CPUTAPID] } {
   set _CPUTAPID $CPUTAPID
} else {
   if { [using_jtag] } {
      # See STM Document RM0008 Section 26.6.3
      set _CPUTAPID 0x3ba00477
   } {
      # this is the SW-DP tap id not the jtag tap id
      set _CPUTAPID 0x2ba01477
   }
}
```

and could try running openocd again:

```
D:\code\OpenOCD\bin>openocd -f interface/stlink-v2-1.cfg -f target/cs32f1x.cfg
GNU MCU Eclipse 64-bit Open On-Chip Debugger 0.10.0+dev-00352-gaa6c7e9b (2018-10-20-06:24)
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
WARNING: interface/stlink-v2-1.cfg is deprecated, please switch to interface/stlink.cfg
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
none separate
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : clock speed 950 kHz
Info : STLINK v2 JTAG v32 API v2 SWIM v7 VID 0x0483 PID 0x3748
Info : using stlink api v2
Info : Target voltage: 3.205816
Info : cs32f1x.cpu: hardware has 6 breakpoints, 4 watchpoints
Info : Listening on port 3333 for gdb connections
_
```

It seems better.

(Thanks to tsman on eevblog forum)

## New project from template

I skept the installation of cargo-generate (because of msvc), so I could not use it to generate the Rust project from the template. I also did not want to create them by cloning the git repository (because I already hade an existing git repo with this readme.md file, so I just download [https://github.com/rust-embedded/cortex-m-quickstart/archive/master.zip] and unziped it in a blue_pill_blinky` subdirectory.and changed the project name to `blue_pill_blinky` in the blue_pill_blinky\Cargo.toml` file (twice)

## memory.x

Since the template is meant for stm32f4 with a differente quantity of flash than mine, I edited to `memory.x` file (which, I believe, is used to generate the linker scripts) with values I found in the stm32f103 datasheet. Hoping that the "C8" at the end of the marking of my CKS mcu means the same thing as the "C8" at the end of a genuin stm32f103, I guess this chip has 64kb ok flash (see chapter 7 of the datasheet) and it should have 20Kb of ram (first page of the datasheet). The memory map (chapter 4 of the same datasheet) tells me that flash memory starts at 0x0800.0000 and static ram starts at 0x2000-0000, giving the following content for the `memory.x` file:
```
/* Linker script for the STM32F103C8T6 */
MEMORY
{
  FLASH : ORIGIN = 0x08000000, LENGTH = 64K
  RAM : ORIGIN = 0x20000000, LENGTH = 20K
}
```
(I removed the comments from the template)

## Compiling for the proper

We want to compile for our microcontoller. A microcontoler is an microprocessor packages with things like RAM, Flash memory, digital to analog converters, timers... Rust needs to know for which mcu we want to compile. The stm32f103 has a Cortex-M3 core (which is a proprietary but standard core found on many mcu form different manufacturers). Its architecture is called "ARMv7-M", this information is in the datasheet but I got it from [wikipedia](https://en.wikipedia.org/wiki/ARM_Cortex-M#Cortex-M3). So in `.cargo/config`, for the Blue Pill it will be:
```
[build]
target = "thumbv7m-none-eabi"    # Cortex-M3

```
`thumb` here relate to the instruction-set we want to use. Since Cortex-M only support the newer Thumb instructon (which is a 16 bits instructions set, as opposed to the older 32 bits ARM set,  it's faster and take less space, see [wikipedia](https://en.wikipedia.org/wiki/ARM_architecture#Thumb) again). 

## Deleting the target directory

I renamed the project, hence the project's directory name. This caused Cargo to be unable to compile (not finding the linker file). The solution was to delete the target directory and build again.

## Changing the dependency and main.rs

The compilation never ended, so I replaced the content of my `main.rs` file with the content of the hello-world found in `template/` directory (it came with the template).
I'm not sur this step is needed bu I did it.

## Switching the linker

The compilation never ended, stucked at step 32/33. Fortunately some comment in the `.cargo/config` file cought my attention:
```
  # LLD (shipped with the Rust toolchain) is used as the default linker
   "-C", "link-arg=-Tlink.x",

  # if you run into problems with LLD switch to the GNU linker by commenting out this line
  # "-C", "linker=arm-none-eabi-ld",

```
I commented the line for the LLD linker and uncommented the one for the GNU linker and could complete the build.

## openocd.cfg

I edited the `openocd.cfg` file that came with the template to use the openocd configuration I made for my weird stm32 clone:
```
source [find target/cs32f1x.cfg]
```
Once this is done, I can run openocd from the same directory the `openocd.cfg` file is in and no longer need to pass the configuration for the CKS clone (nor for the ST-Link V2-1 which was the default configuration in the openocd.cfg file, I didn't need to change that but mays you do):
```
D:\code\rust\blink_bluepill_rust\blue_pill_blinky>d:\code\OpenOCD\bin\openocd.exe
GNU MCU Eclipse 64-bit Open On-Chip Debugger 0.10.0+dev-00352-gaa6c7e9b (2018-10-20-06:24)
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
WARNING: interface/stlink-v2-1.cfg is deprecated, please switch to interface/stlink.cfg
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
none separate
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : clock speed 950 kHz
Info : STLINK v2 JTAG v32 API v2 SWIM v7 VID 0x0483 PID 0x3748
Info : using stlink api v2
Info : Target voltage: 3.204230
Info : cs32f1x.cpu: hardware has 6 breakpoints, 4 watchpoints
Info : Listening on port 3333 for gdb connections
```

## Starting gdb

Start gdb by replace `<gbd>` in the command given in chapter 2.2. with the name of the executable of the gdb you downloaded from ST website. Also since I juste ran cargo build, cargo did not copy the source files in `examples/` then I used a different directory from the one given the Embedded Rust Book:
```
arm-none-eabi-gdb -d target\thumbv7m-none-eabi\debug\
```
but that did no seem to work. Anyway I could use the `file` command to tell where my firmware is, gdb to openocd running in another shell, and upload the firmware:

```
(gdb) file target/thumbv7m-none-eabi/debug/b
blue_pill_blinky    blue_pill_blinky.d  build/
(gdb) file target/thumbv7m-none-eabi/debug/blue_pill_blinky
A program is being debugged already.
Are you sure you want to change the file? (y or n) y
Reading symbols from target/thumbv7m-none-eabi/debug/blue_pill_blinky...
(No debugging symbols found in target/thumbv7m-none-eabi/debug/blue_pill_blinky)
(gdb) load
Start address 0x8000, load size 0
Transfer rate: 0 bits in <1 sec.
(gdb)
```
I guess I uploaded something because the Blue Pill stopped blinking the LED that was controller by the original firmware.

## Trying to execute 

I tryed following the chapter 2.2 form there, but "next" was not of much help when gdb could not find the debugging symbol in my binary. So I tryed running the code (`continue` send to openocd from gdb) but nothing appeared in the openocd console. I was expecting an "Hello, world!".

I quit openocd and gdb, and use `STM32 ST-LINK Utility.exe` from ST. I clicked "connect the target" and look if the flash of the  Blue Pill seemed to contain the "Hello, world!" string: it did not. I reset the Blue Pill and it start blinking. It seems I did not flash the firmware.

## This time it works!

After chatting on IRC, I tryed to use the GCC toolchain instead of just the GNU linker (see comments in `.cargo/config`) and it compiled and I could upload and exectue the firmware.

This is the value that works for me for rustflags in `.cargo/config`file:
```
rustflags = [
  "-C", "linker=arm-none-eabi-gcc",
  "-C", "link-arg=-Wl,-Tlink.x",
  "-C", "link-arg=-nostartfiles",
]
```
I now can build:
```
D:\code\rust\blink_bluepill_rust\blue_pill_blinky>cargo build
[...]
   Compiling cortex-m-rt-macros v0.1.5
    Finished dev [unoptimized + debuginfo] target(s) in 40.82s
```
Now I can continue try flashing the firmware again and debugging it. I understood that I was not using the  `openocd.dbg` file provided by the template, so here is what I do now:

1. Start OpenOCD
 ```
 D:\code\rust\blink_bluepill_rust\blue_pill_blinky>d:\code\OpenOCD\bin\openocd.exe
GNU MCU Eclipse 64-bit Open On-Chip Debugger 0.10.0+dev-00352-gaa6c7e9b (2018-10-20-06:24)
Licensed under GNU GPL v2
For bug reports, read
        http://openocd.org/doc/doxygen/bugs.html
WARNING: interface/stlink-v2-1.cfg is deprecated, please switch to interface/stlink.cfg
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
adapter speed: 1000 kHz
adapter_nsrst_delay: 100
none separate
Info : Listening on port 6666 for tcl connections
Info : Listening on port 4444 for telnet connections
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : Unable to match requested speed 1000 kHz, using 950 kHz
Info : clock speed 950 kHz
Info : STLINK v2 JTAG v32 API v2 SWIM v7 VID 0x0483 PID 0x3748
Info : using stlink api v2
Info : Target voltage: 3.203691
Info : cs32f1x.cpu: hardware has 6 breakpoints, 4 watchpoints
Info : Listening on port 3333 for gdb connections
```
(I start from the directory where `openocd.cfg` file is, so I don't need to provide the `-f interface/stlink-v2-1.cfg -f target/cs32f1x.cfg`. And remember you might or might not need to make and use the `cs32f1x.cfg` file instead of `target/stm32f1x.cfg`)

2. Start gdb
 ```
 D:\code\rust\blink_bluepill_rust\blue_pill_blinky>arm-none-eabi-gdb -x openocd.gdb target\thumbv7m-none-eabi\debug\blue_pill_blinky
d:\Program Files (x86)\GNU Tools ARM Embedded\8 2018-q4-major\bin\arm-none-eabi-gdb.exe: warning: Couldn't determine a path for the index cache directory.
GNU gdb (GNU Tools for Arm Embedded Processors 8-2018-q4-major) 8.2.50.20181213-git
Copyright (C) 2018 Free Software Foundation, Inc.
[...]
Type "apropos word" to search for commands related to "word"...
Reading symbols from target\thumbv7m-none-eabi\debug\blue_pill_blinky...
core::sync::atomic::compiler_fence (order=32) at libcore/sync/atomic.rs:2351
2351    libcore/sync/atomic.rs: No such file or directory.
Breakpoint 1 at 0x8000f68: file C:\Users\Fabien\.cargo\registry\src\github.com-1ecc6299db9ec823\cortex-m-rt-0.6.7\src\lib.rs, line 550.
Function "UserHardFault" not defined.
Make breakpoint pending on future shared library load? (y or [n]) [answered N; input not from terminal]
Breakpoint 2 at 0x80015aa: file C:\Users\Fabien\.cargo\registry\src\github.com-1ecc6299db9ec823\panic-halt-0.2.0\src\lib.rs, line 32.
Breakpoint 3 at 0x8000402: file src\main.rs, line 13.
semihosting is enabled
Loading section .vector_table, size 0x400 lma 0x8000000
Loading section .text, size 0x1220 lma 0x8000400
Loading section .rodata, size 0x2ac lma 0x8001620
Start address 0x8000f26, load size 6348
Transfer rate: 17 KB/sec, 2116 bytes/write.
Note: automatically using hardware breakpoints for read-only addresses.
halted: PC: 0x08000f7c
DefaultPreInit ()
    at C:\Users\Fabien\.cargo\registry\src\github.com-1ecc6299db9ec823\cortex-m-rt-0.6.7\src\lib.rs:559
559     pub unsafe extern "C" fn DefaultPreInit() {}
(gdb) _
```
I now add the `-x openocd.gdb` parameter which is a script that does some things for us (like connecting gdb to openocd). Since the script is ran before we can use the `file` command to tell gdb where the elf file for the firmware is, we add the path to this as the last argument to gdb.
When the script is ran, you will see some information displayed in the other shell (the one with openocd running). The `semihosting is enabled` tells you that semihosting is activated. As the Rust Embedded Book explains, this allows us to basically use the debugger as stdout, hence display messages in OpenOCD.

3. step through
after using the `next` command in dgb, I finally got the expected message in OpenOCD:
```
[...]
Info : halted: PC: 0x08000626
Hello, world!
Info : halted: PC: 0x08000412
[...]
```

# Chapter 3 of The Embedded Rust Book (First led blinking)

Up to now, I have not done much thing wich is specific to the stm32f103c8 (clone) I use:
 * I installed ARM toolchain for Cortex-M (and Cortex-R) but this covers all the mcus in Cortex-M family (ARM design the core, and license the design to different manufacturers who produce them with differents options and package them with different peripherals)
 * I configured the `thumbv7m` target in `.cargo/config` (which covers all the Cortex-M3)
 * I changed the `idcode` in OpenOCD so I can tell it which `idcode` to expect from my clone
 * I set the proper size and base address for the flash and sram in the `memory.x` file.

 From there, it would be possible to directly read and write to the address of the special function registers to use all my microcontroller's peripherals. All I would need is some kind of .h file to give me the addresses, or I could directly read the datasheet and define them in my code. But if I do things this way, I would need to modify the code for every  