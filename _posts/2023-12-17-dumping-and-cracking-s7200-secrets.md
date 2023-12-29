---
layout: post
title: Dumping and cracking Siemens S7-200 PLC secrets
description: Dumping and cracking Siemens S7-200 PLC secrets
summary: 
tags: OT PLC s7200 iot hardwarehacking
minute: 15
---

# Introduction

Just like any computer system, PLCs *(Programmable Logic Controller)* and  __*other operational technology*__ components have **protection mechanisms**. These security features usually involve restricting access to functions through **passwords** and **protection levels**, in order to inhibit changes to software and other parameters of these devices. Even older PLCs, such as the **Siemens S7-200** line, which have limited computing resources by today's standards, already had password protection mechanisms and access control by protection level.

The architecture adopted in the design of Siemens S7-200 PLCs, like that of many hardware devices, has made this line susceptible to the recovery of sensitive information stored in memory. This can occur by extracting data from the EEPROM memory present in the S7-200 hardware. Although these PLCs are considered obsolete today, it is quite common to find industrial machines and processes still using these devices.

When I searched for more details about this information, I came across a lack of detailed data. I found only a few requests for help from PLC programmers who had lost passwords, preventing them from accessing and programming the PLC. In addition, I identified Windows utilities that perform _password cracking_ based on binary readings extracted from EEPROM memory. This case led me to begin more detailed research in order to better understand the architecture and how PLCs in this range store and define the configured **password** and **protection level** in memory.

**Note**: In order to carry out the steps that will be mentioned from here on, it was necessary to have the device physically, as the process that will be described required interactions directly with the PLC hardware.

## PLC Siemens S7-200 (CPU 226CN)

In short, a PLC *(Programmable Logic Controller)* is a device used in industrial automation to control processes, machines and systems. It is designed to perform specific functions, such as controlling machines, driving motors, monitoring sensors, and other tasks related to industrial automation. These devices have computational resources that allow them to carry out the most diverse operations and to manage and communicate via various means of communication *(e.g. Serial, Ethernet, Wireless)*. The PLC is programmable and can be adapted to different applications, making it a versatile choice in various engineering sectors. In the case of the **Siemens S7-200 PLC** range, the software used for code development and peripheral configurations is **STEP7 MicroWin Siemens official software**.

The PLC used in this research was the [S7-226-CN CPU (6ES7-216-2AD23-0XB8)](https://mall.industry.siemens.com/mall/en/cn/Catalog/Product/6ES7216-2AD23-0XB8), which is a specific model of PLC produced [by Siemens](https://pt.wikipedia.org/wiki/Siemens_AG), the company with the largest share of industrial automation and operational technology devices. 

The computing and hardware resources available in this CPU model are, for example: 

- 16x Digital Inputs DC / 16x Digital Ouputs DC;
- Work memory: integrated (for program) 24 kbyte; 16 KB with active run-time edit / integrated (for data) 10 kbyte;
- S7 Timers and Counters;
- Integrated RS-485 communication interface;
- Serial communication standards;
- MPI/PPI communication interface;
- And some others...

The image below shows the device used for this research (before it was opened):

![img1-s7200plc](https://github.com/hev0x/hev0x.github.io/assets/6265911/c7b28a42-ced5-4eea-a799-2893d0b14465)

### S7-200 Protections Mechanisms (Password and Protection Level)

The purpose of this topic is not to delve into the fundamentals of the architecture and code organization of S7-200 PLCs, but it is necessary to understand the organization of privilege levels and password configuration.

As already mentioned, these devices are programmable and use [Siemens STEP7 MicroWin](https://support.industry.siemens.com/cs/document/58523240/step7-microwin-v4-0-sp8-and-sp9?dti=0&lc=en-BR) software to develop the code, from which the __*developer inserts logic, instructions, routines, native code and configurations for the peripherals available on the hardware, such as I/Os, communications interfaces and security configurations*__, which is the subject of this research.

The following image shows the **privilege level configuration and password** verification screen in **STEP7 MicroWin**:

![microwin-password](https://github.com/hev0x/hev0x.github.io/assets/6265911/ccd7bfb6-1cc1-4b75-b6d2-2c7f5b08bf83)

As you can see in the image above, there are __*4 privilege levels*__ that can be configured for access control in the PLC, they are organized as follows:

>* **Full (Level 1)**: *Full Privileges*. All PLC functions are available without restriction.

>* **Partial (Level 2)**: *Read Privileges*. The user is allowed to read and write PLC data, and to upload the program without restriction. A password is required to download a program, force memory locations, or program the memory cartridge.

>* **Minimum (Level 3)**: *Minimum privileges*. The user is allowed to read and write PLC data without restriction. A password is required to upload or download a user program, force memory locations, or program the memory cartridge.

>* **Disallow Upload (Level 4)**: *Disallow Upload*. This level of password protection will prevent the program from ever being uploaded, **even when a correct password is entered**. This option will also disallow execution status, edit in run mode and project compare. Other PLC fuctions are protected in the same manner as a level 3 password.

It was noted that with the exception of privilege *Level 1* (Full), all other levels **require password verification** for certain PLC functions. 

From a security perspective, a breach of access control at any of these levels would have a significant impact on the device's application, since when accessed with partial or full privileges, a malicious actor would be able to:

- Access or modify the program of the PLC without any restriction or limitation;
- Execute native code arbitrarily;
- Exfiltrate sensitive information;
- Privilege escalation;
- Denial of service.

## Hardware identification

After understanding how the privilege levels and password configuration of the S7-200 PLCs are defined, we began to recognize the hardware. As already mentioned, our goal is to __*identify where the privilege level and password information is stored*__ in the hardware memory.

The following image shows the main board of the [S7-226-CN CPU](https://mall.industry.siemens.com/mall/en/cn/Catalog/Product/6ES7216-2AD23-0XB8):

![img3-s7200hardware(2)](https://github.com/hev0x/hev0x.github.io/assets/6265911/4917dde1-d906-4b50-b59b-6a5e8bd4eddf)

It is common for hardware devices to use external memories to store firmware or dynamic information. This type of architecture generally uses *EEPROM* or *NOR Flash* memories, which communicate with a main microcontroller via serial protocols such as SPI _(Serial Peripheral Interface)_ or TWI/I²c _(Two-Wire Interface_).

Searching around, I found some information on an electronic maintenance forum for Siemens devices about replacing/maintaining the memories that store the PLC program in question. The following image shows a guide to the EEPROM memories adopted for each Siemens S7-200 PLC line:

![img4-s7eepromguidBORD](https://github.com/hev0x/hev0x.github.io/assets/6265911/e8cc26a7-67e3-46a8-84fb-0b5546deb7f1)

Knowing this information, the IC *(Integrated Circuit)* **ST4256BWP**, an EEPROM memory with a memory size of 256Kb (32K x 8) with a TWI/I²C *(Two-Wire Interface)* communication interface, was identified according to the guide described above. The image below shows the memory **physically located** on the device board:

![img5-ci4256bwp](https://github.com/hev0x/hev0x.github.io/assets/6265911/c73d9629-e427-429f-ae0e-484646188c02)

By quickly analyzing this architecture, we can deduce that the privilege level and password information may in fact be stored in the identified memory.

### ST4256BWP EEPROM Memory

Before starting any kind of interaction with the memory, an analysis was carried out using the datasheet that references the model identified in the hardware. The memory [``ST4256BWP``](https://www.alldatasheet.com/view_datasheet.jsp?Searchword=4256BWP) is equivalent to the memory model ``M24256-BW``, so the information used was based on the datasheet of the second reference cited. 

Some of the characteristics of this memory are:

> **FEATURES SUMMARY**
>- Compatible with I2 C Extended Addressing   
>- Two-Wire I2 C Serial Interface Supports 400kHz Protocol
>- Memory Organization: 32K x 8 bits (256Kbit)
>- Single Supply Voltage: 2.5 to 5.5V 
>- Hardware Write Control
>- BYTE and PAGE WRITE (up to 64 Bytes)
>- RANDOM and SEQUENTIAL READ Modes

In addition to the resources available in the memory, it was necessary to identify the signals for each pin of the integrated circuit. The image and table below show the memory pinout and the description of each pin's function:

![img6-pinout1bord](https://github.com/hev0x/hev0x.github.io/assets/6265911/da7c96ed-5fb5-46a8-9ad7-5ea84274c507)

The description of the pins can be found in the table below:

* **Signal Names Table**

-------
```table
    Signal    | Description
------------- | -------------
E0, E1, E2    | Chip Enable
SDA           | Serial Data
SCL           | Serial Clock
WC            | Write Control
Vcc           | Supply Voltage
Vss           | Ground  
```
-------

In turn, the memory in question needs some operating instructions in order to read and write to its sectors. More details on each stage of the operation can be found in the datasheet. This information is extremely necessary for the development of possible firmware to communicate with the memory. The image below shows the table containing the EEPROM memory operating modes:

![img7-deviceoperationmembord](https://github.com/hev0x/hev0x.github.io/assets/6265911/337dc905-89a3-4764-ac0e-6410712e2f93)

Once we have identified the main electrical and organizational characteristics of the memory, we can highlight the use of TWI/I²C *(Two-Wire Interface)* communication to communicate with the memory.

#### Two-Wire Interface (TWI/I²C) communication

In short, the TWI/I²C *(Two-Wire Interface)* protocol is a two-wire serial communication standard **(SDA and SCL)** used to connect electronic devices over a short distance. Operating in *master-slav*e mode, it facilitates efficient data transmission between components. In EEPROM memory applications, TWI/I²C allows the master to communicate with slave devices to read or write data, using unique addresses to identify each device on the bus, making it a common and effective choice for non-volatile information storage.

The following image shows an example of a TWI/I²C communication frame format:

![img8-i2cframebord](https://github.com/hev0x/hev0x.github.io/assets/6265911/3e38b5b4-e8c2-45c1-82c2-837006bdd79c)

At this point our objective is not to delve into the TWI/I²C protocol, but to summarize it in order to put the type of communication present in the component in question into context.

## Memory Dumping

Based on the information gathered about the software and hardware architecture applied to the **S7-226-CN** PLC, it was possible to have inputs to start the process of extracting data from the __*ST4256BWP EEPROM memory*__. 

The first step in this process was to **desolder** the board's memory, because for some reason, when trying to communicate with it directly on the board using a [Soic 8 clip](https://www.google.com/search?client=firefox-b-d&q=Soic+8+clip), the information was read incorrectly. The image below shows the EEPROM memory unsoldered from the board and inserted into a Soic8 adapter:

![img9-eepromproto](https://github.com/hev0x/hev0x.github.io/assets/6265911/88bf6a04-3945-48f5-8652-083b1c21676a)

Once the memory chip had been removed from the board and inserted into the adapter, it was possible to interconnect the memory interface to the communication with a microcontroller or serial programmer.

### CH341a / STM32xxx

To _read and write_ the information **stored** in the memory, it is necessary to have a serial programmer that supports the connection of EEPROM memory chips with TWI/I²C. In my case, the _serial programmer_ I used was the [CH341a](https://www.onetransistor.eu/2017/08/ch341a-mini-programmer-schematic.html) model:

![img10ch341a](https://github.com/hev0x/hev0x.github.io/assets/6265911/811daf36-138e-4205-a8ac-86030ae5188c)

__*In order to guarantee compatibility, integrity and that no damage would occur to the memory when carrying out any type of operation via the serial programmer*__, it was necessary to develop firmware for the STM32 microcontroller, due to the fact that the memory in question *(ST4256BWP)* was not listed in the **compatible ICs of the CH341a** serial programmer.

The microcontroller used was the [STM32F103C8T6](https://www.st.com/en/microcontrollers-microprocessors/stm32f103c8.html#st-also-like), found on the [BluePill board](https://stm32-base.org/boards/STM32F103C8T6-Blue-Pill.html). This microcontroller has a *CPU ARM Cortex M3, 64 Kbytes of Flash memory, 72 MHz of max Clock, DMA, USB, CAN* and other hardware periphericals. 

The following image shows the microcontroller connected to the ST4256BWP memory on the protoboard:

![img11-mcueeeprom](https://github.com/hev0x/hev0x.github.io/assets/6265911/012e972e-2a97-4272-baac-022949d3b780)

Having connected the memory to the microcontroller's TWI/I²C bus, I²C communication parameters was configured in the **microcontroller's hardware**. The image below shows the parameters configured for communication in [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html):

![stm32hardconfig](https://github.com/hev0x/hev0x.github.io/assets/6265911/3964eb7c-1af2-4ea6-9be1-2c5087d43af3)

The firmware developed for memory checking and reading operations was based on the datasheets of the following memory models:

- [M24256-BW (ST)](https://www.st.com/en/memories/m24256-bw.html)
- [AT24C256A (Microchip/ATMEL)](https://ww1.microchip.com/downloads/en/devicedoc/doc0670.pdf)
 
The models mentioned above have a similar memory organization and operation to the ST4256BWP memory, for example:

```text
Memory array:
– 256 Kbit (32 Kbyte) of EEPROM
– Page size: 64 byte
``` 

The firmware developed consists of __*reading all the memory pages*__. To do this, the ``eeprom_twi`` library was developed, using the STM32 HAL Driver *(Hardware Abstraction Layer)*  as a base, which provides **API** for interacting with hardware peripherals, so we can use the native ``HAL_I²C_Mem_Read()`` function to read the EEPROM memory. 

The ``eeprom_twi`` library provides functions for partial or total reading and writing of memory sectors, for example:

```C
#include "eeprom_twi.h"

void writeDataEEPROM(uint16_t targetPage, uint16_t offset, uint8_t *data, uint16_t dataSize);
void readDataEEPROM(uint16_t targetPage, uint16_t offset, uint8_t *data, uint16_t dataSize);
void eraseEEPROMPage(uint16_t targetPage);

void EEPROM_dumpMemory();
``````

For debugging and reading information, USB CDC _(Communication Device Class)_ was used, which allows a Virtual COM Port port to be opened trhough USB. It is important to note that modified files from the USB *stack* were used, namely **usb_cdc_if.h** and **usb_cdc_if.c**. The code in these files was developed by [Nefastor](https://github.com/Nefastor-Online/STM32_VCP) and it provides a familiar __*"init / send / receive" API*__ as well as circular buffers.

The GIF below shows the firmware running on the STM32 microcontroller, reading the memory via the I²C bus and displaying the information on the serial port *(USB-CDC as VirtualCOMPort)*:

![img13-stm32term](https://github.com/hev0x/hev0x.github.io/assets/6265911/847b0070-6d04-4a32-951e-b6d252bcfc23)

> The firmware code is available [**here**](https://github.com/hev0x/S7200-toolkit-decrypt/tree/main/firmware)  and can be built using the [STM32CubeIDE](https://www.st.com/en/development-tools/stm32cubeide.html). 

Once the compatibility of the memory model in question had been confirmed, the process of interacting with the memory via the [CH341a serial programmer](https://www.onetransistor.eu/2017/08/ch341a-mini-programmer-schematic.html) began. The image below shows the memory connected to the CH341a via the Soic8 adapter:

![img14-ch341aconn](https://github.com/hev0x/hev0x.github.io/assets/6265911/9d62c8db-41f1-4b83-92df-46ef01258461)

To communicate with the CH341a, the [IMSProg software](https://github.com/bigbigmdm/IMSProg) was used, which allows interaction with the serial programmer.

The IMSProg is an __*I²C and SPI EEPROM/Flash chip programmer for CH341a*__ devices. More information about IMSProg can be found in its [official repository](https://github.com/bigbigmdm/IMSProg).

The image below shows the execution of the ST4256BWP memory read operation using IMSProg:

![img15-progread](https://github.com/hev0x/hev0x.github.io/assets/6265911/cc66e299-f411-4c25-8712-53c944bd5441)

With access to all the content stored in memory, this content was saved in a **binary file (.bin)**.

## Binary image analysis

The process of analyzing the contents of the binary began with the objective of identifying the **privilege level**, **password** or other possible sensitive information in plain text.

To start this process, the [hexdump utility](https://man7.org/linux/man-pages/man1/hexdump.1.html) was run, specifying the .bin file containing the *dumping of the EEPROM memory*. The following GIF and image show the execution and result of the tool:

![img17-hexdumpbin](https://github.com/hev0x/hev0x.github.io/assets/6265911/5e2f972f-974a-47dd-bad7-c4777f13f0f6)

The results show that some data has been **encrypted** in some way, making it impossible to read the information clearly.

### Password Encryption

At this point, I already knew that the bytes referring to the **privilege level** and **password** information were encrypted, and after a lot of searching I managed to identify a **.txt** file containing a simple table with the memory addresses (offset) that corresponded to the **protection level**. **encrypted password** and predefined characters in a password encoding table.

The table below shows the memory addresses corresponding to the **privilege level** and **password** information:

-------
```table
    Address    | Description
-------------  | -------------
1E75           | Protection Level (01-04)
1E76~1E7D      | Encrypted Password
--------------------------------------
1E76           | 1st character
1E77           | 2st character
1E76 XOR 1E78  | 3st character
1E77 XOR 1E79  | 4st character
1E78 XOR 1E7A  | 5st character
1E79 XOR 1E7B  | 6st character
1E7A XOR 1E7C  | 7st character
1E7B XOR 1E7D  | 8st character
```
-------


Luckily for me, and without giving much of a reaction, I realized that the cipher applied to the password was an *XOR operation*, alternating bytes in fixed address offset. 
 
![80896736380a6349a49ee228a74dc632](https://github.com/hev0x/hev0x.github.io/assets/6265911/518257b9-1660-42d4-88bb-d8812cfe17bd)


## Password Cracking

The fact that the addresses storing the security parameters are not random makes the attack even more exploitable, since the addresses are fixed and known. 

To perform the __*password cracking process*__, it was necessary to create a python script that correlates the result of _XOR operations_ with predefined characters in a password encoding table, this table is defined as ``char_map = {...}`` in the code.

Based on the **addresses** and **password coding table** identified in the **.txt** content found, the **password in clear text** can be obtained using the following code:

```python
def decrypt_password(binary_data):
    try:
        protectionLevel = binary_data[0x1E75]
        rawBytes = binary_data[0x1E76:0x1E7E]

        rawXOR1 = binary_data[0x1E76] ^ binary_data[0x1E78]
        rawXOR2 = binary_data[0x1E77] ^ binary_data[0x1E79]
        rawXOR3 = binary_data[0x1E78] ^ binary_data[0x1E7A]
        rawXOR4 = binary_data[0x1E79] ^ binary_data[0x1E7B]
        rawXOR5 = binary_data[0x1E7A] ^ binary_data[0x1E7C]
        rawXOR6 = binary_data[0x1E7B] ^ binary_data[0x1E7D]

        XOR1 = char_map[f'{rawXOR1:02X}']
        XOR2 = char_map[f'{rawXOR2:02X}']
        XOR3 = char_map[f'{rawXOR3:02X}']
        XOR4 = char_map[f'{rawXOR4:02X}']
        XOR5 = char_map[f'{rawXOR5:02X}']
        XOR6 = char_map[f'{rawXOR6:02X}']

        decryptPasswd = [char_map[f'{rawBytes[0]:02X}'], char_map[f'{rawBytes[1]:02X}'], XOR1,XOR2, XOR3, XOR4, XOR5, XOR6]

        return protectionLevel, rawBytes, decryptPasswd
    
    except Exception as e:
        return None, None, str(e)
```

The image below shows the script being executed with the binary extracted from memory in the previous steps as input:

![img18-scriptcracking](https://github.com/hev0x/hev0x.github.io/assets/6265911/535a5bf2-8a2f-4a36-b3cf-2ac41f96f15b)

> The script for cracking the password is available from the repository: [S7-ToolkitDecrypt](https://github.com/hev0x/S7200-toolkit-decrypt)

It is also possible to __*change the protection level value*__ in the binary, in the case of **Protection Level 04**, **even with the correct password entered access to the PLC would be limited**, once this value is changed in the binary and rewritten in the PLC's EEPROM memory, the malicious actor would have unrestricted access to the device. 

The following image shows the memory address corresponding to the protection level changed to the value 01 (Full Privileges):

![img19-levelcrackingok](https://github.com/hev0x/hev0x.github.io/assets/6265911/e1562474-bbad-4f54-bafe-508df0768bf9)

Once the protection level value had been changed to **01**, the image rewritten in the EEPROM memory and the component resoldered on the hardware board, it was possible to subvert the restrictions on the device.

## Considerations

After carrying out all the steps mentioned above, it was possible to _obtain the PLC's password and change the protection level successfully_, so it was possible to confirm that the security mechanisms present in this device were implemented inadequately or due to the fact that it has limited hardware resources, since more modern microcontrollers and microprocessors already have integrated hardware modules that allow the use of secure encryption algorithms.

In any case, it was a fun process and one that led me to start other searches on more up-to-date PLC platforms, like S7-1200 and S7-1500 Siemens PLC.

## References

- [https://conference.hitb.org/hitbsecconf2021ams/materials/D2%20COMMSEC%20-%20Breaking%20Siemens%20SIMATIC%20S7%20PLC%20Protection%20Mechanism%20-%20Gao%20Jian.pdf](https://conference.hitb.org/hitbsecconf2021ams/materials/D2%20COMMSEC%20-%20Breaking%20Siemens%20SIMATIC%20S7%20PLC%20Protection%20Mechanism%20-%20Gao%20Jian.pdf) ```Special thanks to the author of this research, from which I had a starting point to carry out the process described in this article.```
- [https://www.st.com/en/memories/m24256-bw.html?rt=ds&id=DS1766#documentation](https://www.st.com/en/memories/m24256-bw.html?rt=ds&id=DS1766#documentation)
- [https://github.com/bigbigmdm/IMSProg](https://github.com/bigbigmdm/IMSProg)
- [https://github.com/hev0x/S7200-toolkit-decrypt](https://github.com/hev0x/S7200-toolkit-decrypt)




