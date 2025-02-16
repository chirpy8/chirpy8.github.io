---
title: "AJ27 Firmware Patching (Jaguar AJ27 specific)"
classes: wide
categories:
  - Software and firmware
tags:
  - ECU
  - Jaguar
  - AJ27
  - Ghidra
---

Simple modifications to an AJ27 ECU firmware file can be made using Ghidra. Modification of target instructions or memory values can be identified, and altered by clearing the instruction, and using the hex editor in edit mode. However, the AJ27 firmware includes firmware checksum routines that will cause DTCs to be triggered if the relevant checksums are not updated with the modified code. In particular, errors in CPU1 checksums can trigger P1633 (CPU1 memory failure).

The relevant validation of checksums for CPU1 is as follows (excerpt from F27SC074.b68). First a 16 bit checksum over words from 0x18000-1993e is computed, and should be equal to 0xaa55. If this check passes, the another 16 bit checksum over words from 0xbabf8-baf34 is computed, which should also be equal to 0xaa55. If both checks pass, then bit0 of b10b4 is cleared, which in turn is checked as part of the error conditions for P1633. Words 1993e and baf34 appear to adjusted as necessary to make the checksums equal to the target 0xaa55 value.

![patching code snapshot1]({{ site.url }}{{ site.baseurl }}/assets/images/patching code snapshot1.png)

![patching code snapshot2]({{ site.url }}{{ site.baseurl }}/assets/images/patching code snapshot2.png)

In addition, for both CPU1 and CPU2, the checksum over 0x0-1fffe and b8000-bffff should be 0xaa55. Adjustment words are at 1fff6 and bfff6.

One way of patching a file whilst respecting these checksums is to use a script. The script can read a csv file, with each line containing an address value, and then a series of byte values to be written starting at that address. The script can be used to patch a target firmware file, and then update the checksum adjustment values as necessary. An example script is show below. It requires that the target patch file (exampleName.csv) has the same name as the target firmware file (exampleName.b68") that should be opened in Ghidra codebrowser window prior to running the script. The current script requires that a CPU1 target filename begins with "F27SC" and uses this to force the CPU1 checksums as above. If will save the patched file, using the first 6 characters of the current file name and appending "085.b68" to that name. These settings can obviously be easily changed as required. Note that after the script is run, the file open in the Codebrowser window will have been modifed, so do not save changes if the original pre-patched file is still needed.

```java

import java.io.BufferedReader;
import java.io.File;
import java.io.FileNotFoundException;
import java.io.FileOutputStream;
import java.io.FileReader;
import java.io.IOException;
import java.util.ArrayList;

import ghidra.app.script.GhidraScript;
import ghidra.program.model.address.Address;
import ghidra.program.model.address.AddressOutOfBoundsException;
import ghidra.program.model.mem.MemoryAccessException;
import ghidra.util.exception.CancelledException;

public class F27Sx074_PatchAndSave extends GhidraScript {
	
	// class to hold patch data
	public class patchDataItem
	{
		Address a;
		byte[] values;
		
		public patchDataItem(Address addr, byte[] vals)
		{
			a = addr;
			values = new byte[vals.length];
			System.arraycopy(vals,0,values,0,vals.length);
		}
	
		//don't bother to make object copies...
	
		public Address getAddress()
		{
			return(a);
		}
			
		public byte[] getValues()
		{
			return values;
		}
	}
	
	
	ArrayList<patchDataItem> patchData = new ArrayList<patchDataItem>();

	@Override
	protected void run() throws Exception {
		
		//use current program file to derive patch file name
		
		File loadedFile = getProgramFile();
		String loadedFileName = loadedFile.getName();
		boolean isB68File = (loadedFileName.endsWith(".B68") || loadedFileName.endsWith(".b68"));
		
		if (!isB68File)
		{
			popup("Loaded File is not .b68 format, exiting script");
			return;
		}
		
		int len = loadedFileName.length();
		String patchFileName = loadedFileName.substring(0,len-4);
		patchFileName = patchFileName.concat(".csv");
		
		//ask user to select patch file, and check it is of the correct name
		
		File patchFile = askFile("Select the patch file","Open");
		
		boolean namesMatch = patchFile.getName().equals(patchFileName);
		if (!namesMatch)
		{
			popup("Patch File does not match loaded program, exiting script");
			return;	
		}
		
		//try to load patch file
		
		try ( BufferedReader br = new BufferedReader(new FileReader(patchFile) ))
		{
			String sLine = null;
			do
			{
				sLine = br.readLine();
				if (sLine != null)
				{
					String[] tokens = sLine.split(",");
					if (tokens.length < 2)
					{
						throw new IOException("Bad data format in patch file");
					}	
					Address address = toAddr(Integer.parseInt(tokens[0],16));
					
					int numBytes = tokens.length - 1;
					byte[] byteData = new byte[numBytes];
					
					for (int i=0;i<numBytes;i++)
					{
						int value = Integer.parseInt(tokens[i+1],16);
						byteData[i] = (byte) value;
					}
							
					patchDataItem pdi = new patchDataItem(address, byteData);
					patchData.add(pdi);
				}
			}
			while (sLine != null);
		}
		catch (IOException e)
		{
			e.printStackTrace();
		}
		
		//apply patches to loaded file
		//note that the whole code unit at the target address will be cleared
		//so need to patch code units, not individual bytes
		//note that instructions will not be re-disassembled..
		
		for (patchDataItem i : patchData)
		{
			Address a = i.getAddress();
			clearListing(a);
			setBytes(a,i.getValues());
		}
		
		//recompute checksums and apply to loaded file
		//
		// words 0x18000-1993e have checksum 0xaa55, adjust using 1993e (only for file type "C")
		// words 0x0-1fffe have checksum 0xaa55, adjust using 1fff6
		// words 0xbabf8-baf34 have checksum 0xaa55, adjust using baf34 (only for file type "C")
		// words 0xb8000-bfffe have checksum 0xaa55, adjust using bfff6
		
		boolean isCPU1 = loadedFileName.startsWith("F27SC");
		
		if (isCPU1)
		{
			byte[] bytes = getBytes(toAddr(0x18000), (0x19940-0x18000));
			computeChecksumAdjust(bytes, 0xaa55, toAddr(0x18000), toAddr(0x1993e));
		}
		
		byte[] bytes = getBytes(toAddr(0x0), 0x20000);
		computeChecksumAdjust(bytes, 0xaa55, toAddr(0x0), toAddr(0x1fff6));
		
		if (isCPU1)
		{
			bytes = getBytes(toAddr(0xbabf8), (0xbaf36-0xbabf8));
			computeChecksumAdjust(bytes, 0xaa55,toAddr(0xbabf8),  toAddr(0xbaf34));
		}
		
		bytes = getBytes(toAddr(0xb8000), 0x8000);
		computeChecksumAdjust(bytes, 0xaa55, toAddr(0xb8000), toAddr(0xbfff6));
		
		
		//export the patched file with new file name and b68 format
		
		// 160k without TPU
		// 6 + 1029 * 160 = 164646

		byte[] saveData = new byte[164646];
		
		int targetBlockAddressByte0 = 0;
		int targetBlockAddressByte1 = 0;
		
		byte[] dataBlock = null;

		//write CPU signature
		saveData[0] = 04;
		saveData[1] = 00;
		saveData[2] = 00;
		saveData[3] = (byte) 0xa0;
		
		saveData[4] = (byte) 0x54;
		saveData[5] = (byte) 0xaa;
		saveData[6] = (byte) 0;
		saveData[7] = (byte) 0;
		saveData[8] = (byte) 0;
		
		//last two byte are 0
		saveData[164644] = (byte) 0;
		saveData[164645] = (byte) 0;

		for (int blockCount=0;blockCount<160;blockCount++)
		{
			//compute start address of data block
			//first 128k is 0 - 0x20000
			//next 32k is 0xb8000 - bffff
			
			if (blockCount < 128) //first 128k
			{
				targetBlockAddressByte0 = (byte) (blockCount / 64);
				targetBlockAddressByte1 = (byte) (blockCount % 64)*4;
				dataBlock = getBytes(toAddr(blockCount*1024),1024);
			}
			else if ((blockCount > 127) && (blockCount < 160)) //32k
			{
				targetBlockAddressByte0 = 0xb;
				targetBlockAddressByte1 = (byte)(128+(blockCount - 128)*4);
				dataBlock = getBytes(toAddr(0xb8000+(blockCount-128)*1024),1024);
			}
			
			for (int x=0;x<1024;x++)
			{		
				saveData[9+(blockCount*1029)+x] = dataBlock[x];
			}

			//write header for block, unless first block which is skipped
			if (blockCount != 0)
			{
				saveData[9+(blockCount*1029)-5] = 0x00;
				saveData[9+(blockCount*1029)-4] = 0x00;
				saveData[9+(blockCount*1029)-3] = (byte) targetBlockAddressByte0;
				saveData[9+(blockCount*1029)-2] = (byte) targetBlockAddressByte1;
				saveData[9+(blockCount*1029)-1] = 0x00;
			}
		}
				
		//save file in same path as patch file
		//so create target file name and path
		//name format should be F27SxNNN.b68
		
		String saveFileName = loadedFileName.substring(0,5)+"085.b68";
		String pathName = patchFile.getPath();
		pathName = pathName.substring(0, pathName.length()-12);
		
		File saveFile = new File(pathName,saveFileName);
						
		try (FileOutputStream out = new FileOutputStream(saveFile))
		{
			out.write(saveData);
		}
		catch (FileNotFoundException e)
		{
			e.printStackTrace();
		}
		catch (IOException e)
		{
			e.printStackTrace();
		}
				
		popup("Created new patched file "+saveFile.getPath());

	}
	
	protected void computeChecksumAdjust(byte[] memBlock, int targetChecksum, Address startAddress, Address adjustAddress)
	{
		int checksum = 0;
		int correctionValue = 0;
		int adjustIndex = (int) (adjustAddress.getUnsignedOffset() - startAddress.getUnsignedOffset()) / 2;
		
		for (int i=0;i<(memBlock.length/2);i++)
		{
			if (i != adjustIndex)  //don't include checksum adjust value in the calculation
			{
				int word = (memBlock[i*2] << 8) & 0xff00;
				word += (memBlock[(i*2)+1] & 0xff);
				checksum += word;
				checksum = (checksum & 0xffff);
			}
		}
		
		try {

			correctionValue = ((targetChecksum - checksum) & 0xffff);
			clearListing(adjustAddress,adjustAddress.add(1));
			byte upperByte = (byte) (correctionValue >> 8);
			byte lowerByte = (byte) correctionValue;
			setByte(adjustAddress,upperByte);
			setByte(adjustAddress.add(1),lowerByte);

		} catch (MemoryAccessException e) {
			e.printStackTrace();
		} catch (AddressOutOfBoundsException e) {
			e.printStackTrace();
		} catch (CancelledException e) {
			e.printStackTrace();
		}
		
	}
}
```
