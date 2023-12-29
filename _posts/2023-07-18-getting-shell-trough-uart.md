---
layout: post
title: Getting shell on routers through UART
description: 
summary: 
tags: hardwarehacking iot
minute: 10
---

For some time, I’ve been thinking about why I haven’t written about what I’ve been researching, so I decided to start this blog showing how to access a router’s shell through a **UART communication** interface.

The motivation to write about this came from an attempt and desire to install the [**OpenWRT Firmware**](https://openwrt.org/) on a fiber optic modem/router from an unknown manufacturer that I found in some old stuff.

From a security perspective, there are several points to consider when analyzing a hardware or IoT (as it is called today) device. Even knowing that most manufacturers of home and general purpose routers use very similar architectures to create these products, the curiosity to interact and know what could be done through the interfaces available in the hardware was much greater and also very fun. 😁

Manufacturers of electronic devices usually provide communication interfaces in hardware so that some interactions can be performed directly with the peripherals contained in the hardware (e.g., EEPROM, Flash memories, and others) and even with the current operating system. One of these interfaces, which is usually present, allows serial communication through the UART protocol.

## But what is the **UART** protocol?
*UART* stands for "universal asynchronous receiver/transmitter" or universal asynchronous transmitter/receiver and defines a protocol, that is a set of rules for the exchange of serial data between two devices. *UART* is simple and uses only two wires between transmitter and receiver to transmit and receive in both directions. Both ends also have a ground. *UART* communication can be simplex (data is sent in one direction only), half-duplex (both sides transmit but only one at a time), or full-duplex (both sides can transmit simultaneously). *UART* data is transmitted in the form of frames. The format and content of these frames will be briefly described and explained.

Example of connected devices exchanging data via *UART*:

![1img](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/fa24192e-aab2-43ac-b94a-41cce88ce082)---

##### When *UART* is used?
*UART* is one of the oldest serial protocols. Currently, it is possible to find the *UART* protocol in all of the most widely used microcontrollers, as many peripherals and integrated circuits still use this protocol for data exchange.

Recently, the use of *UART* has declined somewhat: protocols such as *SPI* and *I²C* have replaced *UART* between chips and components. Instead of communicating through a serial port, most modern computers and peripherals now use technologies such as Ethernet and USB. However, the *UART* is still used for lower speed and flow applications due to its simplicity, low cost and ease of implementation.

---- 

##### *UART* protocol timing and synchronization
One of the great advantages of *UART* is that it is asynchronous – the transmitter and receiver do not share a common clock signal. Even though this significantly simplifies the protocol, it also imposes some requirements on both the sender and the receiver. Since they don't share a clock, both ends must transmit at the same time and at a preset rate in order to have the same bit timing. The most common baud rates used in *UART* today are 4800, 9600, 19.2K, 57.6K, and 115.2K. In addition to having the same baud rate, both sides of a *UART* connection they also have to use the same frame structure and parameters. The best way to understand this is to look at a *UART* frame.

---

#### *UART* frame format
As with most digital systems, a "high" voltage level is used to indicate a logical "1" and a "low" voltage level is used to indicate a logical "0". Since the *UART* protocol does not define specific voltages or voltage ranges for these levels, sometimes the high level is called "mark" while the low level is called "space". Note that in the idle state (where no data is being transmitted) the line is held high. This allows you to easily detect damage to a line or transmitter.

> ![2img](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/830f40c1-6cbb-4f17-b0a2-3d91ef82091f)
> *UART* protocol frames contain **start** and **stop** bits, **data** bits and an optional **parity** bit which will be explained below.
>

---
##### Start and stop bits

Because the *UART* protocol is asynchronous, the sender needs to signal that data bits are arriving. This is possible when using the **start bit**. The **start bit** is a transition from the inactive state to a low state, immediately followed by the user **data bits**.

---

##### Data bits
Data bits are user data or "useful" bits and come immediately after the start bit. There can be 5-9 bits of user data, although 7 or 8 bits are more common. These bits of data are usually transmitted least significant bit first.


- Example:
If we want to send the capital letter “S” in a 7-bit ASCII, the sequence of bits is `1 0 1 0 0 1 1`. First we reverse the order of the bits to put them in the least significant order, that is, `1 1 0 0 1 0 1`, before sending them. After the last bit of data is sent, the **stop bit** is used to end the frame and the line returns to the idle state.

> 7 bits ASCII 'S' (0x52) = 1 0 1 0 0 1 1

> LSB order = 1 1 0 0 1 0 1


---

##### Parity bit
A *UART* frame can also contain an optional parity bit that can be used for error detection. This bit is inserted between the end of the data bits and the final bit. The value of the parity bit depends on the type of parity being used (even or odd):

> In even parity, this bit is set so that the total number of 1s in the frame is even.

> In odd parity, this bit is set so that the total number of 1s in the frame is odd.


#### Summary about *UART*
+ *UART* stands for "Universal Asynchronous Transmitter/Receiver" and is a simple two-wire protocol for exchanging serial data;
+ The term "asynchronous" in this case means that there is no shared clock, so for the *UART* to work, the same bit or baud rate must be set on both sides of the connection;
+ The start and stop bits are used to indicate where the user data starts and ends, i.e. as "frame", to "frame" the data;
+ An optional parity bit can be used to detect individual bit errors;
+ *UART* is still a widely used serial data protocol, but has recently been replaced in some applications by technologies such as *SPI*, *I²C*, *USB* and *Ethernet*.

### Information gathering - finding *UART* taps

As with all vulnerability assessments in hardware devices or software, it is necessary to gather information to understand the device's architecture and thus be able to identify an entry point to perform a specific action, malicious or not.

To do this, I started by identifying all the electronic components of the hardware and all the pins marked on the board. The following image shows the board of the analyzed device:

![3img](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/eed85c4c-d4a8-40fe-b55a-a81609a60496)

In some cases, the pins related to the communication interfaces will not be labeled on the electronic board; it was not my case, as the image below demonstrates the identification of the ports necessary to start the communication through the *UART* are identified:

In some cases, the pins related to the communication interfaces will not be labeled on the electronic board; it was not my case, the image below demonstrates the identification of the ports necessary to start the communication through the **UART** are identified:

![4img](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/fb790b70-6e4d-450b-94a2-0f84d6a500df)

It is possible to identify that there is a label of two connections of serial communication interfaces, one of them uses the *I²C* protocol, it is not as if we would despise it, though it is possible to dump the *EEPROM*/Flash memory to get the device firmware. In the future, this will be interesting for other articles, for the moment I will limit myself to analyzing only the **UART** communication interface.

As I have already mentioned, it is possible that some devices do not have this label on the board, in this case you will need a logic analyzer or a multimeter to assist in such identification, I do not want to detail the multimeter and logic analyzer testing process, but it is necessary to keep in mind that you mainly need to identify the TX, RX and GND pins... for this you will need to use the **continuity test** and **continuous voltage measurement** functions of the multimeter.

### Starting communication

To initialize the communication between your computer and a serial device, it is necessary to use an FTDI controller to convert the serial signal to USB.

Serial to USB converters can be easily found in the market, in this case I used the following converter:

![5img](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/1fd5b387-6a40-403d-8b24-e8df9f7a71ee)


```
It is important to note that during this step I had already soldered a connector to the pins of the *UART* ports to facilitate the connection of the jumpers connected to my FTDI board.
```

![5 5img](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/1a844ea3-8fb8-472f-aa2e-84870bfb3e20)

#### Eletrical connections

Luckily in this case, the ports are clearly labeled on these device models, the ports are as follows:

```
1. VCC - 3.3 volts DC constant
2. GND - Ground
3. RX - Receive data
4. TX - Transfer data
```

Fine, so what if we need to initial communication between two *UART*s?

As the previous example we will need to do the following steps:

1. Connect the TX of the first *UART* with the RX of the second *UART*.
2. Connect the RX of the first *UART* with the TX of the second *UART*.
3. Connect the GND of the first *UART* with the GND of the second *UART*.  


| GND      | GND |
| TX      | RX       |
| RX   | TX        |


Note that we will not use the VCC pin, it will not be of any use at the moment.

To establish the electrical connection between the FTDI controller board and the device (router/modem), we can consider the following connection scheme:

![6img](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/80114db4-a8a2-49b0-a183-42ba9aa8a0cf)

After making the connection between the devices, the connection was as follows:

![7img](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/cab8bfd3-1bfb-4b34-900e-94576133f351)

---

![7img2](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/a213a316-1dde-4aeb-b9e5-06e0ffad5469)


#### Console setup

With all electrical connections completed, it is time to start communication from the terminal. There are several tool options to open serial communication:

> Linux/Mac
- minicom: https://linux.die.net/man/1/minicom
- screen: https://linux.die.net/man/1/screen

> Windows
- PuTTY: https://www.putty.org/


In my case I used **screen** because I use linux and this way it is possible to install it easily with its package manager.

If your package manager is ``apt'', you can use the following command:

```bash
$ apt install screen
```

In order to communicate you will need to identify where the FTDI controller is mapped within your operating system, in Linux USB based serial ports may use a name such as ttyUSB0. All of these devices are located in the ``/dev/tty*`` directory.

It is also important to note that some devices require non-standard configurations for the following *UART* protocol parameters:

```text
- Parity
- Stop bit
- Baud Rate 
``` 

The possible *baud rates* of the *UART* protocol are:

```text
[110 300 600 1200 2400 4800 9600 14400 19200 38400 57600 115200 128000 256000]
``` 

Normally devices use a communication speed of 115200(baud rate) and this is also the case with our device.

At this point we can already try to communicate with the device through the *UART*. We can establish a connection with the following command:

```bash
$ screen /dev/ttyUSB0 115200
```

When the command is executed, we can observe that an embedded system boot is initiated and it is already possible to obtain a number of information about the embedded system and various peripherals present in the *hardware*, such as the flash memory. The following picture shows this fact:

![8gif](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/e657fd47-7f0b-4c01-bff9-77f720451740)

Well, at that moment I was quite excited, because I was sure that there was a Linux embedded, and that way I could even start a *firmware* dump process if I couldn't access a shell.

That's when I realized that after booting and initialization, when pressing any key on the keyboard, it was possible to get an authentication message asking for 'login' and 'password':

![9img](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/ecb144ee-d0a6-4514-a63c-4041b5aaf918)

Well that was a pretty easy task, I googled for a reference on a label I found behind the lid of the device and easily came up with a valid username and password ;)

![10gif](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/5b869cac-bf73-46ee-99aa-9035fd4bb020)

After authenticating, it was possible to execute commands with the root user:

![11gif](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/9f24ca41-94b9-45a6-931f-a86afba274e4)

---

![11img](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/feaae915-a4e8-4769-903e-472d365d11e0)

I was also able to identify that after *bootloader* initialization, it is possible to arbitrarily interrupt the *boot* of the system and enter a command execution mode that contains several reading and writing functions, which allow from memory reading to even writing new *firmware* for the device. The result can be seen below:

![12gif](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/56c9e13a-cfaa-4e90-af75-4e79684fe60c)

---

![12img](https://github.com/h3v0x/h3v0x.github.io/assets/6265911/aefe0e07-ef3c-4f51-85cb-f67dc34b7543)


## Conclusion

Having achieved my goal, I was also able to confirm that it is indeed possible to access the modem/router's file system via a shell obtained with the *UART* protocol.

Well, then you can use your imagination and perform some actions against the device, keeping in mind that this shell is a bit limited in terms of available commands... still, it's enough to compromise the device.

### What could be done with the shell through the *UART*?
 
- Use command injection or other trickery to bypass the shell restrictions;

- Discover how login password is created and attempt to crack it;

- Dump Firmware over serial console;

- Interrupt bootloader and use tricks to get a root shell;

- Event if it's *read only we can get some gifts* like:
    - boot logs showing *hardware* specifications;
    - Encryption keys and passwords;
    - Crash logs from our exploitation attempts.

---
### Troubleshooting

In some cases, many problems may occur, it is important to pay attention to some points such as

- Make sure that the baud rate set is the same as the one configured on the device, otherwise you will receive a sequence of bytes that does not make sense;
- Make sure all electrical connections are good.

