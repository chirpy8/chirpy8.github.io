---
title: "AJ27 Reflash Tool Part 1 (Jaguar AJ27 specific)"
classes: wide
categories:
  - Software and firmware
tags:
  - ECU
  - Jaguar
  - AJ27
---
As analyzed in previous posts, the Jaguar AJ27 ECU can be reflashed using Canbus commands. In order to send these commands and receive the responses, a canbus adaptor is needed. A adaptor can be created using an Arduino Uno R3 together with a Canbus shield, which is then controlled by a PC. Several designs of shield are available, and for this tool the Sparkfun Canbus shield is used (SKU: DEV-13262), which uses the MCP2515 CAN controller chip with SPI interface.

The Arduino Uno R3 is very easy to program, including use of interrupts. The main downside is very limited RAM memory (2k bytes), but since the reflash procedures only transfer data in 1k byte chunks, this is adequate.

Software design considerations for this application include:
* Protocol for communication between PC and Arduino, to transfer Canbus commanrds
* Strategies to deal with the reflash protocol timeouts
* Strategies to deal with the very small serial port buffer in the Arduino

Since standard canbus packets have an 11 bit header and 8 bytes of data, a simple implementation for serial port communications between PC and Arduino is to use 10 byte packets. The first 2 bytes contain the 11 bit canbus SID (left justified) and the remaining 8 bytes contain the data. We also need a way for the PC and Arduino adaptor to communicate, and to do this we can use the same format, but reserve some special SID values. We then intercept these packets rather than treating them as Canbus packets.

To send a 10 byte packet to the ECU, a routine such as the following can be used, which interacts with the MCP2515 canbus controller via SPI. First we check if the transmit buffer 1 is empty and available to transmit the packet. Since the MCP2515 has multiple TX buffers, we could use more than one to speed things up, but this requires additional logic to ensure packets are not transmitted out of order, which could introduce errors in certain procedures. So in this case, if the buffer is busy, we just wait until it is emptied, and then continue. Interrupts are disabled whilst interfacing with the MCP2515, because (as will be discussed later) we will also interact with the chip inside interrupt service routines, and need to avoid conflicts. Once the TX buffer is available, it is a simple matter of loading the packet (no manipulation of the SID is required using a left justified definition), and initiating the transmission. At the end of the routine, a timer is reset. This timer is used as a keep-alive, in order to prevent the ECU timing out during key procedures.

```cpp
void sendToECU(byte sendPacket[10]) //sendPacket is assumed to have 10 elements
{
    // use TXbuffer 1
    boolean isBusy = true;

    do
    {
      noInterrupts();
      //read status of TXB 1
      PORTB = 0x02; 
      SPDR = 0xa0; //read status
      while ((SPSR & 0x80) == 0) {};
      temp = SPDR;

      SPDR = 0x00; //receive contents
      while ((SPSR & 0x80) == 0) {};
      spiRx = SPDR;
      PORTB = 0x06;

      interrupts();

      // bit 4 is TXREQ1 status
      isBusy = ((spiRx & 0x10) != 0);
    }
    while (isBusy);

    // send data in TX1 buffer

    noInterrupts();

    PORTB = 0x02;
    SPDR = 0x42; // load TX buffer 1 starting a SIDH
    while ((SPSR & 0x80) == 0) {};
    temp = SPDR;

    SPDR = sendPacket[0]; //load tx buffer with SIDH
    while ((SPSR & 0x80) == 0) {};
    temp = SPDR;

    SPDR = sendPacket[1]; //load tx buffer with SIDL
    while ((SPSR & 0x80) == 0) {};
    temp = SPDR;

    SPDR = 0; //load tx buffer with EID8
    while ((SPSR & 0x80) == 0) {};
    temp = SPDR;

    SPDR = 0; //load tx buffer with EID0
    while ((SPSR & 0x80) == 0) {};
    temp = SPDR;

    SPDR = 0x08; //load tx buffer with DLC
    while ((SPSR & 0x80) == 0) {};
    temp = SPDR;

    for (byte x = 2;x<10;x++) 
    {       
      SPDR = sendPacket[x]; //load tx buffer with D0 - D7
      while ((SPSR & 0x80) == 0) {};
      temp = SPDR;
    }

    PORTB = 0x06; // No CS
    DELAY_CYCLES(2);

    //initiate message transmission by setting TXB1CTRL.TXREQ
    PORTB = 0x02; // No CS   
    SPDR = 0x82; // set TXB1CTRL.TXREQ
    while ((SPSR & 0x80) == 0) {};
    temp = SPDR;

    PORTB = 0x06; // No CS

    TCNT2 = 0x0; //  reset ping timeout since packet just sent

    interrupts();
}
```

We can implement a buffer (called 'ecuBUffer') to store the packets to be sent to the ECU. Packets to be sent are added to the buffer by the function bufferedSendToECU.

```cpp
void bufferedSendToECU(byte bufferedPacket[10])
{
  for (byte x=0;x<10;x++)
  {
    ecuBuffer[ecuWritePointer][x] = bufferedPacket[x];
  }
  ecuWritePointer = ((ecuWritePointer + 1) & 0x07);
}
```

And a routine called 'processECUBuffer()' is called from the main program loop to process this buffer. This also delays sending packets when the 'keepalive' ping command has been sent and a response is expected.

```cpp
void processECUBuffer() //sendPacket is assumed to have 10 elements
{

  //wait if ping just sent until it is acked, or if packet sent and waiting for ack
  if (!pingSent && !packetSent && (ecuReadPointer != ecuWritePointer))
  {
    byte sendPacket[10];

    for (byte i=0;i<10;i++)
    {
      sendPacket[i] = ecuBuffer[ecuReadPointer][i];
    }
    ecuReadPointer = (ecuReadPointer + 1) & 0x7;
    packetSent = true;
    sendToECU(sendPacket);
  }
}
```

To receive Canbus packets from the ECU, an interrupt service routine is used, triggered by the MCP2515 asserting pin D2 when a packet is received. The interrupt service routine first determines which receive buffer has received the packet, since the MCP2515 will be configured to overflow to the second buffer if the first has not been read yet.

```cpp
ISR(INT0_vect)
{
  //receive new canbus packet, check headers, filter, and forward to PC
  //check both RX buffers for a message
  byte txPacket[10];
  boolean receivedData = false;
  boolean pingResponse = false;
  byte canbusRxBuffer = 0;

  //read canstat  register which receive buffer triggered the interrupt
  PORTB = 0x02; //CS
  SPDR = 0x03; // read
  while ((SPSR & 0x80) == 0) {};
  temp = SPDR;

  SPDR = 0x0e; //CANSTAT register
  while ((SPSR & 0x80) == 0) {};
  temp = SPDR;

  SPDR = 0x00; //read contents
  while ((SPSR & 0x80) == 0) {};
  canStat = SPDR;

  PORTB = 0x06;

  if ((canStat & 0x0e) == 0x0c)  //0x0c is RXB0 interrupt
  {
    canbusRxBuffer = 1;
    // indicate to read RX buffer 0
  }
  else if ((canStat & 0x0e) == 0x0e)  //0x0e is RXB1 interrupt
  {
    canbusRxBuffer = 2;
    // indicate to read RX buffer 1
  }
```

Then the appropriate buffer is read, the contents stored in the byte array 'txPacket', and the interrupt cleared. The code below is for RX buffer 0, with RX buffer 1 code omitted (as it is essentially the same).

``` cpp
  if (canbusRxBuffer == 1)
  {
    //read rx buffer 0 - message ID and length
    PORTB = 0x02; //CS
    SPDR = 0x90; // send read RX buffer 0 instruction
    while ((SPSR & 0x80) == 0) {};
    temp = SPDR;
  
    SPDR = 0x00; //receive RXB0SIDH
    while ((SPSR & 0x80) == 0) {};  
    spiRx = SPDR; //store value of sidh in spiRX

    SPDR = 0x00; // start next SPI transfer - receive RXB0SIDL

    //if message from ECU in boot mode, spiRX will equal 0x06 0x00 (id=0x030)
    //if ECU is in normal mode, heartbeat will have spiRX = 0xfa 0x00 (id = 0x7d0)
    //if ECU is sending a diagnostic response, spiRX == 0xfd 0x80 (id 0x7ec)
    // if not any of these then abort rest of data read
    if ((spiRx == 0x06) || (spiRx == 0xfa) || (spiRx == 0xfd))
    {
      receivedData = true;
      txPacket[0] = spiRx;

      while ((SPSR & 0x80) == 0) {};  
      spiRx = SPDR; //store value of sidl in spiRX

      //if message from ECU, spiRX will equal 0x00, if not then abort rest of data read
      if (((spiRx == 0x00) && (txPacket[0] != 0xfd)) || ((spiRx == 0x80) && (txPacket[0] == 0xfd)))
      {
        txPacket[1] = spiRx;
        SPDR = 0x00; //start next SPI transfer - receive RXB0EID8
        while ((SPSR & 0x80) == 0) {};
        temp = SPDR;
 
        SPDR = 0x00; //receive RXB0EID0
        while ((SPSR & 0x80) == 0) {};
        temp = SPDR;

        SPDR = 0x00; //receive RXB0DLC
        while ((SPSR & 0x80) == 0) {};
        spiRx = SPDR; //store value
        dataLength = (spiRx & 0x0f) + 2; //store value - add 2 for loop below

        for (int x=2; x<10;x++) //executes to receive 8 data bytes, DLC field length is ignored
        {
          SPDR = 0x00; //receive RXB0Dn
          while ((SPSR & 0x80) == 0) {};
          txPacket[x] = SPDR;
        }
      }
    }
    
    PORTB = 0x06; // No CS
    DELAY_CYCLES(2);
  
    //clear the interrupt flag CANINTF.RX0IF
    PORTB = 0x02; //CS
    SPDR = 0x05; // send bit modify instruction
    while ((SPSR & 0x80) == 0) {};
    temp = SPDR;
 
    SPDR = 0x2c; // CANINTF register
    while ((SPSR & 0x80) == 0) {};
    temp = SPDR;

    SPDR = 0x01; // mask for RX0IF
    while ((SPSR & 0x80) == 0) {};
    temp = SPDR;

    SPDR = 0x00; // reset RX0IF
    while ((SPSR & 0x80) == 0) {};
    temp = SPDR;

    PORTB = 0x06; // No CS

  }
```

Finally, we store the received packet in a buffer called 'spiBuffer', for further processing outside of the interrupt service routine. 'spiWritePointer' tracks the next write location in the array 'spiBuffer', and is updated after the packet is stored. The variable 'spiBuffer' has storage for 8 packets, so the write pointer wraps around after 8 writes. Also, if the received packet is a response to a 0x3f command (ping) then we set a flag. We will use the ping command as a keep-alive, as discussed earlier, and we will not want the ping process to interfere with other processes, so we reset a flag (which is set when the ping is sent) once the response is received.

```cpp
  if (receivedData)
  {
    for (int x=0;x<10;x++)
    {
      spiBuffer[spiWritePointer][x] = txPacket[x];
    }
    spiWritePointer = ((spiWritePointer + 1) & 0x07);

    //reset ping sent flag if ping response
    if ((txPacket[3] == 0x7f) && (txPacket[4] == 0x00) && (txPacket[2] == 0x02) && (txPacket[0] == 0x06) && (txPacket[1] == 0x00))
    {
      pingSent = false; //used to avoid sending packet to ecu between ping sent and ping ack
    }
  }
```

The above covers exchanging canbus packets with the ECU. Next we have to consider exchanging packets with the PC over serial. To send canbus packets to the PC, we can create an array (also of size 8) called 'transmitBuffer', and add packets received from the ECU to this buffer.

```cpp
void sendTransmitPacket(byte packet[10])
{
  for (int x=0;x<10;x++)
  {
    transmitBuffer[transmitWritePointer][x] = packet[x];
  }
  transmitWritePointer = ((transmitWritePointer + 1) & 0x07);
}
```

Then in the main program loop, we call a function called 'processTransmitBuffer()' that sends the data over the serial port, and ensures that the serial port buffer does not overflow. This routine also pauses sending packets if there is an on-going 'active download' which occurs when reading data from the ECU. In that case, 171 packets are sent overcanbus to the Arduino in very quick succession, comprising 1k block of data. Forwarding of those packets is prioritized in order to avoid buffer overflows.

```cpp
//if Serial transmit has space, and there is data to send, then send it
//pause sending if activeDownload is set
void processTransmitBuffer()
{
  if ((Serial.availableForWrite() >= 10) && !activeDownload)
  {
    if (transmitWritePointer != transmitReadPointer)
    {
      Serial.write(transmitBuffer[transmitReadPointer],10);
      transmitReadPointer = ((transmitReadPointer + 1) & 0x07);
    }
  }
}
```

To receive packets from the PC over the serial port, we create a function call 'checkSerial()', which will be called from the main program loop. This ensures that the serial receive buffer does not overflow. We also use this function to process a number of special packets
* if the packet is part of a 1k block to be uploaded to the ECU, it is intercepted and stored in a special buffer called 'dataBlock'
* if the packet is special ping packet from the PC, it is intercepted and an acknowledgement sent back to the PC
* if the packet signals the start of a 1k block transfer, then counters are initialized and flags set, to prepare to receive the 1k block
* if the packet signals the end of a 1k block transfer, then flags are set to initiate the 1k upload to the ECU CPU1 or CPU2, as appropriate
* if the packet signals the start of a 1k download from the ECU, then flags are set to prepare for the download
* otherwise the packet is forwarded to the ECU

```cpp
// if serial data received
void checkSerial()
{
  int receivedBytes = Serial.available();

  if (receivedBytes >= 10)
  {
    Serial.readBytes(serialRxPacket, 10);

    if ((serialRxPacket[0] == 0x02) && (serialRxPacket[1] == 0x00))  //SID 0x10
    {
      if ((serialRxPacket[2] == 0x07) && (serialRxPacket[3] == 0x36))
      {
        //receive upload data
        if (uploadOngoing)
        {
          if (uploadPacketCount < 170)
          {
            //receive 6 bytes and add to data block
            for (int x=4;x<10;x++)
            {
              dataBlock[uploadByteCounter] = serialRxPacket[x];
              uploadByteCounter++;
            }
            uploadPacketCount++;
          }
          else if (uploadPacketCount == 170)
          {
            //receive final 4 bytes and add to data block
            for (int x=4;x<8;x++)
            {
              dataBlock[uploadByteCounter] = serialRxPacket[x];
              uploadByteCounter++;
            }
          }
        }  
      }
      else if ((serialRxPacket[2] == 0x02) && (serialRxPacket[3] == 0x55) && (serialRxPacket[4] == 0x55))
      {
        //send acknowledge: pcPingAck
        sendTransmitPacket(pcPingAck);
      }
      else if ((serialRxPacket[2] == 0x02) && (serialRxPacket[3] == 0x34) && (serialRxPacket[4] == 0x00))
      {
        //prepare to receive 1k block upload
        uploadOngoing = true;
        uploadPacketCount = 0;
        uploadByteCounter = 0;
        //send acknowledge
        sendTransmitPacket(uploadRequestAck);
      }
      else if ((serialRxPacket[2] == 0x02) && (serialRxPacket[3] == 0x31) && (serialRxPacket[4] == 0xa2))
      {
        uploadCompleteAck1[5] = (byte) uploadPacketCount; //0x0a, 0x00, 0x03, 0x31, 0xa2, packetcount
        sendTransmitPacket(uploadCompleteAck1);
        //PC indicating upload is complete and requests acknowledge
        if ((uploadOngoing) && (uploadPacketCount == 170))
        {
          uploadOngoing = false;
          uploadData(true);
          sendTransmitPacket(uploadAck1);
         }
        else
        {
          uploadOngoing = false;
        }
      }
      else if ((serialRxPacket[2] == 0x02) && (serialRxPacket[3] == 0x31) && (serialRxPacket[4] == 0xb2))
      {
        uploadCompleteAck2[5] = (byte) uploadPacketCount; //0x0a, 0x00, 0x03, 0x31, 0xb2, packetcount
        sendTransmitPacket(uploadCompleteAck2);
        //PC indicating upload is complete and requests acknowledge
        if ((uploadOngoing) && (uploadPacketCount == 170))
        {
          uploadOngoing = false;
          uploadData(false);
          sendTransmitPacket(uploadAck2);
        }
        else
        {
          uploadOngoing = false;
        }
      }
    }
    else if ((serialRxPacket[0] == 0x04) && (serialRxPacket[1] == 0x00) && (serialRxPacket[2] == 0x07) && (serialRxPacket[3] == 0x35))  //SID 0x20, message 35
    {
      activeDownload = true;
      blockCounter = 0;
      downloadByteCounter = 0;
      bufferedSendToECU(serialRxPacket);
    }
    else
    {
      //forward packet to ECU, for SID 0x20 or SID 0x7e8
      bufferedSendToECU(serialRxPacket);
    }
  }
}
```

Earlier I talked about the 'keepalive' ping process to avoid ECU timeout, which is needed after unlocking the ECU reflash commands using a security procedure. This is implemented using the Arduino timer2, which is configured to trigger an interrupt. It sends the canbus 0x3f command to the ECU, which is effectively a ping, every 8ms, after the security process has been executed. Using various flags, as mentioned earlier, this regular ping is effectively hidden and transparent to all the other canbus processes.

```cpp
ISR(TIMER2_COMPA_vect)//interrupt service routine for Timer2
{
  //send a hard coded 01 3f to ECU using SID 0x20 (which is 0x04 0x00 when left justifed)
  // byte ecuPing[] = {0x04, 0x00, 0x01, 0x3f};
  //this is used as a keep-alive to stop timeout when ECU commands have been unlocked
  //especially when downloading 1k data blocks to the Arduino before uploading to the ECU

  byte count=0;

  //wait until TX buffer 0 not busy
  do
  {
    //read status of TXB 0
    PORTB = 0x02; 
    SPDR = 0x03; //read
    while ((SPSR & 0x80) == 0) {};
    temp = SPDR;

    SPDR = 0x30; //TXB0CTRL
    while ((SPSR & 0x80) == 0) {};
    temp = SPDR;

    SPDR = 0x00; //receive contents
    while ((SPSR & 0x80) == 0) {};
    spiRx = SPDR;

    PORTB = 0x06;
    count = count + 1;
  }
  while (((spiRx & 0x08) != 0) && (count < 250));

  //send data in TXB 0
  PORTB = 0x02;
  SPDR = 0x40; // load TX buffer 0 starting a SIDH
  while ((SPSR & 0x80) == 0) {};
  temp = SPDR;

  SPDR = 0x04; //load tx buffer with SIDH
  while ((SPSR & 0x80) == 0) {};
  temp = SPDR;

  SPDR = 0x00; //load tx buffer with SIDL
  while ((SPSR & 0x80) == 0) {};
  temp = SPDR;

  SPDR = 0; //load tx buffer with EID8
  while ((SPSR & 0x80) == 0) {};
  temp = SPDR;

  SPDR = 0; //load tx buffer with EID0
  while ((SPSR & 0x80) == 0) {};
  temp = SPDR;

  SPDR = 0x02; //load tx buffer with DLC
  while ((SPSR & 0x80) == 0) {};
  temp = SPDR;
     
  SPDR = 0x01; //load tx buffer with D0
  while ((SPSR & 0x80) == 0) {};
  temp = SPDR;

  SPDR = 0x3f; //load tx buffer with D0
  while ((SPSR & 0x80) == 0) {};
  temp = SPDR;

  PORTB = 0x06; // No CS
  DELAY_CYCLES(2);

  //initiate message transmission by setting TXB0CTRL.TXREQ
  PORTB = 0x02; // No CS   
  SPDR = 0x81; // set TXB0CTRL.TXREQ
  while ((SPSR & 0x80) == 0) {};
  temp = SPDR;
  PORTB = 0x06; // No CS
  DELAY_CYCLES(2);

  pingSent = true; //set flag indicating ping just sent
}

```

Downloading data from the ECU requires some special handling because the data is transmitted rapidly. Download of a 1k byte block of data is requested using canbus command 0x35. In the 'checkSerial()' function listed earlier, this command is intercepted, and the following measures taken.
* set flag activeDownload to TRUE
* set blockCounter to 0
* set downloadByteCounter to 0

Once the command has been received by the ECU, it will rapidly send 171 canbus packets (using command 0x36) back over the canbus. These are intercepted by the canbus receive process and stored in a 1k byte buffer called 'dataBlock'. Once all the packets have been received, the flag 'readyToForwardDownload' is set to TRUE.

```cpp
      if (activeDownload && (spiDataPacket[2] == 0x07) && (spiDataPacket[3] == 0x36))
      {
        if (blockCounter < 170)
        {
          dataBlock[downloadByteCounter] = spiDataPacket[4];
          downloadByteCounter++;
          dataBlock[downloadByteCounter] = spiDataPacket[5];
          downloadByteCounter++;
          dataBlock[downloadByteCounter] = spiDataPacket[6];
          downloadByteCounter++;
          dataBlock[downloadByteCounter] = spiDataPacket[7];
          downloadByteCounter++;
          dataBlock[downloadByteCounter] = spiDataPacket[8];
          downloadByteCounter++;
          dataBlock[downloadByteCounter] = spiDataPacket[9];
          downloadByteCounter++;
          blockCounter++;
        }
        else if (blockCounter == 170)
        {
          for (int x=4;x<8;x++)
          {
          dataBlock[downloadByteCounter] = spiDataPacket[4];
          downloadByteCounter++;
          dataBlock[downloadByteCounter] = spiDataPacket[5];
          downloadByteCounter++;
          dataBlock[downloadByteCounter] = spiDataPacket[6];
          downloadByteCounter++;
          dataBlock[downloadByteCounter] = spiDataPacket[7];
          downloadByteCounter++;
          }
          readyToForwardDownload = true;
          blockCounter = 0;
          downloadByteCounter = 0;
        }
      }

```

The main loop monitors the 'readyToForwardDownload' flag, and if TRUE then the 'downloadForwarding' flag is set TRUE. This in turn allows repeated calls to the forwardDownloadData() function.

```cpp
  if (readyToForwardDownload)
  {
    readyToForwardDownload = false;
    downloadForwarding = true;
    readOutCounter = 0;
    forwardedCounter = 0;
  }

  if (downloadForwarding)
  {
    forwardDownloadData();
  }
```

The 'forwardDownloadData()' function will then forward the 171 packets to the PC over the serial link, taking care not to overflow the serial port transmit buffer, and then the flags are reset for the next download. These buffering procedures, moving the 1k data block first to the Arduino, and then to the PC, prevent overload of the serial link between PC and Arduino, and ensure that no data is lost.

```cpp
void forwardDownloadData()
{
  byte bufferSize = getTransmitBufferSize();

  if (forwardedCounter < 170)
  {
    if (bufferSize > 3)
    {
      for (int x=4;x<10;x++)
      {
        downloadPacket[x] = dataBlock[readOutCounter];
        readOutCounter++;
      }
      sendTransmitPacket(downloadPacket);
      forwardedCounter++;
    }
  }
  else if (forwardedCounter == 170)
  {
    //only 4 bytes in last block
    if (bufferSize > 3)
    {
      for (int x=4;x<8;x++)
      {
        downloadPacket[x] = dataBlock[readOutCounter];
        readOutCounter++;
      }
      downloadPacket[8] =0;
      downloadPacket[9] =0;

      sendTransmitPacket(downloadPacket);
      downloadForwarding = false;
    }
  }
}
```

A somewhat similar process is used to upload 1k of data to the ECU. In this case, the challenge to is transmit the packets quickly enough, rather than too quickly. Successive 0x36 packets must be received within 850 us of each other, otherwise the ECU will timeout and void the process. The upload initiation command 0x34 from the PC is intercepted, and the 'uploadOngoing' flag is set TRUE.

```cpp
      else if ((serialRxPacket[2] == 0x02) && (serialRxPacket[3] == 0x34) && (serialRxPacket[4] == 0x00))
      {
        //prepare to receive 1k block upload
        uploadOngoing = true;
        uploadPacketCount = 0;
        uploadByteCounter = 0;
        //send acknowledge
        sendTransmitPacket(uploadRequestAck);
      }

```

Once this flag is set, packets from the PC will be intercepted and stored in the buffer 'dataBlock'

```cpp
      if ((serialRxPacket[2] == 0x07) && (serialRxPacket[3] == 0x36))
      {
        //receive upload data
        if (uploadOngoing)
        {
          if (uploadPacketCount < 170)
          {
            //receive 6 bytes and add to data block
            for (int x=4;x<10;x++)
            {
              dataBlock[uploadByteCounter] = serialRxPacket[x];
              uploadByteCounter++;
            }
            uploadPacketCount++;
          }
          else if (uploadPacketCount == 170)
          {
            //receive final 4 bytes and add to data block
            for (int x=4;x<8;x++)
            {
              dataBlock[uploadByteCounter] = serialRxPacket[x];
              uploadByteCounter++;
            }
          }
        }  
      }

```

After a full 1k of data has been uploaded, the PC will issue the command 0x31 a2 (or 0x31 b2, depending on whether the upload is to CPU1 or CPU2). These commands are intercepted, and the function 'uploadData()' is called.

```cpp
else if ((serialRxPacket[2] == 0x02) && (serialRxPacket[3] == 0x31) && (serialRxPacket[4] == 0xa2))
      {
        uploadCompleteAck1[5] = (byte) uploadPacketCount; //0x0a, 0x00, 0x03, 0x31, 0xa2, packetcount
        sendTransmitPacket(uploadCompleteAck1);
        //PC indicating upload is complete and requests acknowledge
        if ((uploadOngoing) && (uploadPacketCount == 170))
        {
          uploadOngoing = false;
          uploadData(true);
          sendTransmitPacket(uploadAck1);
         }
        else
        {
          uploadOngoing = false;
        }
      }
```

The function 'uploadData()' sends all 171 packets of the 1k data block rapidly to the ECU, and then sets the flag 'processingUpload'.

```cpp
void uploadData(boolean targetIs1) //true for cpu1, false for cpu2
{
  uploadByteCounter = 0;

  for (int x=0;x<170;x++)
  {
    for (int i=4;i<10;i++) 
    {       
      ecuUploadPacket[i] = dataBlock[uploadByteCounter]; //load tx buffer with D2 - D7
      uploadByteCounter++;
    }
    sendToECU(ecuUploadPacket);
  }
        
  //send last 4 bytes

  for (int i=4;i<8;i++) 
  {       
      ecuUploadPacket[i] = dataBlock[uploadByteCounter]; //load tx buffer with D2 - D7
      uploadByteCounter++;
  }
  ecuUploadPacket[8] = 0;
  ecuUploadPacket[9] = 0;

  sendToECU(ecuUploadPacket);

  if (targetIs1)
  {
    sendToECU(ecuUploadNotify1);
  }
  else
  {
    sendToECU(ecuUploadNotify2);
  } 

processingUpload = true;

}
```

The 'processingUpload' flag allows interception of the upload complete 0x31 a2/b2 message from the PC, and implements the remaining steps of the 1k block upload protocol. These steps, as described in the previous post, include sending the message 0x37 (clear target address) message, and the 0x32 a2/b2 upload complete message, which must be repeated until a positive acknowledgement is received. Note that full processing of error conditions is not included in the current code implementation.

```cpp
else if (processingUpload)
      {
        if ((spiDataPacket[2] == 0x06) && (spiDataPacket[3] == 0x7f) && (spiDataPacket[4] == 0x31) && (spiDataPacket[5] == 0xa2))
        {
          //if processing upload, and recieve 31 a2 ack, then send msg 37
          byte msg37[10] = {0x04, 0x0, 0x02, 0x37, 0x80, 0, 0, 0, 0, 0};
          sendToECU(msg37);
        }
        else if ((spiDataPacket[2] == 0x06) && (spiDataPacket[3] == 0x7f) && (spiDataPacket[4] == 0x31) && (spiDataPacket[5] == 0xb2))
        {
          //if processing upload, and recieve 31 a2 ack, then send msg 37
          byte msg37[10] = {0x04, 0x0, 0x02, 0x37, 0x00, 0, 0, 0, 0, 0};
          sendToECU(msg37);
        }
        else if ((spiDataPacket[2] == 0x06) && (spiDataPacket[3] == 0x7f) && (spiDataPacket[4] == 0x37) && (spiDataPacket[5] == 0x80))
        {
          //if processing upload, and recieve 37 80 ack, then send msg 32 a2
          byte msg32_a2[10] = {0x04, 0x0, 0x02, 0x32, 0xa2, 0, 0, 0, 0, 0};
          sendToECU(msg32_a2);
        }
        else if ((spiDataPacket[2] == 0x06) && (spiDataPacket[3] == 0x7f) && (spiDataPacket[4] == 0x37) && (spiDataPacket[5] == 0x00))
        {
          //if processing upload, and recieve 37 80 ack, then send msg 32 a2
          byte msg32_b2[10] = {0x04, 0x0, 0x02, 0x32, 0xb2, 0, 0, 0, 0, 0};
          sendToECU(msg32_b2);
        }
        else if ((spiDataPacket[2] == 0x06) && (spiDataPacket[3] == 0x7f) && (spiDataPacket[4] == 0x32) && (spiDataPacket[5] == 0xa2)
        && (spiDataPacket[6] == 0x00) && (spiDataPacket[7] == 0x00) && (spiDataPacket[8] == 0x21))
        {
          //if processing upload, and receive 32 a2 ack indicating still processing, then re-send msg 32 a2
          byte msg32_a2[10] = {0x04, 0x0, 0x02, 0x32, 0xa2, 0, 0, 0, 0, 0};
          sendToECU(msg32_a2);
        }
        else if ((spiDataPacket[2] == 0x06) && (spiDataPacket[3] == 0x7f) && (spiDataPacket[4] == 0x32) && (spiDataPacket[5] == 0xb2)
        && (spiDataPacket[6] == 0x00) && (spiDataPacket[7] == 0x00) && (spiDataPacket[8] == 0x21))
        {
          //if processing upload, and receive 32 a2 ack indicating still processing, then re-send msg 32 a2
          byte msg32_b2[10] = {0x04, 0x0, 0x02, 0x32, 0xb2, 0, 0, 0, 0, 0};
          sendToECU(msg32_b2);
        }
        else if ((spiDataPacket[2] == 0x06) && (spiDataPacket[3] == 0x7f) && (spiDataPacket[4] == 0x32) && ((spiDataPacket[5] == 0xa2) || (spiDataPacket[5] == 0xb2))
        && (spiDataPacket[6] == 0x00) && (spiDataPacket[7] == 0x00) && (spiDataPacket[8] == 0x00))
        {
          //if processing upload, and receive 32 a2 ack indicating still success, then clear uploadProcessing flag
          processingUpload = false;
        }
      }
```

The other two key processes that are intercepted by reading packets received from the ECU, are the security process, and the eeprom reflash validation commands. The latter require also require repeated polling, until a valid acknowledgement received. The security process is initiated by the 0x27 01 command, and the response from the ECU, which contains the seed number, is intercepted. This allows the correct response to be computed and sent back to the ECU. Once this is acknowledged, the 'keepalive' ping process is initialized to prevent the ECU from timing out.

```cpp
      //if message is 04 67 01 xx yy then this is a request for extended commands unlock
      else if ((spiDataPacket[2] == 0x04) && (spiDataPacket[3] == 0x67) && (spiDataPacket[4] == 0x01))
      {
        //retrieve seed values
        unsigned int seedByte0 = spiDataPacket[5] & 0xff;
        unsigned int seedByte1 = spiDataPacket[6] & 0xff;

        //compute unlock codes
        unsigned int temp1 = ((seedByte0) << 8) + (seedByte1);
        unsigned long temp2 = (temp1 * temp1) + 0x5151;
        byte response3 = (byte) (temp2 & 0xff);
        unsigned long temp3 = (temp2 & 0xff00) >> 8;
        byte response4 = (byte) (temp3 & 0xff);

        //send unlock codes
        byte unlock[10] = {0x04, 0x0, 0x04, 0x27, 0x02, response3, response4, 0, 0, 0};
        sendToECU(unlock);
      }
      //if message is 03 67 02 34 then extended commands are unlocked, and need to start sending keep-alive ping
      else if ((spiDataPacket[2] == 0x03) && (spiDataPacket[3] == 0x67) && (spiDataPacket[4] == 0x02) && (spiDataPacket[5] == 0x34))
      {
        // enable 0x3f keep alive ping to ecu
        TCCR2A = 0x02;
        TCCR2B = 0x07;
        OCR2A = 0x7d;  // 9 ms with 1024 prescalar - 0x8c (64 us per tick) (8ms 0x7d) (7 ms 0x6d) (6ms 0x5e) (4ms 0x3e)
        TCNT2 = 0x0;
        TIMSK2 = 0x02;
      }

```

The overall processing of packets received from the ECU, and stored in spiBuffer, is performed via the function 'processSPI()'. First there is a check to see if a new packet is available, using the function 'spiAvailable()'.

```cpp
boolean spiAvailable()
{
  if (spiReadPointer != spiWritePointer)
  {
    return true;
  }
  else
  {
    return false;
  }
}
```

If a packet is available, it is retrieved by function 'getSPIPacket()'

```cpp
void getSPIPacket()
{
  noInterrupts();
  for (int x=0;x<10;x++)
  {
    spiDataPacket[x] = spiBuffer[spiReadPointer][x];
  }
  interrupts();
  spiReadPointer = ((spiReadPointer + 1) & 0x07);
}
```

The overall flow of the program is controlled via the Arduino main loop, and is as follows.

```cpp
void loop() {

  checkSerial();

  if (readyToForwardDownload)
  {
    readyToForwardDownload = false;
    downloadForwarding = true;
    readOutCounter = 0;
    forwardedCounter = 0;
  }

  if (downloadForwarding)
  {
    forwardDownloadData();
  }

  processTransmitBuffer();

  processSPI();

  processECUBuffer();

}
```
A full listing of this arduino code can be found in the Github respository [here](https://github.com/chirpy8/AJ27EcuTool).

The next post will discuss a PC Java based implementation of the AJ27 reflash process, which will use this Arduino canbus adaptor to reflash the ECU.
