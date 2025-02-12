---
title: "Analyzing reflash routines Part 4 (Jaguar AJ27 specific)"
classes: wide
categories:
  - Software and firmware
tags:
  - ECU
  - Jaguar
  - AJ27
  - Ghidra
---
The previous two posts analyzed the canbus commands implemented by  CPU1 TPU bootcode, and how to use them to load the 5k byte "MainBoot" program into CPU1 RAM. Once the MainBoot program is running, it provides similar canbus commands to upload and execute the "SubBoot" to CPU2. And then when both Boot programs are running, they enable the capabilities to read, erase, and reprogram the EEPROM contents for CPU1 and CPU2.

Rather than going into the fine details of the programs, I'll just provide a rough summary of each program, the available instruction sets, and how they can be used. These details are based on the implementations of MainBoot and SubBoot in F27SA074.b68 and F27SB074.b68, which are for software load LNG1410AG.

The MainBoot program for CPU1 is structured as follows:
* Monitor Canbus for incoming commands (summarized below) from SID = 0x020, execute any command received, and send responses with SID = 0x030
* Monitor serial port with CPU2 for outcomes of processes requested over the serial port, as part of received canbus commands
* Monitor on-going sub-phases of any EEPROM erase process, and progress as needed
* Monitor on-going sub-phases of any EEPROM reflash process, and progress as needed
* Monitor on-going sub-phases of any EEPROM download process, and progress as needed
* Maintain watchdog timer and hardware watchdog process

Similarly the SubBoot program for CPU2 structure is:
* Monitor serial port with CPU1 for incoming commands, execute any command received, and send response as necessary
* Monitor on-going sub-phases of any EEPROM erase process, and progress as needed
* Monitor on-going sub-phases of any EEPROM reflash process, and progress as needed
* Monitor on-going sub-phases of any EEPROM download process, and progress as needed
* Maintain watchdog timer and hardware watchdog process

The canbus instruction set implemented by MainBoot is described in the table below.

| basic command | byte 2,3 options | comments |
|---|---|---|
| 0x20 | none | ping |
| 0x22 | 0xe729-2c | get CPU1 and CPU2 erase counts, CPU1 and CPU2 Flash Volts readings |
| 0x31 | 0xa0, a1, a2, a3, b0, b1, b2, b3 | request erase and reflash eproms |
| 0x32 | 0xa0, a1, a2, a3, b0, b1, b2, b3 | validate erase and reflash requests |
| 0x34 | 0x00, 80 | set upload address target location |
| 0x35 | 0x00, 80 | download 1k bytes from target address |
| 0x36 | - | upload data bytes |
| 0x37 | 0x00, 80 | finalize upload |
| 0x3f | - | ping |


To reflash CPU1/CPU2 EEPROMs, the procedure is:
* optionally read erase count and decide whether to continue (0x22 e7 29)
* reset RTC timeout counter, by	sending read cpu2 flash volts (0x22 e7 2c)
* erase eeproms
  *	initiate erase: send 06 31 a1/b1 00 00 00 00, wait for response 06 7f 31 a1 00 00 00
  * validate erase: send 06 32 a1/b1 00 00 00 00
  * - success is response 06 7f 32 a1/b1 00 00 00
  * - fail is response 06 7f 32 a1/b1 00 58 63
  * - on-going is response 06 7f 32 a1/b1 00 00 21
  * - error is response 06 7f 32 a1/b1 00 4a 63
* program eeproms
  * set number of blocks to be uploaded: 06 31 a0/b0 00 a0 00 00  (for 160 (0xa0) 1k blocks)
  * for each of 160 1k blocks
  * - send target address: 07 34 80/00 04 00 addr1 addr2 addr3, response 06 7f 34 80/00 04 00 addr1
  * - send 171 upload packets: 36 pp qq rr ss tt uu
  * - send upload finished: 02 31 a2/b2, response 06 7f 31 a2/b2 00 00 00
  * - send reset target address: 02 37 80/00, response 06 7f 37 80/00 00 00 00
  * - send block upload complete: 02 32 a2/b2
  * --- success is response 06 7f 32 a2/b2 00 00 00
  * --- fail is response 06 7f 32 a2/b2 00 58 63
  * --- on-going is response 06 7f 32 a2/b2 00 00 21
  * send expected checksum ppqq: 04 31 a3/b3 pp qq, response 06 7f 31 a3/b3 pp qq 00
  * validate checksum: 02 32 a3/b3
  * --- success is response 06 7f 32 a3/b3 00 00 00
  * --- fail is response 06 7f 32 a3/b3 00 58 63
  * --- on-going is response 06 7f 32 a3/b3 00 00 21
  * (note that the checksum computation effectively assumes the last 4 bytes of the 160 kbytes are 0xff ff 00 00, since these contain erase and reflash counts)
  
This will erase and reprogram all the EEPROMs (all 160 kbytes of storage). There is no option to reflash an individual EEPROM (noting that the number of EEPROMs is different depending on whether the CPU is  Y5 or Y6). The erase and reprogram counts, stored in the last EEPROM, will be incremented each time the reflash procedure is executed.


To download the contents of all EEPROMs from CPU1/CPU2
* for each of 160 1k blocks
  * send target address: 07 35 00/80 04 00 addr1 addr2 addr3, response 06 7f 35 00/80 04 00 addr1
  * receive 171 consecutive messages: 07 36 pp qq rr ss tt uu
  * - 171st message will only contain 4 bytes, last two are 00 00
  * - 171st message of last (160th) block is set to 07 36 ff ff 00 00 00 00 (ignore erase/flash counts)


Any tool that will implement the above procedures needs to consider the timeouts for the various commands. In particular, uploads and downloads of 1k byte blocks will require a real time or near real time canbus implementation.

Typical manufacturer tools use a PC in combination with a hardware adaptor, which normally implements a J2534 interface.

An alternative approach, which will be described in the next post, is to use an Arduino Uno R3 with Canbus shield, connected to a PC using Serial Port (Serial over USB).
