---
layout: post
title: Debugging 8-bit AVR® microcontrollers with avr-gdb and Atmel-ICE.
description: Pratical debugging 8-bit AVR® microcontrollers with avr-gdb and Atmel-ICE.
summary: 
tags: debugging atmelice jtag avr mcu avrgdb gdb
minute: 7
---

Not only during the firmware development process, in embedded systems security analysis the debugging process is also essential, so I have been dedicating myself to deeply understand the memory operation of *8-bit AVR® microcontrollers*, considering that these microcontrollers implement the [Havard architecture](https://en.wikipedia.org/wiki/Harvard_architecture), this task it's not so easy (but it can be quite fun too).

In this case, it was extremely necessary to find some pleasant way to do the debugging process, due to already having contact with memory debugging using GDB on Linux and also because the official IDE [Atmel Studio](https://www.microchip.com/en-us/tools-resources/develop/microchip-studio) has an integrated GDB that honestly, the interface has many limitations and is not optimized at all, which ends up consuming a lot of resources compared to the GDB used in any Linux distribution. 

There is a unofficial Linux *toolchain* for AVR microcontrollers that has several tools, among them the <code>avr-gdb</code> that as well as the classic GDB that we know, performs several operations to perform debugging. In the case of microcontrollers and microprocessors, debugging is usually performed through a physical interface provided by the hardware known as **JTAG**. 

For research with these devices I selected a microcontroller [AVR ATmega2560](https://www.microchip.com/en-us/product/atmega2560) and it provides the **JTAG** interface through its hardware. It is important to note that not all *8-bit AVR® microcontrollers* provide this interface. Another important point is that in this blogpost, my idea is not to delve into details of the microcontroller used, about commands used during debugging or even about details of the **JTAG** interface, but I will make a brief introduction about the main aspects of the **JTAG** interface.


## How **JTAG** interface works?

Basically, **JTAG** is a physical interface provided by the hardware that allows, to perform actions such as debugging, memory manipulation and the extraction of the firmware image from electronic devices. Currently, this interface is used and found in most processors and microcontrollers of various architectures such as *ARM*, *AVR*, *x86*, *MIPS* and some others....

With **JTAG** we can manipulate the firmware execution (***stop execution, inspect memory, set breakpoints, execute step-by-step, etc***). In this way, we can inspect the state of the processor and its registers, read/write to memory and trigger I/O devices.

Through a feature called Boundary Scan, the **JTAG** interface allows access to all pins of the chip. In this way, we can read and write individually on each pin, and consequently manipulate the peripherals connected to the processor/microcontroller (***GPIO, memory, flash, etc***).

In practice, with the JTAG interface we can:

+ Identify information about the hardware (processor, memory, etc).
+ Change the behavior of programs at runtime.
+ Dump RAM memory and get access to data.
+ Manipulate peripherals and registers, such as turning an I/O pin on or off.
+ Capturing data from hardware devices, such as information stored in an EEPROM.
+ Debugging xD

### JTAG Interface Signals

There are some variations of the standard, the **JTAG** interface is usually implemented by the chip manufacturer through 4 mandatory pins `(TDI, TDO, TCK, TMS)` and 1 optional pin `(TRST)`.

The interface connects to an on-chip *Test Access Port (TAP)* that implements a stateful protocol to access a set of test registers that present chip logic levels and device capabilities of various parts. JTAG uses the following signals to support the operation of boundary scan:

+ TDI (Test Data In): Data input.
+ TDO (Test Data Out): Data output.
+ TCK (Test Clock): Clock whose maximum frequency depends on the chip (10MHz to 100MHz).
+ TMS (Test Mode Select): State machine control pin.
+ TRST (Test Reset): Is an optional pin to reset the **JTAG** controller state machine.


> The **JTAG** interface pins connect internally to the chip through a module called *TAP (Test Access Port)*.

The *TAP* implements the basic communication protocol with the chip via JTAG, and several TAPs can be connected simultaneously in a [daisy chain architecture](https://en.wikipedia.org/wiki/Daisy_chain_(electrical_engineering)). The image below demonstrates the internal architecture of the *TAP*.

![20200220-jtag-pins](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/9b436a3c-818a-4541-b082-007be77ef484)


Up to this point it has been discussed in a basic way what the standard is about and how the **JTAG** interface works minimally... in any case, it is possible to go deeper into the subject in other sources.


## Atmel-ICE hardware 

To perform the communication between the microcontroller and the GDB, it is necessary a hardware that has and enables communication through **JTAG**, in the case of AVR microcontrollers, there are some hardware possibilities in the market that provides such a function. In my case, I use [Atmel-ICE](https://www.microchip.com/en-us/development-tool/atatmel-ice) for this purpose, it is a development tool for debugging and programming ARM® Cortex®-M based Atmel® SAM and Atmel AVR® microcontrollers with On-Chip Debug capability. This tool has other interfaces that use other protocols such as *SPI*, *SWD*, *PDI*...

![[Atmel-ICE Hardware Debugger](https://github.com/hev0x/hev0x.github.io/assets/6265911/6d782a8b-269e-4524-a30f-205cddd64958)


Basically, through USB we connect the Atmel-ICE to a computer and through different protocols, we can communicate with debugging tools like GDB and perform actions already mentioned in *ARM® Cortex®-M based Atmel® SAM and Atmel AVR® microcontrollers*.

## Connecting Atmel-ICE to a JTAG AVR target

First of all, it is necessary to electrically connect the **JTAG** communication interface between the Atmel-ICE hardware and the microcontroller. As I have seen in forums like [AVR Freaks](https://www.avrfreaks.net), it is always and has already been the subject of my doubt, how to interconnect the interface between the devices, so let's go.

To understand the connection scheme between the devices, it is necessary to identify where each pin is located and what signal is transmitted by these pins. As we are talking specifically about AVR devices, we can find this information from the ***datasheets*** of each of the components, located at the URLs below:

* [ATMega2560 Datasheet](https://ww1.microchip.com/downloads/aemDocuments/documents/OTH/ProductDocuments/DataSheets/ATmega640-1280-1281-2560-2561-Datasheet-DS40002211A.pdf)
* [Atmel-ICE Hardware Debugger](https://ww1.microchip.com/downloads/en/DeviceDoc/Atmel-ICE_UserGuide.pdf)

##### AVR® JTAG Header Pinout

As I have already mentioned, Atmel (or Microchip hahaha), manufacture some hardwares for debugging their microcontrollers/microprocessors, so they standardize the connector scheme for interconnecting their devices through a standard pin scheme. It is no different for the [Atmel-ICE Debugger](https://onlinedocs.microchip.com/pr/GUID-DDB0017E-84E3-4E77-AAE9-7AC4290E5E8B-en-US-4/index.html?GUID-27BF3A3E-B61E-485F-8816-EBB7F5642827), which has the following standard [AVR® JTAG pinout](https://onlinedocs.microchip.com/pr/GUID-DDB0017E-84E3-4E77-AAE9-7AC4290E5E8B-en-US-4/index.html?GUID-27BF3A3E-B61E-485F-8816-EBB7F5642827):

![GUID-E82E83BA-49F0-4C7D-8368-651F988A95BE-low-PhotoRoom png-PhotoRoom](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/c6d19198-3741-422b-9939-9842ef2f83db)


##### AVR® ATmega2560 pinout

In the case of the ATmega2560 microcontroller pins, the **JTAG** interface signals are present on the *Port F* pin set, which also adds ADC functions, but if the **JTAG** interface is enabled *(JTAGEN fuse bit)*, the pull-up resistors on pins *PF7(TDI)*, *PF5(TMS)*, and *PF4(TCK)* will be activated even if a reset occurs.

The pin layout on the microcontroller for connection to the JTAG interface is as follows:

```text
PF4 - TCK
PF5 - TMS
PF6 - TDO
PF7 - TDI
```

![Captura de tela 2023-08-11 234859](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/090bcfb3-1c57-4dc8-82ae-fc9960ca52ca)


## Softwares for debugging

Until this point I talked about the hardware and electrical connections, but now we can start the configuration process to communicate the **avr-gdb** with our **JTAG** interface connected to the ATmega2560 microcontroller. By default, the GDB does not offer this type of communication, so to communicate the **Atmel-ICE**, it was necessary to use a tool called [**Bloom**](https://bloom.oscillate.io/). Bloom is a debug interface for AVR-based embedded systems development on GNU/Linux, it creates a process that enables integration with GDB.

Bloom works in an architecture similar to the flow in the image below:

![bloom-architecture](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/410019d8-5cdf-4082-a213-2c185a747c9c)

### Setting up and installing Bloom

The tool documentation already has the necessary information to perform this process, but I will summarize what was necessary for configuration.

About the download, for each Linux distribution, the process can be identified on the official developer page: [https://bloom.oscillate.io/download](https://bloom.oscillate.io/download).

Before running Bloom, it is necessary to create a configuration file *yaml* format, in my case the file was as follows:

```yaml
environments:
  default:
    debugTool:
      name: "atmel-ice"

    target:
      name: "atmega2560"
      physicalInterface: "jtag"

    debugServer:
      name: "avr-gdb-rsp"
      ipAddress: "127.0.0.1"
      port: 1442

insight:
  enabled: true
```   


### Setting up and installing avr-gdb

In the case of avr-gdb, it is also present in some Linux distributions, but I will only indicate the installation using the native package manager of the distribution that I used:

```bash
sudo apt-get install -y gcc-avr avr-libc avrdude libtool texinfo elfutils \
                     libglu1-mesa-dev freeglut3-dev gdb-avr libelf-dev
``` 

## Starting the debugging process

To start the debugging process with avr-gdb, it is necessary to run **bloom** from the directory in which the configuration file in *yaml* format was created. 

Running **bloom** should result in output similar to that shown below:


![bloom](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/b31ab2ef-6572-48d3-8804-958c70b8218b)


Bloom is indicate when it's ready to accept incoming GDB connections :D

##### Starting GDB

From this point, a connection can be established via GDB. This can be handled by an IDE that supports it, or, directly via avr-gdb's CLI. A connection with Bloom's GDB server can be established via avr-gdb's command line interface (CLI), using the ```target remote``` *command*. 

Run avr-gdb and pass the path to the ELF executable of the AVR program that is to be debugged. From the GDB console, use the following command to establish a connection with Bloom’s GDB server:

```bash 
target remote localost:port
``` 

The GIF below demonstrates a series of commands performed in GDB when debugging the AVR ATmega2560 microcontroller:

![avr-gdb](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/32dd4178-19f2-4d7c-ad21-fbc12f56cf4d)

* List of performed commands

| Command             | Short form           | Description                            |
| :---                | :--                  | :---                                   |
| continue | c | Run the program |
| backtrack -full | bt -fu | Display backtrace including local variables |
| info registers | i r | Dump MCU registers |
| list main | l main | Show the source code of `loop()` |
| disassemble | disas | Disassemble the current program location |
| print $pc | p $pc | Print the value of PC (Program Counter) |
| print $sp | p $sp | Print the value of SP (Stack Pointer) |
| x/16b $sp | | Dump 16 memory bytes starting at $sp (stack pointer) |
| tui enable | tu e | Enable the TUI mode (also Ctrl X+A) |
| tui reg all | tu r a | Display registers window |
| tui disable | tu d | Disable the TUI mode (also Ctrl X+A) |




Now it is possible to debug AVR microcontrollers and control all program flow, without suffering in the official IDE. More information about the use and commands of the **avr-gdb** can be consulted in one of the links provided in the references below.

In future blogposts, I will go deeper into the memory manipulation of these microcontrollers and with that, we can also delve into commands and details in debugging. That's all folks.


## References

* [https://www.microchip.com/en-us/product/atmega2560](https://www.microchip.com/en-us/product/atmega2560)
* [https://www.microchip.com/en-us/development-tool/atatmel-ice](https://www.microchip.com/en-us/development-tool/atatmel-ice)
* [https://bloom.oscillate.io/](https://bloom.oscillate.io/)
* [https://www.avrfreaks.net/s/topic/a5C3l000000Ucn7EAC/t162905](https://www.avrfreaks.net/s/topic/a5C3l000000Ucn7EAC/t162905)
* [https://www.avrfreaks.net/s/topic/a5C3l000000BpqjEAC/t391261](https://www.avrfreaks.net/s/topic/a5C3l000000BpqjEAC/t391261)
