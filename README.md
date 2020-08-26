# Repo that aims at reverse-engineering the TTLock protocol

## The basics

The Bluetooth lock name consists of the last 6 MAC digits, in inverted order. For instance, the lock with the name "S202F_abcdef" has the MAC "ee:c6:a2:**ef**:**cd**:**ab**".

### Getting all primary UUIDs with gatttool:

```
attr handle: 0x0001, end grp handle: 0x0007 uuid: 00001800-0000-1000-8000-00805f9b34fb
attr handle: 0x0008, end grp handle: 0x000b uuid: 00001801-0000-1000-8000-00805f9b34fb
attr handle: 0x000c, end grp handle: 0x0011 uuid: 00001910-0000-1000-8000-00805f9b34fb
attr handle: 0x0012, end grp handle: 0x0015 uuid: 0000180f-0000-1000-8000-00805f9b34fb
attr handle: 0x0016, end grp handle: 0x001e uuid: 0000180a-0000-1000-8000-00805f9b34fb
attr handle: 0x001f, end grp handle: 0xffff uuid: 00001530-1212-efde-1523-785feabcd123
```
