# VAG ME7 Logging RCE exploit

## Description
This repository details the approach that can be used to inject a custom diagnostic service into a running VAG ME7 ECU, and provides a handler, as well as the Keil simulator config for development.

## License
The handler is original work and is licensed under GPLv3. If you intend to use it in software that you distribute, you must also distribute the source code of your software.

## Background information
### Introduction
ME7 on VAG is pretty much completely open. Reading from the flash is not supported outside of *RequestUpload*, but there is *ReadMemoryByAddress* that lets you read the whole RAM, and *WriteMemoryByAddress* that lets you write the whole RAM.

For datalogging however, there is no service that can return more than one ram address in one request. The $2C service is limited to a single memory address, and $23 supports a single request by design. This way logging is limited to 50hz with a single variable, and only when using a COM port interface that ignores all standards and sends data as fast as possible. Logging more variables meaning dividing this maximum 50hz rate by the number of variables - e.g. 25 variables would only log at 2hz.

There is an old exploit used by APR in their ECU Explorer software, where the $2C service routine is hijacked, but it causes quite high CPU load and is not applicable to non-VAG ME7 control units, which do not have $2C service available. Injecting a custom handler is more efficient (albeit much more complicated).

### The service distributor
Most ECU's have some kind of distributor routine, which looks at the incoming diagnostic service ID and runs a handler routine based on the SID number. Usually this is a table in flash, and this is also the case for ME7 (actually there are two tables, one specifies the offset of the other table).

What makes ME7 a little special is that it has an internal BootRom. This BootRom contains a full set of diagnostic services, including the service table, the service distributor itself is also in BootRom code. Then there's another service table and some more services in the ASW, which only come into play once the ASW starts.

To save space, the ASW calls into the BootRom for the service distributor routine, and the internal ram contains a pointer to the service table.
When the BootRom boots up, it sets the pointer to the BootRom internal service table, and when the ASW starts, it changes this pointer to point to the ASW service table instead.

### The service table
The service table simply contains a bunch of 32bit addresses. The first address is always the SNS (*ServiceNotSupported*) routine. This routine gets called by the distributor if you call a service that does not exist. It sends a 3 byte negative response `7F <SID> 11`.
Immediately following is a 8 bit table with offsets, e.g. for service $3E, you'd take offset 0x3E from that table, get the number, and use this number as an offset of 32 bit values from the first table to arrive at the handler.

## The exploit
### Re-directing the service table
Because we have full *WriteMemoryByAddress* access to the RAM, we can write a new service table into the memory. Actually it is enough to write a single value at the first address and immediately call a non-existing service, which makes the ECU execute code from the specified address. For good measure, and to avoid any ECU resets in case some other service gets sent, you can fill the entire table with your subroutine address. Around 40 entries are enough, there's usually not more service entries than that in any ME7.

The next step is to write your handler to the memory address you wrote into the table. Unless you want to hardcode all the variables for every ECU or change them in the code, it makes sense to fill the information the handler will need to successfully operate at a fixed address, then read those addresses in the handler.

### Setting up the handler
The following table lists the information the handler needs, and example addresses for the BootRom found in the `8D0907551K 0001` ECU variant.
There are many different BootRoms and the detection and pattern matching of the data for the others will be left as an excercise to the reader.
| Name  | C Type |Description  | 551K |
| :- |:- | :- | -: |
| resptype | __near&nbsp;uint16_t* | Service response type. AND with 0xF7FF for negative response, OR with 0x800 for positive response | `0xE074` |
| orgdistadr | __far uint32_t* | Far ptr to original service table, read out from `0xE228` | `0x81ABBE` |
| recbufptr | __near uint8_t** | Ptr to table ptr. This changes between service calls, depending on format bytes. The handler must always dereference the pointer. | `0xE1CE` |
| reclen | __near uint_8_t* | Length of request | `0xE1CA` |
| setupcomplete | uin16_t | Flag indicating handler is initialized, set to not 0 | `0xFFFF` |

The response buffer is passed as __near uint8_t* in the first argument (R12) to the handler.
The return value uint8_t (RL4) is the length of the response in bytes.

The handler provided here talks on SID $BE (response on $FE). Normally when no handler is installed, when you send $BE, the ECU will reply with SNS (`7F BE 11`). However, if the handler is installed, the handler will reply with IMLOIF instead (`7F BE 13`).

### First execution of the handler
Once the handler information is set up, it is time to redirect the service table by writing the new service table address to the far ptr at `0xE228`.
As soon as this is done, the next call is just the handler SID $BE without any arguments. If everything went well, the handler will reply with `7F BE 13`, otherwise the ECU will crash and there will be a timeout.

A flag is checked during the execution of the handler, if it is not zero, then the initialization needs to be run.
The new RAM service table has to be populated with valid data, apart from the first position. The handler loops over the original service table from the 2nd position and copies this data into the new service table. We still want the first SNS routine to always redirect to our handler, because the service distributor will not find our handler in the service table, and always call the SNS routine. The handler can then check the service ID, and if it is not ours just call the SNS routine, the near ptr of which we store in the first half of orgdistadr during initialization.

### Logging data
The handler takes just a 2 byte argument, which is a pointer into the `#038h` segment. This address should contain an address array, which should be written there using the $3D service before calling the handler.

Each entry of the array consists of four bytes. The first byte is the data length to read, and the remaining three bytes contain the 24 bit HiLo address. The array should always be terminated by a single 0x00 (meaning a zero length). There are no alignment restrictions.

Care should be taken that the response does not exceed 254 bytes (maximum size of KWP2000 response - 1, as the SID has to fit), it is also not recommended to log more than ~80 variables in one request, as this starts to cause >5% cpu load. To work around this, multiple address arrays should be written into the extram and multiple requests made with different addresses. The sampling rate will be a little lower, but the extra CPU load also won't exceed 5%.

## Notes
The handler uses the same memory area as ME7Logger's handler to avoid interfering with custom code, as ME7Logger has been the gold standard for a very long time (good job setzi62) and most recent code has likely been designed with it mind.

However, because the handler is 40% smaller and better optimized (around 35% less instructions executed in tight loop), it causes less CPU load and more space remains to write the ram address arrays.
