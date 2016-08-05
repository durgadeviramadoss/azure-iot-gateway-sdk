# Identity Map Module Requirements

##Overview
This document describes the identity map module.  This module maps MAC addresses 
to device id and keys, and device ids to MAC Addresses. This module is 
not multi-threaded, all work will be completed in the Receive callback.
 
#### MAC Address to device name (Device to Cloud)
The module identifies the messages that it needs to process by the following 
properties that must exist:

>| PropertyName | Description                                                                  |
>|--------------|------------------------------------------------------------------------------|
>| macAddress   | The MAC address of a sensor, in canonical form                               |

*and* if any of the following *do not exist*.

>| PropertyName | Description                                                                    |
>|--------------|--------------------------------------------------------------------------------|
>| deviceName   | The deviceName as registered with IoTHub                                       |
>| deviceKey    | The key as registered with IoTHub                                              |
>| source       | Set to "mapping" - this shall ensure the mapping module rejects its own messages |

When this module publishes a message, each message will have the the following properties:
>| PropertyName | Description                                                                  |
>|--------------|------------------------------------------------------------------------------|
>| source       | Source will be set to "mapping".                                             |
>| deviceName   | The deviceName as registered with IoTHub                                     |
>| deviceKey    | The key as registered with IoTHub                                            |

#### Device Id to MAC Address (Cloud to Device)

The module identifies the messages that it needs to process by the following properties that must exist:
>| PropertyName | Description                                                                  |
>|--------------|------------------------------------------------------------------------------|
>| deviceName   | The deviceName as registered with IoTHub                                     |
>| source       | Set to "IoTHubHttp"                                                          |

When this module publishes a message, each message will have the following proporties:
>| PropertyName | Description                                                                  |
>|--------------|------------------------------------------------------------------------------|
>| source       | Source will be set to "mapping".                                             |
>| macAddress   | The MAC address of a sensor, in canonical form                               |
      

The MAC addresses are expected in canonical form. For the purposes of this module, 
the canonical form is "XX:XX:XX:XX:XX:XX", a sequence of six two-digit hexadecimal 
numbers separated by colons, ie "AC:DE:48:12:7B:80".  The MAC address is 
case-insenstive.

##References

[module.h](../../../../devdoc/module.md)

[vector.h](../../../../azure-c-shared-utility/c/inc/vector.h)

IEEE Std 802-2014, section 8.1 (MAC Address canonical form)

##Exposed API
```c

typedef struct IDENTITY_MAP_CONFIG_TAG
{
    const char* macAddress;
    const char* deviceId;
    const char* deviceKey;
} IDENTITY_MAP_CONFIG;

MODULE_EXPORT const MODULE_APIS* Module_GetAPIS(void);

```

## Module_GetAPIs

This is the primary public interface for the module.  It returns a pointer to 
the `MODULE_APIS` structure containing the implementation functions for this module.  
The following functions are the implementation of those APIs.

**SRS_IDMAP_17_001 [** `Module_GetAPIs` shall return a non-`NULL` pointer to a MODULE_APIS structure.**]** **SRS_IDMAP_17_002: [**The `MODULE_APIS` structure shall have non-`NULL` `Module_Create`, `Module_Destroy`, and `Module_Receive` fields.**]**

##IdentityMap_Create
```C
MODULE_HANDLE IdentityMap_Create(MESSAGE_BUS_HANDLE busHandle, const void* configuration);
```

This function creates the identity map module.  This module expects a `VECTOR_HANDLE` 
of `IDENTITY_MAP_CONFIG`, which contains a triplet of canonical form MAC 
address, device ID and device key. The MAC address will be treated as the key for the MAC address to device array, and the deviceName will be treated as the key for the device to MAC address array.

**SRS_IDMAP_17_003: [**Upon success, this function shall return a valid pointer to a `MODULE_HANDLE`.**]**
**SRS_IDMAP_17_004: [**If the `busHandle` is `NULL`, this function shall fail and return `NULL`.**]**
**SRS_IDMAP_17_005: [**If the configuration is `NULL`, this function shall fail and return `NULL`.**]**
**SRS_IDMAP_17_041: [**If the configuration has no vector elements, this function shall fail and return `NULL`.**]**
**SRS_IDMAP_17_019: [**If any `macAddress`, `deviceId` or `deviceKey` are `NULL`, this function shall fail and return `NULL`.**]**
**SRS_IDMAP_17_006: [**If any `macAddress` string in configuration is **not** a MAC address in canonical form, this function shall fail and return `NULL`.**]**

Note that this module does not confirm the device ID and key are valid to IoT Hub.

The valid module handle will be a pointer to the structure:

```C
typedef struct IDENTITY_MAP_DATA_TAG
{
    MESSAGE_BUS_HANDLE busHandle;
    size_t mappingSize;
    IDENTITY_MAP_CONFIG * macToDeviceArray;
    IDENTITY_MAP_CONFIG * deviceToMacArray;    
} IDENTITY_MAP_DATA;
```    

Where `busHandle` is the message bus passed in as input, `mappingSize` is the number of 
elements in the vector `configuration`, and the `macToDeviceArray` and `deviceToMacArray` are the sorted lists 
of mapping triplets. 

**SRS_IDMAP_17_010: [**If `IdentityMap_Create` fails to allocate a new `IDENTITY_MAP_DATA` structure, then this function shall fail, and return `NULL`.**]**
**SRS_IDMAP_17_011: [**If `IdentityMap_Create` fails to create memory for the macToDeviceArray, then this function shall fail and return `NULL`.**]**
**SRS_IDMAP_17_042: [** If `IdentityMap_Create` fails to create memory for the deviceToMacArray, then this function shall fail and return `NULL`. **]**   
**SRS_IDMAP_17_012: [**If `IdentityMap_Create` fails to add a MAC address triplet to the macToDeviceArray, then this function shall fail, release all resources, and return `NULL`.**]**
**SRS_IDMAP_17_043: [** If `IdentityMap_Create` fails to add a MAC address triplet to the deviceToMacArray, then this function shall fail, release all resources, and return `NULL`. **]**


##Module_Destroy
```C
static void IdentityMap_Destroy(MODULE_HANDLE moduleHandle);
```

This function released all resources owned by the module specified by the `moduleHandle`.

**SRS_IDMAP_17_018: [**If `moduleHandle` is `NULL`, `IdentityMap_Destroy` shall return.**]**
**SRS_IDMAP_17_015: [**`IdentityMap_Destroy` shall release all resources allocated for the module.**]**



##IdentityMap_Receive
```C
static void IdentityMap_Receive(MODULE_HANDLE moduleHandle, MESSAGE_HANDLE messageHandle);
```

This function will be the main work of this module. The processing of each 
message in pseudocode is as follows:

```
01: If message properties contain a "macAddress" key and does not contain "source"=="mapping", or both "deviceName" and "deviceKey" keys,
02:     Get MAC address from message properties via the "macAddress" key
03:     Search macToDeviceArray for MAC address
04:     If found, there is a new message to publish
05:         Get deviceId and deviceKey from macToDeviceArray.
06:         Create a new MAP from message properties.
07:         Add or replace "deviceName" with deviceId
08:         Add or replace "deviceKey" with deviceKey
09:         Add or replace "source".
10: Else if message properties contain a "deviceName" key and does not contain "source"=="mapping" key,
11:     Get deviceId from messages properties via the "deviceName" key
12:     Search deviceToMacArray for deviceId
13:     If found, there is a new message to publish
14:         Get MAC address from deviceToMacArray
15:         Create a new MAP from message properties.
16:         Add or replace "macAddress" with MAC address.
17:         Replace "source".
18: If there is a new message to publish,
19:         Clone original mesage content.
20:         Create a new message from MAP and original message content.
21:         Publish new message on busHandle
22:         Destroy all resources created
```

**SRS_IDMAP_17_020: [**If `moduleHandle` or `messageHandle` is `NULL`, then the function shall return.**]**
#### MAC Address to device name (D2C)
**SRS_IDMAP_17_021: [**If `messageHandle` properties does not contain "macAddress" property, then the message shall not be marked as a D2C message.**]**   
**SRS_IDMAP_17_024: [**If `messageHandle` properties contains properties "deviceName" **and** "deviceKey", then the message shall not be marked as a D2C message.**]**
**SRS_IDMAP_17_044: [** If messageHandle properties contains a "source" property that is set to "mapping", the message shall not be marked as a D2C message. **]**
**SRS_IDMAP_17_040: [**If the `macAddress` of the message is not in canonical form, the message shall not be marked as a D2C message.**]**
**SRS_IDMAP_17_025: [**If the `macAddress` of the message is not found in the `macToDeviceArray` list, the message shall not be marked as a D2C message.**]**
On a message which passes all checks, the message shall be marked as a D2C message.
**SRS_IDMAP_17_026: [**On a D2C message received, `IdentityMap_Receive` shall call `ConstMap_CloneWriteable` on the message properties.**]**
**SRS_IDMAP_17_027: [**If `ConstMap_CloneWriteable` fails, `IdentityMap_Receive` shall deallocate any resources and return.**]**
Upon recognition of a D2C message, the following transformations will be done to create a message to send:
**SRS_IDMAP_17_028: [**`IdentityMap_Receive` shall call `Map_AddOrUpdate` with key of "deviceName" and value of found `deviceId`.**]**
**SRS_IDMAP_17_029: [**If adding `deviceName` fails,`IdentityMap_Receive` shall deallocate all resources and return.**]**  
**SRS_IDMAP_17_030: [**`IdentityMap_Receive` shall call `Map_AddOrUpdate` with key of "deviceKey" and value of found `deviceKey`.**]**
**SRS_IDMAP_17_031: [**If adding `deviceKey` fails, `IdentityMap_Receive` shall deallocate all resources and return.**]**  

#### Device Id to MAC Address (C2D)
**SRS_IDMAP_17_045: [** If `messageHandle` properties does not contain "deviceName" property, then the message shall not be marked as a C2D message. **]**    
**SRS_IDMAP_17_046: [** If messageHandle properties does not contain a "source" property, then the message shall not be marked as a C2D message. **]**   
**SRS_IDMAP_17_047: [** If messageHandle property "source" is not equal to "IoTHubHttp", then the message shall not be marked as a C2D message. **]**   
**SRS_IDMAP_17_048: [** If the `deviceName` of the message is not found in deviceToMacArray, then the message shall not be marked as a C2D message. **]**   
On a message which passes all these checks, the message will be marked as a C2D message.
**SRS_IDMAP_17_049: [** On a C2D message received, `IdentityMap_Receive` shall call `ConstMap_CloneWriteable` on the message properties. **]**   
**SRS_IDMAP_17_050: [** If `ConstMap_CloneWriteable` fails, `IdentityMap_Receive` shall deallocate any resources and return. **]**   
Upon recognition of a C2D message, the following transformations will be done to create a message to send:
**SRS_IDMAP_17_051: [** `IdentityMap_Receive` shall call `Map_AddOrUpdate` with key of "macAddress" and value of found `macAddress`. **]**   
**SRS_IDMAP_17_052: [** If adding `macAddress` fails, `IdentityMap_Receive` shall deallocate all resources and return. **]**   

#### Message to send exists
Upon recognition of a C2D or D2C message, then a new message shall be published.

**SRS_IDMAP_17_032: [**`IdentityMap_Receive` shall call `Map_AddOrUpdate` with key of "source" and value of "mapping".**]**   
**SRS_IDMAP_17_033: [**If adding source fails, `IdentityMap_Receive` shall deallocate all resources and return.**]**   
**SRS_IDMAP_17_034: [**`IdentityMap_Receive` shall clone message content.**]**
**SRS_IDMAP_17_035: [**If cloning message content fails, `IdentityMap_Receive` shall deallocate all resources and return.**]**   
**SRS_IDMAP_17_036: [**`IdentityMap_Receive` shall create a new message by calling `Message_CreateFromBuffer` with new map and cloned content.**]**   
**SRS_IDMAP_17_037: [**If creating new message fails, `IdentityMap_Receive` shall deallocate all resources and return.**]**   
**SRS_IDMAP_17_038: [**`IdentityMap_Receive` shall call `MessageBus_Publish` with `busHandle` and new message.**]**   
**SRS_IDMAP_17_039: [**`IdentityMap_Receive` will destroy all resources it created.**]**   
