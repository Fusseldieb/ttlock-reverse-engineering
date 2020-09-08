# Repo that aims at reverse-engineering the TTLock protocol

#### You might be asking **why**... 
Well, turns out, our beloved TTLocks only work with a black-box gateway from China, and, as you may have guessed, only work phoning to China. You can't use your TTLock inside your LAN or open any TTLock door remotely if the China servers are down or too slow (and they sometimes are). This also means that if one day the servers get shut down (and they eventually **will**, as do *any* services), we'll lose the remote functionality, which is useful in many cases.

To try and remedy this issue, this Repo aims to reverse-engineer the TTLock SDK (which processes locally) and get it to work with any Bluetooth-capable device, such as a Raspberry Pi 3 or greater. Wouldn't it be neat to make a RPi as a Gateway for your locks? Or maybe even a simple ESP32. Well, it might not be impossible, but some work still needs to be done.

#### Disclaimer before proceeding:
Please be aware that I've never worked with BL or even BLE, so feel free to fix, and most importantly, expect mistakes!

## The basics

The Bluetooth lock name consists of the last 6 MAC digits, in inverted order. For instance, the lock with the name "S202F_abcdef" has the MAC "ee:c6:a2:**ef**:**cd**:**ab**".


### Getting all available services (primary) with gatttool:

```
attr handle: 0x0001, end grp handle: 0x0007 uuid: 00001800-0000-1000-8000-00805f9b34fb
attr handle: 0x0008, end grp handle: 0x000b uuid: 00001801-0000-1000-8000-00805f9b34fb
attr handle: 0x000c, end grp handle: 0x0011 uuid: 00001910-0000-1000-8000-00805f9b34fb
attr handle: 0x0012, end grp handle: 0x0015 uuid: 0000180f-0000-1000-8000-00805f9b34fb
attr handle: 0x0016, end grp handle: 0x001e uuid: 0000180a-0000-1000-8000-00805f9b34fb
attr handle: 0x001f, end grp handle: 0xffff uuid: 00001530-1212-efde-1523-785feabcd123
```

### Getting all available characteristics (handles) with gatttool:

```
handle: 0x0001, uuid: 00002800-0000-1000-8000-00805f9b34fb
handle: 0x0002, uuid: 00002803-0000-1000-8000-00805f9b34fb
handle: 0x0003, uuid: 00002a00-0000-1000-8000-00805f9b34fb
handle: 0x0004, uuid: 00002803-0000-1000-8000-00805f9b34fb
handle: 0x0005, uuid: 00002a01-0000-1000-8000-00805f9b34fb
handle: 0x0006, uuid: 00002803-0000-1000-8000-00805f9b34fb
handle: 0x0007, uuid: 00002a04-0000-1000-8000-00805f9b34fb
handle: 0x0008, uuid: 00002800-0000-1000-8000-00805f9b34fb
handle: 0x0009, uuid: 00002803-0000-1000-8000-00805f9b34fb
handle: 0x000a, uuid: 00002a05-0000-1000-8000-00805f9b34fb
handle: 0x000b, uuid: 00002902-0000-1000-8000-00805f9b34fb
handle: 0x000c, uuid: 00002800-0000-1000-8000-00805f9b34fb
handle: 0x000d, uuid: 00002803-0000-1000-8000-00805f9b34fb
handle: 0x000e, uuid: 0000fff4-0000-1000-8000-00805f9b34fb *
handle: 0x000f, uuid: 00002902-0000-1000-8000-00805f9b34fb
handle: 0x0010, uuid: 00002803-0000-1000-8000-00805f9b34fb
handle: 0x0011, uuid: 0000fff2-0000-1000-8000-00805f9b34fb *
handle: 0x0012, uuid: 00002800-0000-1000-8000-00805f9b34fb
handle: 0x0013, uuid: 00002803-0000-1000-8000-00805f9b34fb
handle: 0x0014, uuid: 00002a19-0000-1000-8000-00805f9b34fb *
handle: 0x0015, uuid: 00002902-0000-1000-8000-00805f9b34fb
handle: 0x0016, uuid: 00002800-0000-1000-8000-00805f9b34fb
handle: 0x0017, uuid: 00002803-0000-1000-8000-00805f9b34fb
handle: 0x0018, uuid: 00002a29-0000-1000-8000-00805f9b34fb
handle: 0x0019, uuid: 00002803-0000-1000-8000-00805f9b34fb
handle: 0x001a, uuid: 00002a24-0000-1000-8000-00805f9b34fb
handle: 0x001b, uuid: 00002803-0000-1000-8000-00805f9b34fb
handle: 0x001c, uuid: 00002a27-0000-1000-8000-00805f9b34fb
handle: 0x001d, uuid: 00002803-0000-1000-8000-00805f9b34fb
handle: 0x001e, uuid: 00002a26-0000-1000-8000-00805f9b34fb
handle: 0x001f, uuid: 00002800-0000-1000-8000-00805f9b34fb
handle: 0x0020, uuid: 00002803-0000-1000-8000-00805f9b34fb
handle: 0x0021, uuid: 00001532-1212-efde-1523-785feabcd123
handle: 0x0022, uuid: 00002803-0000-1000-8000-00805f9b34fb
handle: 0x0023, uuid: 00001531-1212-efde-1523-785feabcd123
handle: 0x0024, uuid: 00002902-0000-1000-8000-00805f9b34fb
handle: 0x0025, uuid: 00002803-0000-1000-8000-00805f9b34fb
handle: 0x0026, uuid: 00001534-1212-efde-1523-785feabcd123

* Some kind of knowledge about what it does, explained below
```

### Handle descriptions

#### `00002a19-0000-1000-8000-00805f9b34fb` [READ]

- Battery percentage, 1 byte

#### `0000fff2-0000-1000-8000-00805f9b34fb` [WRITE NO-RESP]

- Appears to handle exchange of keys to open the lock, after the exchange of data, the lock opens

The packet value seems to consist of following (This is the *first* packet being exchanged going from PHONE to LOCK):

```
-----------------------------------------------------------------------------------------------------
|         HEADER         |      DATA      |             DATA PACKET 2              |  DP 3  |  TAIL |
|7f5a0503020001000155aa20|7c247a4d52ac7ee8 90a9158c42380dca524ffafd927212d375681e97 3dcf670a|59 0d0a|
-----------------------------------------------------------------------------------------------------

Bytes (in DEC):
00 = "Header[0]"
01 = "Header[1]"
02 = "Protocol Type"
03 = "Sub Version" - lockVersion.getProtocolVersion()
04 = "Scene"
05 = "Organization[0]" - lockVersion.getGroupId()
06 = "Organization[1]" - |
07 = "Sub organization[0]" - lockVersion.getOrgId()
08 = "Sub organization[1]" - |
09 = "Command" (ID?)
10 = "Encrypt" (Byte?)
11 = "Length" (Of whole packet)
12 = Actual Data

PACKET TAIL
"59 0d 0a" = "59" is the CRC byte of whole header+data (in this case begins from 7f and ends at 0a) in CRC8/MAXIM format and "0d 0a" occurrs at the end of every packet, which is a carriage return + line feed
```

#### `0000fff4-0000-1000-8000-00805f9b34fb` [NOTIFY]

- The TTLock Official App seems to subscribe to this in order to get back if the lock was opened

Listening on this handle (0x000f) and unlocking the door with the fingerprint yields following result:
```
pi@raspberrypi:~ $ gatttool -b EE:C6:A2:04:9A:F6 --char-write-req --handle=0x000f --value=0100 --listen
Characteristic value was written successfully
Notification handle = 0x000e value: 7f 5a 05 03 02 00 01 00 01 54 00 10 20 d2 fe 7a 7c 82 87 da
Notification handle = 0x000e value: a0 31 80 fa 24 09 1a 3e 19 0d 0a
^C

pi@raspberrypi:~ $ gatttool -b EE:C6:A2:04:9A:F6 --char-write-req --handle=0x000f --value=0100 --listen
Characteristic value was written successfully
Notification handle = 0x000e value: 7f 5a 05 03 02 00 01 00 01 54 00 10 5f 69 3f b8 38 6f 78 1d
Notification handle = 0x000e value: cc 65 c3 b5 16 00 73 75 fe 0d 0a
^C

pi@raspberrypi:~ $ gatttool -b EE:C6:A2:04:9A:F6 --char-write-req --handle=0x000f --value=0100 --listen
Characteristic value was written successfully
Notification handle = 0x000e value: 7f 5a 05 03 02 00 01 00 01 54 00 10 bd 44 1c 43 fe 20 18 83
Notification handle = 0x000e value: 27 a4 f3 ce 30 cf 09 08 c9 0d 0a
```

Notice how the hex `7f 5a 05 03 02 00 01 00 01 54 00 10` is static between all 3 attempts at unlocking the door (that's the header btw). In the second packet, only the final hex `0d 0a` is static. It is probably one packet split in two.

### Encryption

Considerig that the data is always 16 or 32 bytes long, we can guess that it's AES128 CBC encrypted with a padding of 16 (IV 16). However, trying to decrypt it results in bad padding, which means that it isn't "only" AES encrypted. There is also a XOR function involved, which encodes the bytes with the 10th byte (encrypt byte) afaik. More details here: https://reverseengineering.stackexchange.com/questions/25760/getting-the-algorithm-used-inside-this-so-file.

In a nutshell:

```
dscrc_table = [0x00, 0x5E, 0xBC, 0xE2, 0x61, 0x3F, 0xDD, 0x83, 0xC2, 0x9C, 0x7E, 0x20,
    0xA3, 0xFD, 0x1F, 0x41, 0x9D, 0xC3, 0x21, 0x7F, 0xFC, 0xA2, 0x40, 0x1E,
    0x5F, 0x01, 0xE3, 0xBD, 0x3E, 0x60, 0x82, 0xDC, 0x23, 0x07, 0x9F, 0xC1,
    0x42, 0x1C, 0xFE, 0xA0, 0xE1, 0xBF, 0x5D, 0x03, 0x80, 0xDE, 0x3C, 0x62,
    0xBE, 0xE0, 0x02, 0x5C, 0xDF, 0x81, 0x63, 0x3D, 0x7C, 0x22, 0xC0, 0x9E,
    0x1D, 0x43, 0xA1, 0xFF, 0x46, 0x18, 0xFA, 0xA4, 0x27, 0x79, 0x9B, 0xC5,
    0x84, 0xDA, 0x38, 0x66, 0xE5, 0xBB, 0x59, 0x07, 0xDB, 0x85, 0x67, 0x39,
    0xBA, 0xE4, 0x06, 0x58, 0x19, 0x47, 0xA5, 0xFB, 0x78, 0x26, 0xC4, 0x9A,
    0x65, 0x3B, 0xD9, 0x87, 0x04, 0x5A, 0xB8, 0xE6, 0xA7, 0xF9, 0x1B, 0x45,
    0xC6, 0x98, 0x7A, 0x24, 0xF8, 0xA6, 0x44, 0x1A, 0x99, 0xC7, 0x25, 0x7B,
    0x3A, 0x64, 0x86, 0xD8, 0x5B, 0x05, 0xE7, 0xB9, 0x8C, 0xD2, 0x30, 0x6E,
    0xED, 0xB3, 0x51, 0x0F, 0x4E, 0x10, 0xF2, 0xAC, 0x2F, 0x71, 0x93, 0xCD,
    0x11, 0x4F, 0xAD, 0xF3, 0x70, 0x2E, 0xCC, 0x92, 0xD3, 0x8D, 0x6F, 0x31,
    0xB2, 0xEC, 0x0E, 0x50, 0xAF, 0xF1, 0x13, 0x4D, 0xCE, 0x90, 0x72, 0x2C,
    0x6D, 0x33, 0xD1, 0x8F, 0x0C, 0x52, 0xB0, 0xEE, 0x32, 0x6C, 0x8E, 0xD0,
    0x53, 0x0D, 0xEF, 0xB1, 0xF0, 0xAE, 0x4C, 0x12, 0x91, 0xCF, 0x2D, 0x73,
    0xCA, 0x94, 0x76, 0x28, 0xAB, 0xF5, 0x17, 0x49, 0x08, 0x56, 0xB4, 0xEA,
    0x69, 0x37, 0xD5, 0x8B, 0x57, 0x09, 0xEB, 0xB5, 0x36, 0x68, 0x8A, 0xD4,
    0x95, 0xCB, 0x29, 0x77, 0xF4, 0xAA, 0x48, 0x16, 0xE9, 0xB7, 0x55, 0x0B,
    0x88, 0xD6, 0x34, 0x6A, 0x2B, 0x75, 0x97, 0xC9, 0x4A, 0x14, 0xF6, 0xA8,
    0x74, 0x2A, 0xC8, 0x96, 0x15, 0x4B, 0xA9, 0xF7, 0xB6, 0xE8, 0x0A, 0x54,
    0xD7, 0x89, 0x6B, 0x35]
    
def xordata(data_bytes):
    da_len = len(data_bytes)
    for i in range(0,da_len):
        data_bytes[i] ^= a4KeyByte ^ dscrc_table[da_len]
```

The inverse of a XOR function is also XOR [\*](https://stackoverflow.com/a/14279896/3525780) 

Even after trying to XOR and AES decrypt it, it still gets a padding error. Something is obviously missing.

## Decompiling the provided SDK and helping out

The SDK in question is [this one](https://github.com/ttlock/Android_SDK_Demo/tree/master/app/src/main/java/ttlock/demo) and to get the source code from it, simply unpack it and run `classes.jar` through Procyon or CFR (Online tool [here](http://www.javadecompilers.com/)). The `.so` file, which basically just contains a basic CRC8/MAXIM encode/decode algorithm, you can find inside the SDK by unpacking it and looking into `jni/<platform>`.
