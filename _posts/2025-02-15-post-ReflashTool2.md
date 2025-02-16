---
title: "AJ27 Reflash Tool Part 2 (Jaguar AJ27 specific)"
classes: wide
categories:
  - Software and firmware
tags:
  - ECU
  - Jaguar
  - AJ27
---

This post complements the previous post on an Arduino canbus adaptor for Jaguar AJ27 ECU, and describes a Java implementation of the following processes:
* Sending individual canbus commands to read certain ECU parameters
* Uploading the reflash firmware programs to CPU1 and CPU2
* Reflashing CPU1 and CPU2
* Downloading the current firmware from CPU1 and CPU2
* Downloading the RAM contents of CPU1 and CPU2
* Downloading the key persistent data from the ECU (Configuration and VIN)
* Uploading key persistent data to the ECU (for cloning)

The framework can obviously be extended to add additional functions, such as downloading 'blackbox' data, reading DTCs, etc., but the focus here is on reflash, not on creating an ECUTool that overlaps other tools already available.

For the arduino adaptor, a 10 byte packet format was defined to transfer canbus packets between the PC and the Arduino over serial. A class called 'CanbusMessage' is defined to represent these 10 byte messages. Various methods are added to examine the SID and the data, and also to send the message to the arduino over a serial port.

```java
	protected class CanbusMessage
	{
		private byte[] id; //11 bit left justified
		private byte[] data; //8 bytes
		private String commandDetails;
				
		//constructor for creating messages
		//header is 11 bit left justified
		//so 0x10<>02 00, 0x20<>04 00, 0x30<>06 00, 0x50<> 0a 00
		//info is up to 7 byte message, in info[1] to info[7]
		//data[0] is set to message length
		CanbusMessage(int header, byte[] info, String cmd)
		{
			id = new byte[2];
			data = new byte[]{0,0,0,0,0,0,0,0};
			id[0] = (byte) ((header >> 3) & 0xff);
			id[1] = (byte) ((header & 0x7) << 5);
			int infoLength = info.length;
			if (infoLength > 7)
				infoLength = 7;
			for (int x=0; x < info.length; x++)
			{
				data[x+1] = info[x];
			}
			data[0] = (byte) infoLength;
			commandDetails = cmd;
		}
		
		//constructor for receiving messages
		// first 2 bytes are ID
		// subsequent 8 bytes are data
		CanbusMessage(byte[] message)
		{
			if (message.length == 10)
			{
				data = new byte[8];
				id = new byte[2];
				id[0] = message[0];
				id[1] = message[1];
				for (int x=0;x<8;x++)
				{
					data[x] = message[x+2];
				}
				commandDetails = "";
			}
			else
			{
				data = new byte[]{0,0,0,0,0,0,0,0};
				id = new byte[]{0,0};
				commandDetails = "null message";
			}
		}
		
		int getID()
		{
			int value = ((id[0] & 0xff) << 3) + ((id[1] & 0xff) >> 5);
			return value;
		}
		
		byte[] getData()
		{
			byte[] value = new byte[8];
			System.arraycopy(data, 0, value, 0, 8);
			return value;
		}
		
		void updateData(byte[] newData)
		{
			//updates the data bytes, not the ID. Length is computed
			int length = newData.length;
			if (length > 7)
				length = 7;
			if (length < 1)
				return;
			
			data[0] = (byte) length;
			for (int x=1;x<length;x++)
			{
				data[x] = newData[x-1];
			}
		}
		
		public String toDetailsString()
		{
			String s = commandDetails;
			s = s + String.format(": header 0x%3x : data(hex) %2x %2x %2x %2x %2x %2x %2x %2x",
					this.getID(), data[0], data[1],data[2],data[3],data[4],data[5],data[6],data[7]);
			return s;
		}
		
		public String toString()
		{
			return commandDetails;
		}
				
		boolean sendMessage()
		{
			byte[] txData = new byte[10];
			txData[0] = id[0];
			txData[1] = id[1];
			for (int x=0;x<8;x++)
			{
				txData[x+2] = data[x];
			}
			
			if (arduinoPort.isOpen())
			{				
				arduinoPort.writeBytes(txData, 10);
				return true;
			}
			else
			{
				return false;
			}
		}
	}
```

For serial port implementation [JSerialComm](https://fazecast.github.io/jSerialComm/) is used. To select the serial port connected to the Arduino, we can use the following function, called 'selectSerialPort()', which assigns the relevant port to the global variable 'arduinoPort'.


```java
	private void selectSerialPort()
	{
		SerialPort[] availableSerialPorts = SerialPort.getCommPorts();
		boolean portsAvailable = (availableSerialPorts.length != 0);
		
		JPanel panel = new JPanel();
		
		if (portsAvailable)
		{
	        panel.add(new JLabel("Please make a selection:"));
		}
		else
		{
	        panel.add(new JLabel("No ports currently available"));			
		}
		
        DefaultComboBoxModel<SerialPort> model = new DefaultComboBoxModel<SerialPort>(availableSerialPorts);
               
        JComboBox<SerialPort> comboBox = new JComboBox<SerialPort>(model);
        panel.add(comboBox);

        int result = JOptionPane.showConfirmDialog(null, panel, "Port", JOptionPane.OK_CANCEL_OPTION, JOptionPane.QUESTION_MESSAGE);
        switch (result) {
            case JOptionPane.OK_OPTION:
            	if (portsAvailable)
            	{
            		arduinoPort = (SerialPort) comboBox.getSelectedItem();
            		serialPortLabel.setText(arduinoPort.toString());
            		configureArduinoPort();
            		setupArduinoAndEcu();
            	}
                break;
        }
		
	}
```

After 'arduinoPort' is initialized, the methods 'configureArduinoPort() and 'setupArduinoAndEcu()' are called. The first of these configures the serial port, creates 'PacketListener' to receive canbus messages from the arduino, and creates 'receiveQueue' to store these messages.

```java
	private void configureArduinoPort()
	{
		executorService1.execute(new Runnable() {
			@Override
			public void run() {
				//configure serial port: 8 bits, no parity, 1 stop bit
				arduinoPort.setComPortParameters(baudRate, 8, SerialPort.ONE_STOP_BIT, SerialPort.NO_PARITY);
				// open the port, no checking if successful..
				arduinoPort.openPort();
				//configure serial events - fixed 10 byte receive packet, event based
				PacketListener listener = new PacketListener();
				arduinoPort.addDataListener(listener);
				receiveQueue = new ReceiveQueue();
			}
		});	
	}
```

'Packetlistener' defines the 10 byte packet format, and handles the disconnect event and the datareceived event. If data is received, a new canbusMessage object is created. Received messages are filtered to only accept SIDs 0x30 (which is the SID used for canbus message responses from the ECU), 0x50 (which is used for PC <> Ardunio message exchange) and 0x7ec (which is reponse to canbus commands in normal mode). If the ECU is in bootmode (flashvolts applied and flash comms port asserted) , then it will not be generating the usual canbus traffic, which will include the ECU heartbeat message, with SID 0x7d. Detection of this heartbeat message indicates that the ECU is not in bootmode but in normalmode, and the GUI can be updated accordingly.

```java
	private final class PacketListener implements SerialPortPacketListener
	{
		@Override
	   public int getListeningEvents() 
		{ return (SerialPort.LISTENING_EVENT_DATA_RECEIVED | SerialPort.LISTENING_EVENT_PORT_DISCONNECTED); }
		
		@Override
		   public int getPacketSize() { return 10; }
		
	   @Override
	   public void serialEvent(SerialPortEvent event)
	   {
		   if ((event.getEventType() & SerialPort.LISTENING_EVENT_PORT_DISCONNECTED) != 0)
		   {
			   //remove listener and close port
			   arduinoPort.removeDataListener();
			   arduinoPort.closePort();
			   arduinoDetected = false;
			   ecuBootMode = false;
			   ecuNormalMode = false;
			   serialPortLabel.setText("<Unknown>");
			   arduinoStatusLabel.setText("Disconnected");
			   commandButton.setEnabled(arduinoDetected);
			   ecuStatusLabel.setText("Disconnected");
			   serialSelectButton.setEnabled(true);
		   }
		   else
		   {
			   // read data from serial receive
				CanbusMessage newMessage = new CanbusMessage(event.getReceivedData());
				
				int id = newMessage.getID();
				 
				//filter out ecu heartbeat, and use to detect if ecu is in normal mode
				//if not, set status to normal mode
				if (!ecuNormalMode && (id == 0x7d0))
				{
					ecuBootMode = false;
					ecuNormalMode = true;
					ecuStatusLabel.setText("Inactive - Normal");
					updateNormalCommandsSelection();
				}
				
				//process messages
				if ((id == 0x030) || (id == 0x50)  || (id == 0x7ec))
				{
					receiveQueue.add(newMessage);
				}
				
		   }
	   }
	}
```

Canbus packets that are received, after SID filtering, are added to a ReceiveQueue class, which basically is a buffer to hold received messages.

```java
	protected class ReceiveQueue
	{
		private ArrayList<CanbusMessage> receiveMessages;

		ReceiveQueue()
		{
			receiveMessages = new ArrayList<CanbusMessage>();
		}
		
		protected void add(CanbusMessage m)
		{
			synchronized(this)
			{
				receiveMessages.add(m);
			}
		}
		
		protected CanbusMessage getFirstElement()
		{
			synchronized(this)
			{
				return receiveMessages.get(0);
			}
		}
		
		protected int size()
		{
			synchronized(this)
			{
				return receiveMessages.size();
			}
		}
		
		protected void removeFirstElement()
		{
			synchronized(this)
			{
				receiveMessages.remove(0);
			}
		}
		
		protected void flush()
		{
			synchronized(this)
			{
				receiveMessages.clear();
			}
			
		}

	}
```

Once the serial port has been initiatilized, the 'setupArduinoAndEcu()' method is called to ping the arduino and check it is connected. If successful, and no normalmode heartbeat has been detected from the ECU, the ECU is pinged to determine if it is in bootmode, and if so the GUI is updated accordingly.

```java
	private void setupArduinoAndEcu()
	{
		//this method is called once a Serial port has been selected
		//   see selectSerialPort()

		monitorTextArea.append("Sending arduino ping\n");		
		CanbusResponses reply = canbusRequestAndResponses(5, pingArduino,
				new byte[] {(byte) 0x02, (byte) 0xaa, (byte) 0xaa}, 0x50, null, 1000L);
		printRequestResult(reply);
		
		if (reply.getResult())
		{
			monitorTextArea.append("Received arduino ping response\n");
			arduinoDetected = true;
			//enable commands
			arduinoStatusLabel.setText("Connected");
			serialSelectButton.setEnabled(false);
		}
		
		//delay to check for ecu heartbeat in normal mode
		long start = System.currentTimeMillis();
		long end = start + 1000;
		
		while (System.currentTimeMillis() < end) {};
						
		//see if ecu is in normal mode
		if (arduinoDetected && (ecuNormalMode == true))
		{
			savePersistentButton.setEnabled(true);
			saveRAMButton.setEnabled(true);
			commandButton.setEnabled(true);
		}
		
		//if not in normal mode, see if ecu is in boot mode
		if (arduinoDetected && !ecuNormalMode)
		{
			monitorTextArea.append("Sending ecu ping\n");			
			reply = canbusRequestAndResponses(5, pingECU,
					new byte[] {(byte) 0x07, (byte) 0x62, (byte) 0xe7, (byte) 0x25}, 0x30, null, 1000L);
			printRequestResult(reply);
			
			if (reply.getResult())
			{
				monitorTextArea.append("Received ecu boot mode ping response\n");
				ecuBootMode = true;
				ecuNormalMode = false;
				//enable commands
				ecuStatusLabel.setText("Active");
				commandButton.setEnabled(true);
				//retrieve eeprom signatures
				retrieveEepromSignatures();
			}
		}

	}
```

Most of the interactions with the ECU are in the form of a canbus request and response exchange. A method 'canbusRequestAndResponses' is defined to take care of these interactions. The request message is sent, and then responses are evaluated to see if any have an ID that matches 'responseID' parameter, and data contents that match 'responseVector' parameter. The 'responseVector' parameter can be shorter than the possible full 8 bytes, allowing matching of only some of the initial data bytes of the message. Also a 'suppressVector' parameter can be added that will prevent matches of certain data patterns. A timeout is specified using the parameter waitTime, and if there is no match after this times out, then the 'requestMessage' will be resent, up the the number of times specified by the 'attempts' parameter. Any responses that are received, whether match the responseVector or not, are added to the 'responses' List, unless they match the 'suppressVector', in which case they are ignored. The method will return when either there has been a successful match of a response to the request, or the process has timed out after the specified number of allowed iterations. An object of type CanbusResponses is created as the returned parameter, which will contain the outcome (targetResponse received or not, the targetResponse message (if received), and all the other responses received).

```java
	private CanbusResponses canbusRequestAndResponses(int attempts, CanbusMessage requestMessage, byte[] responseVector, int responseID,
			byte[] suppressVector, long waitTime)
	{
		List<CanbusMessage> responses = new ArrayList<CanbusMessage>();

		//returns expected response as List of Bytes, or null if incorrect response
		int iterations = attempts;
		byte[] msgResponse = new byte[8];
		boolean correctResponse = false;
		boolean successResult = false;
		CanbusMessage targetResponse = null;
				
		//send request message
		//if no response received, exception will be triggered and iterations decremented
		//if response is received, and matches expected reply, true is returned
		while ((iterations > 0) && !correctResponse)
		{
			// set a timeout
			long start = System.currentTimeMillis();
			long end = start + waitTime;
			boolean timeOut = false;
					
			//send message
			requestMessage.sendMessage();
												
			//get correct response or timeout
			while ( !correctResponse && !timeOut)
			{
				if (receiveQueue.size() != 0)
				{
					CanbusMessage rxMessage = receiveQueue.getFirstElement();
					//can remove message from real queue
					receiveQueue.removeFirstElement();
					boolean suppress = true;
					if (rxMessage.getID() == responseID)
					{
						msgResponse = rxMessage.getData();
						int responseCount = responseVector.length;
						correctResponse = true;
						for (int x=0;x<responseCount;x++)
						{
							correctResponse = correctResponse && ( msgResponse[x] == responseVector[x]);
						}
						
						if (correctResponse)
						{
							targetResponse = rxMessage;
						}
						
						if (suppressVector != null)
						{
							int suppressCount = suppressVector.length;
							for (int x=0;x<suppressCount;x++)
							{
								suppress = suppress && ( msgResponse[x] == suppressVector[x]);
							}
						}
						else
						{
							suppress = false;
						}
					}
					if (!suppress)
					{
						responses.add(rxMessage);
					}
				}
				//check for timeout
				if (System.currentTimeMillis() > end)
				{
					timeOut = true;
				}
			}
					
			//if timeout, decrement iterations and try again
			if (!correctResponse)
			{
				iterations--;
			}
			else
			{
				successResult = true;
			}
		}
		
		return new CanbusResponses(successResult, targetResponse, responses);
	}	
```

The remainder of the tool implements the various command processes, as detailed in an earlier post, and some very basic GUI functions (using Swing) to identify the various files required.

The Jaguar .b68 file format is used for the reflash firmware files, and the Jaguar supplied reflash boot code files. To reflash, apply >= 15.3V to Flash Programming Voltage and pull Flash Comms Control Port (FCCP) low, then apply 12V power to the ECU. Click "Select Serial Port" button, wait a couple of seconds, and check that Arduino Status in the top right changes to "Connected" and Ecu Boot Mode changes to "Active". Use the buttons to select the required CPU1 and CPU2 Programmer Files (these are the MainBoot and SubBoot files, e.g. F27SA074.b68 and S27SB074.b68), then push the "Upload Programmers" button to load these into the ECU. Select the required reflash firmware loads using the CPU1 and CPU2 Code File buttons (e.g. F27SC074.b68 and F27SD074.b68), and then click the "Reflash EEPROMS" button to reflash the ECU.

The .b68 format is also used for downloaded firmware files, and CPU ram data files. To download, ensure that ECU Boot Mode is "Active" as above. The click the Download EEPROMs or SaveRAM buttons as appropriate, and provide a file name (no extension).

For ECU persistent data, a '.p68' file format is defined, which is a file of 31 bytes, with the first 2 bytes being 0x35 0x7e, and the remaining 29 bytes being the key persistent data bytes, as stored in the persistent data eeprom on the ECU (IC202, 93C86C 2k byte eeprom). This data can be accessed with the ECU in normal mode, i.e. ECU powered up without flash volts being applied and FCCP being asserted. The first 13 bytes are configuration data for the ECU, such as options fitted, fueling parameters, key information, etc. and are model and car specific. Bytes 14-24 are the car VIN code (First 8 bytes are ascii encoded Vin characters 4-11, and the last 4 bytes are hex values, the initial 3 VIN characters "SAJ" are not included). The final 5 bytes are additional configuration data. To download the persistent data, click the "Save Persistent Data" button and provide a file name (no extension). This data can be uploaded to the ECU by selecting the appropriate file using the "Persistent Data File" button, and then clicking the "Program Persistent Data" button. This (to be confirmed...) allows cloning of an existing ECU, but note that the entire contents of the persistent data EEPROM (including stored DTCs, black box data, etc.) will be erased by this procedure. 

Potential future improvements to this tool could include much improved handling of various error cases, and coding a much better GUI, but the tool is functional 'as is'. The current implementation includes the JSerialComm library as a .jar file (v10.4), and the program is exported as an executable .jar file. If the program does not execute on the PC, it likely due to incorrect Java version. Attempting to execute using the command line will display relevant error messages to confirm if this is the case. The source file can be found on Github [here](https://github.com/chirpy8/AJ27EcuTool).
