# RakNet Protocol Documentation

This is the latest RakNet protocol documentation. It includes information on the data types used in the protocol and details about each packet and their associated fields.

TODO: modify to make it better for the eye and remove a lot of useless stuff.
And also organize things, such as enums/constants in 1 place, and also remove repeated things.

This documentation assumes you want similar/same security method of libcat, that's why if the server/client you're targeting doesn't have libcat or whatever same as that but they also have security enabled, the client will crash because they will only give you a cookie, no identity proof or anything of the likes. but you can pull request for supporting both cases or you can wait until this is rewritten.

## Data Types

| Type | Size | Note |
| ---- | ---- | ---- |
| uint8 | 1 byte | Unsigned 8-bit integer |
| uint16 | 2 bytes | Unsigned 16-bit integer |
| uint24 | 3 bytes | Unsigned 24-bit integer with a minimum value of 0 and a maximum value of 16777215 |
| uint32 | 4 bytes | Unsigned 32-bit integer |
| uint64 | 4/8 bytes | Unsigned 64-bit integer (4 bytes for 32-bit systems, 8 bytes for 64-bit systems) |
| uint16-string | variable | UTF-8 encoded string with a length of 2 bytes preceding the string |
| magic | 16 bytes | An array of unsigned 8-bit integers with a specific sequence `[0x00, 0xFF, 0xFF, 0x00, 0xFE, 0xFE, 0xFE, 0xFE, 0xFD, 0xFD, 0xFD, 0xFD, 0x12, 0x34, 0x56, 0x78]` |
| pad-with-zero | variable | Null bytes used for padding with a size of your choice |
| bool | 1 byte | Write or read as a single unsigned 8-bit integer, with a value of 0 or 1 (Zero is used to represent false, and One is used to represent true) |
| address | 7-29 bytes | IPv4: 1 byte (address version), 4 bytes (IP address) and to decode you would `bitwise NOT` the uint32 that you got and then do endian-related things to decode it and then join each part with the seperator and if you are encoding it then do the other way around, 2 bytes (port), IPv6: 1 byte (address version), unsigned short for address family (in little-endian), unsigned short for port number, unsigned integer for flow info, 16 bytes for the address, an unsigned integer for the scope ID. |
| bit | 1 bit | Write or read the bit inside the buffer after you completed it |
| float | 4 bytes | IEEE 754 single-precision floating-point number |

## General Constants

You can instantly define those without reading how they were made if you want to make your implemention faster.

| Name | Value |
| ---- | ----- |
| MtuSize | 1492 |
| UdpHeaderSize | 28 |
| PublicKeySize | 294 |
| RequstChallengeSize | 64 |
| RespondingEncryptionKey | 128 |
| MaxNumberOfLocalAddresses | 10 |
| IdentityProofSize | 294 |
| ClientProofSize | 32 |
| DefaultProtocolVersion | 6 |
| NumberOfArrangedStreams | 32 |

## Minecraft

If you are here expecting this to be for minecraft then you're at the wrong place(but not now).

Only a few things are different in the protocol as far as i know(see below to know what to change in here)

The `MtuSize` would be 1400(but it changes depending on the network used by the client connecting to the server but you won't need to care about it as you will use that mtu for the client)

The `MaxNumberOfLocalAddresses` would be 20.

The `DefaultProtocolVersion` would be 11.

The `message` field inside UnconnectedPong would contain the server motd or whatever you call it and each part of info is separated by `;`.

The first part would be the edition where `MCPE` would be for the default minecraft edition and `MCEE` for education edition.

The second part would be the first line of the motd which can be anything.

The third part would be the game protocol version(not raknet).

The fourth part would be the current minecraft version(can also be * if you're doing some multiversion thing - optional).

The fifth part would be the number of online players.

The sixth part would be the number of maximum players(it's important since that's what the client will actually use to determine whether to join or not).

The seventh part would be the unqiue id of the server(can be the server guid).

The eighth part would be the world name.

The ninth part would be the the string version of the world gamemode.

The tenth part would be the numeric version of the world gamemode(optional).

The eleventh part would be the ipv4 server port(optional).

The twelfth part would be the ipv6 server port(optional).

## Packets

### Packet Identifiers

| Name | ID | Type |
| ---- | -- | ---- |
| UnconnectedPing | 0x01 | OFFLINE |
| UnconnectedPingOpenConnections | 0x02 | OFFLINE |
| UnconnectedPong | 0x1c | OFFLINE |
| ConnectedPing | 0x00 | ONLINE [from/to datagram] |
| ConnectedPong | 0x03 | ONLINE [from/to datagram] |
| OpenConnectionRequestOne | 0x05 | OFFLINE |
| OpenConnectionReplyOne | 0x06 | OFFLINE |
| OpenConnectionRequestTwo | 0x07 | OFFLINE |
| OpenConnectionReplyTwo | 0x08 | OFFLINE |
| ConnectionRequest | 0x09 | ONLINE [from/to datagram] |
| RemoteSystemRequiresPublicKey | 0x0a | OFFLINE |
| OurSystemRequiresSecurity | 0x0b | BOTH [?] |
| ConnectionAttemptFailed | 0x11 | OFFLINE |
| AlreadyConnected | 0x12 | OFFLINE |
| ConnectionRequestAccepted | 0x10 | ONLINE [from/to datagram] |
| NewIncomingConnection | 0x13 | ONLINE [from/to datagram] |
| DisconnectionNotification | 0x15 | ONLINE |
| ConnectionLost | 0x16 | BOTH [?] |
| IncompatibleProtocolVersion | 0x19 | ONLINE |

### RakNet Protocol Packet Send and Receive Sequence

#### Server

1. Wait for an UnconnectedPing packet from a client.
   - This is a request from the client to check if the server is available and responding.
   - Respond with an UnconnectedPong packet to let the client know that the server is available.
   
2. Wait for an OpenConnectionRequestOne packet from a client.
   - This is the first part of a handshake protocol to initiate a connection request with the server.
   - If the protocol version is correct, respond with an OpenConnectionReplyOne packet to acknowledge the connection request.
   - If the protocol version is incorrect, respond with an IncompatibleProtocolVersion packet to inform the client that the connection request is rejected.
   
3. Wait for an OpenConnectionRequestTwo packet from the client.
   - This is the second part of the handshake protocol to establish a connection with the server.
   - Respond with an OpenConnectionReplyTwo packet to let the client know that the connection is established.
   - Create a new connection for the client and save its connection information to handle future online packets.
   
4. Wait for datagrams from the client.
   - Handle the datagrams received from the client as required, whether they are AckedDatagrams, NackedDatagrams, require B and AS values, or are segmented packets.
   - After that you will receive inside the datagram received a list of packets that will be sent seperately down you will see them and understand
		- Wait for an ConnectionRequest packet from the client
			- Respond with an ConnectionRequestAccepted that is reliable.
		- Wait for an NewIncomingConneciton packet
			- Check if the server port is the same as the address of the client inside NewIncomingConnection then mark it as connected if so.
			- Response with a ConnectedPong packet that is unreliable.
		- Wait for a DisconnectNotification
			- Handle it as you want.
		- Wait for a ConnectedPing
			- Response with a ConnectedPong that is unreliable.

#### Client

1. Send an UnconnectedPing packet to the server.
   - This is a request to check if the server is available and responding.
   - Wait for an UnconnectedPong packet from the server to confirm that the server is available.

2. Send an OpenConnectionRequestOne packet to the server.
   - This is the first part of a handshake protocol to initiate a connection request with the server.
   - Wait for an OpenConnectionReplyOne packet from the server to acknowledge the connection request.
   - If an IncompatibleProtocolVersion packet is received instead of an OpenConnectionReplyOne packet, the connection request is rejected.

3. Send an OpenConnectionRequestTwo packet to the server.
   - This is the second part of the handshake protocol to establish a connection with the server.
   - Wait for an OpenConnectionReplyTwo packet from the server to confirm that the connection is established.
   - If the connection request is rejected, the client should start again from step 1.

4. Send datagrams to the server.
   - Handle the datagrams sent to the client, whether they are AckedDatagrams, NackedDatagrams, require B and AS values, or are segmented packets.
   - But before handling, at when you handle the OpenConnectionReplyTwo or anywhere related such as if the OpenConnectionReplyTwo got received, you need to do the following after you do what you want:
		- Send a ConnectionRequest packet to the server.
			- Wait for a ConnectionRequestAccepted packet from the server.
		- Send a NewIncomingConnection packet to the server.
			- Send a ConnectedPing packet that is unreliable.
		- Send a DisconnectNotification packet to the server. (if you want to disconnect)
		- Wait for ConnectedPong.
			- Response with ConnectedPing.

### UnconnectedPing

This packet is used to determine if a server is online or not.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| onlyReplyOnOpenConnections | bool | N/A | If set to true, the server will only send a reply if the client's connection to the server is currently open. This helps to prevent sending responses to clients that have closed their connections. The resulting message ID for the request would be `UnconnectedPingOpenConnections`. If set to false, then the identifier won't need to be change. |
| id | uint8 | N/A | Unique identifier of the packet |
| clientSendTime | uint64 | Big Endian | Client timestamp that can be used to calculate the latency |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| clientGuid | uint64 | Big Endian | Unique identifier of the client |

### UnconnectedPong

This packet is the response to an unconnected ping packet.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| serverSendTime | uint64 | Big Endian | Server timestamp that can be used to calculate the latency |
| serverGuid | uint64 | Big Endian | Unique identifier of the server |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| message | uint16-string | Big Endian | Response data typically used for server information |

> You can call the `message` whatever you want. For example you can call it `data` instead

### ConnectedPing

This packet is used to keep the connection alive between the client and the server.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| clientSendTime | uint64 | Big Endian | Client timestamp that can be used to calculate the latency |

### ConnectedPong

This packet is the response to a connected ping packet.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| clientSendTime | uint64 | Big Endian | Client timestamp from the ping |
| serverSendTime | uint64 | Big Endian | Server timestamp that can be used to calculate the latency |

### OpenConnectionRequestOne

This packet is used to initiate the handshake process between a client and a server.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| protocolVersion | uint8 | N/A | Protocol version supported by the client |
| mtuSize | pad-with-zero | N/A | Maximum transmission unit (MTU) size of the client |

> When using pad-with-zero, Add to the MTU size the current reading position plus 28 (UDP header size) for reading. For writing, Get the MTU size subtracted with the current buffer writing position (or its size) plus 28 (UDP header size) plus the current buffer size. To validate the packet buffer, check if its size is 28(udp header size) plus the current buffer size.

### OpenConnectionReplyOne

This packet is the response to an open connection request one packet.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| serverGuid | uint64 | Big Endian | Unique identifier for the server |
| serverHasSecurity | bool | N/A | Whether the server requires security or not |
| mtuSize | uint16 | Big Endian | Maximum transmission unit (MTU) size of the server |

### OpenConnectionReplyOne With Security

This packet is the response to an open connection request one packet with additional security information.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| serverGuid | uint64 | Big Endian | Unique identifier for the server |
| serverHasSecurity | bool | N/A | Whether the server requires security or not |
| hasCookie | bool | N/A | Whether the packet includes a cookie |
| cookie | uint32 | Big Endian | Cookie value |
| serverPublicKey | uint8[294] | N/A | Public key used for encryption |
| mtuSize | uint16 | Big Endian | Maximum transmission unit (MTU) size of the server |

### OpenConnectionRequestTwo

This packet is used to complete the handshake process between a client and a server.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| serverAddress | uint8[7-29] | N/A | Server IP address and port combo |
| mtuSize | uint16 | Big Endian | Maximum transmission unit (MTU) size of the client |
| clientGuid | uint64 | Big Endian | Unique identifier of the client |

### OpenConnectionRequestTwo If OpenConnectionReplyOne has security

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| cookie | uint32 | Big Endian | Cookie value |
| containsChallenge | bool | N/A | Whether the system requires handshake challenge |
| challenge | uint8[64] | N/A | The system handshake challenge bytes |
| serverAddress | uint8[7-29] | N/A | Server IP address and port combo |
| mtuSize | uint16 | Big Endian | Maximum transmission unit (MTU) size of the client |
| clientGuid | uint64 | Big Endian | Unique identifier of the client |

> Note: if the OpenConnectionReplyOne packet has security but this packet does not contain a challenge, the client should immediately send a RemoteSystemRequiresPublicKey packet to notify the server that there was no challenge in the OpenConnectionRequestTwo packet.

You can write your own way of calculating the connection outcome but this below is the standard way.

**Calculating ConnectionOutcome**:
- Get the client address.
- Use `bitwise and` to check if the checks below is needed using `contains address` and `contains guid` boolean values.
  - If the client is not currently connected or it's address is not found in the list, then the outcome is 1.
  - Otherwise, set it to 2.
- If the `clientGuid` is already associated with a client that has a different client address, set the connection state to 3.
- If the client address is already associated with a different `clientGuid`, set the connection state to 4.
- Otherwise, set the state to 0.

> Once you have calculated the `ConnectionOutcome`, You will need to check if it is equal to 1 then send the `OpenConnectionReplyTwo` packet.

> If the `ConnectionOutcome` is not 0, send the `AlreadyConnected` packet.

### OpenConnectionReplyTwo

This packet is the response to an open connection request two packet.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| serverGuid | uint64 | Big Endian | Unique identifier of the server |
| clientAddress | uint8[7-29] | N/A | Client IP address and port combo |
| mtuSize | uint16 | Big Endian | Maximum transmission unit (MTU) size of the server |
| requiresEncryption | bit | N/A | Whether the connection requires encryption or not |
| encryptionKey | uint8[128] | N/A | The encryption key of the client - it is only written or read if the `requiresEncryption` field is set to true. |

### ConnectionRequest

This packet is used to establish a connection between a client and a server with security enabled or disabled.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| clientGuid | uint64 | Big Endian | Unique identifier of the client |
| clientSendTime | uint64 | Big Endian | Timestamp of the client when it requested to be connected |
| doSecurity | bool | N/A | Whether the connection requires security or not |
| clientProof | uint8[32] | N/A | Proof of client authentication |
| doIdentity | bool | N/A | Whether the packet requires an identity proof |
| identityProof | uint8[294] | N/A | Proof of client identity |

> Note: If the identity proof is invalid and `doIdentity` is set to true, immediately send a `RemoteSystemRequiresPublicKey` packet with a type ID of `ClientIdentityIsInvalid`. If `doIdentity` is set to false and there is no identity proof, send a `RemoteSystemRequiresPublicKey` packet with a type ID of `ClientIdentityIsMissing`.

### RemoteSystemRequiresPublicKey

This packet is used to throw the errors related to public key requests for client authentication and identification.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| typeID | uint8 | N/A | Type of public key request |

#### RemoteSystemRequiresPublicKey Type IDs

| Name | ID |
| ---- | -- |
| ServerPublicKeyIsMissing | 0 |
| ClientIdentityIsMissing | 1 |
| ClientIdentityIsInvalid | 2 |

### OurSystemRequiresSecurity

This packet is sent when the server does not require security but it is still mandatory.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| clientAddress | uint8[7-29] | N/A | Client IP address and port combo |
| serverGuid | uint64 | Big Endian | Unique identifier of the server |

### ConnectionAttemptFailed

This packet is sent when the attempt count trying to join the server is higher than a certain amount (depends on your implementation) or the client does not contain an assigned address; this is what you check and send if the requirements are met before sending the `OpenConnectionRequestOne` packet.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |

### AlreadyConnected

This packet is sent when the client is already connected.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| clientGuid | uint64 | Big Endian | Unique identifier of the client |

### ConnectionRequestAccepted

This packet is the response to a connection request.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| clientAddress | uint8[7-29] | N/A | Client IP address and port combo |
| clientIndex | uint16 | Big Endian | Unique identifier of the client |
| serverNetAddresses | address[10] | N/A | Server local network addresses |
| clientSendTime | uint64 | Big Endian | Timestamp of the client |
| serverSendTime | uint64 | Big Endian | Timestamp of the server |

### NewIncomingConnection

This packet is sent from the client to the server .

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| serverAddress | uint8[7-29] | N/A | Server IP address and port combo |
| clientNetAddresses | address[10] | N/A | Client local machine addresses |
| clientSendTime | uint64 | Big Endian | Timestamp of the client |
| serverSendTime | uint64 | Big Endian | Timestamp of the server |

Send the `ConnectedPing` packet or `ConnectedPong` packet depending on the side of the action.

### DisconnectionNotification

This packet is sent when a client disconnects from the server.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |

### ConnectionLost

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| clientGuid | uint64 | Big Endian | Unique identifier of the client |
| clientAddress | uint8[7-29] | N/A | Client IP address and port combo |

### IncompatibleProtocolVersion

This packet is sent when a client attempts to connect to a server with an incompatible protocol version.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier of the packet |
| protocolVersion | uint8 | N/A | Protocol version supported by the server |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| serverGuid | uint64 | Big Endian | Unique identifier of the server |

### Datagram

This packet is used for sending and receiving data between clients and the server after connecting which happens after the end of the handshake. It can be one of three types: ValidDatagram, AckedDatagram, or NackedDatagram.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| isValid | bit | N/A | Always true |
| isAck | bit | N/A | If true, the packet is an AckedDatagram |
| isNack | bit | N/A | If true, the packet is a NackedDatagram |

> If `isAck` and `isNack` are both false, the packet is a ValidDatagram.

### AckedDatagram

This packet is a response to a ValidDatagram indicating that the server has received the data.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| requiresBAndAS | bit | N/A | If true, the packet includes B and AS values |
| B | float | Big Endian | Not used |
| AS | float | Big Endian | Data arrival rate |
| ranges | RangeList | N/A | See `RangeList` - nodes where the range is the acknowledged datagram range numbers |

### NackedDatagram

This packet is a response to a ValidDatagram indicating that the server has not received all of the expected data.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| ranges | RangeList | N/A | See `RangeList` - nodes where range is the non-acknowledged datagram range numbers |

### Range

This structure is used to represent the ranges of AckedDatagrams and the missing ranges of NackedDatagrams.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| size | uint16 | Big Endian | Number of ranges in the array |
| isSingle | bool | N/A | If min is equals to max, then this is set to true |
| min | uint24 | Little Endian | Minimum value in the range|
| max | uint24 | Little Endian | Maximum value in the range - Is not wrote if is single |

#### RangeLList
You need a list of ranges which i previously mentioned as `ranges` which is the rangelist.
the range list contains the nodes which is the min index and max index for each datagram range number that needs to be inserted.

when inserting you should reduce the amount of nodes there is that is why min and max exists.

for example lets say you have an array of min and max nodes(where you must insert them as single since you won't be touching them manually):
`[
<min 0, max: 0>
<min: 2, max: 2>
<min: 3, max: 3>
<min: 5, max: 5>
<min: 6, max: 6>
<min: 8, max: 8>
<min: 8, max: 8> - there shouldn't be any repeats anyway but this is an example to show what it should do so it's here
<min: 8, max: 8>
<min: 10, max: 10>
]`

It shall turn into what is below(by your code and remember they must always be sorted):
`[
<min: 0, max: 0>
<min: 2, max: 3>
<min: 5, max: 6>
<min: 8, max: 8>
<min: 10, max: 10>
<
]`

### ValidDatagram

This packet is used for sending and receiving data between clients and the server.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| isPacketPair | bit | N/A | If true, the packet is one of two associated packets |
| isContinuousSend | bit | N/A | If true, the packet is a continuous send packet |
| requiresBAndAS | bit | N/A | If true, the packet includes B and AS values |
| rangeNumber | uint24 | Little Endian | The sequence number of the datagram |
| capsules | DatagramCapsule[] | N/A | Array of capsules in the packet |

You don't need to care about `isPacketPair`, `requiresBAndAS` and `isContinuousSend` unless you want a complete datagram packet implemention.

If you care then the `isContinuousSend` is always true.

**Finding `skipped ranges count` for the valid datagram to check if there was some packets missing**:

Firstly check if the current datagram `range number` is not equals to the `expected range number` which is 0 by default.

Then subtract `range number` by `expected range number` and if it is greater than `0x3e8` and force the `skipped range count` to be `0x3e8` if so. But before forcing it check if the value is greater than `0xc350` and if so then it's `nat related`.

After that the `expected range number` shall be the current `range number` plus 1.

For a simpler implemention you don't need to add those checks that was previously written.

Do your nack related things and checks then loop through the count in the reverse order and insert each index into the nack queue to be sent later.

**Checking for corrupt arrangment channels**:
(the valid datagram must be `Sequenced And Arranged (No ack receipt))

if `arrangmentChannel` is greater or equals to number of arranged streams available which is 2 ^ 5 then it is correupted(skip and do what is needed).

every valid datagram arrangement type of an array max value is the number of arranged streams and must not be greater or equals to.

**Finding hole count in received reliable datagrams**:
(the valid datagram must be `Reliable or in sequence` before proceeding further)

you will need to check hole count in valid datagrams that is `Reliable or in sequence` and the reason is to check for their order and it serves as a check if there was some kind of missing valid datagram received or wrong `rangeNumber` was used in sending or whatever else reason.

to find the hole count subtract the current received valid datagram's `rangeNumber` with the `receivedPacketsBaseIndex`.

You will need to do an array with it's max as 2048 or 0xf4240 and save the hole count as the key and true as value.
You can write it anyway you want but i'm writing the way i did it.

**Steps**:

1. if the hole count is 0 then it is a proper valid datagram so you can handle it normally (and what is done inside was already said).
2. if the hole count `bitwise AND` the maximum value of `uint24` that is `bitwise right shifted` by 1 is smaller than the array and also it exists in the array that the hole count was saved in then it is a duplicated packet (skip and do what is needed).
3. if the hole count is greater than the array size then it is.

Loop through each element while dropping it and increment the reliable base index each time until the array is emptied.

after that you can handle the valid datagrams normally(handle the arrangment before that).

### DatagramCapsule

This structure represents a capsule in a ValidDatagram.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| reliability | 3 bits | Big Endian | Type of reliability used |
| isSegmented | bit | N/A | If true, the packet is segmented |
| size | uint16 | Big Endian | Size of the buffer field down there in bits |
| reliableCapsuleIndex | uint24 | Little Endian | Index used for reliable packets (it means use reliability for this) |
| sequencedCapsuleIndex | uint24 | Little Endian | Index used for sequenced packets (it means use reliability for this) |
| arrangement | CapsuleArrangement | N/A | Arrangement of the capsule used for sequenced and arranged packets (it means use reliability for this) |
| segment | CapsuleSegment | N/A | Segment of the capsule used when capsule is segmented |
| buffer | Buffer | N/A | Buffer data containing the data wanted to be sent through networks |

### CapsuleArrangement

This structure represents the arrangement of a capsule in a ValidDatagram.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| arrangedCapsuleIndex | uint24 | Little Endian | Index of the arranged capsule |
| arrangementChannel | uint8 | N/A | The arrangement channel |

### CapsuleSegment

This structure represents the segmentation of a capsule in a ValidDatagram.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| size | uint32 | Big Endian | Size of the segment |
| id | uint16 | Big Endian | Unique identifier associated with the segment |
| index | uint32 | Big Endian | Index of the segment |

## Datagram related

### Reliability

Each datagram sent in RakNet is assigned a Reliability TypeID that specifies how the data should be handled by the protocol. The following table lists the available Reliability TypeIDs and their properties:

| Name | ID  | Is Reliable | Is Arranged | Is Sequenced | Characteristics/Features |
| ---- | --- | ----------- | ----------- | ------------ | ------------------------ |
| Unreliable              | 0   | No          | No         | No           | This Reliability TypeID sends datagrams without any guarantees that they will arrive at the destination. They are not guaranteed to be delivered in any specific order or at all |
| UnreliableSequenced      | 1   | No          | Yes        | Yes          | This Reliability TypeID sends datagrams without any guarantees that they will arrive at the destination but ensures that they are delivered in the sequence they were sent |
| Reliable                 | 2   | Yes         | No         | No           | This Reliability TypeID sends datagrams guaranteed to be delivered in the order they were sent. If a datagram is lost, RakNet will retransmit it until it is acknowledged by the receiver |
| ReliableArranged         | 3   | Yes         | Yes        | No           | This Reliability TypeID sends datagrams guaranteed to be delivered in the order they were sent. If a datagram is lost, RakNet will not retransmit it and all datagrams before it that have not been acknowledged |
| ReliableSequenced        | 4   | Yes         | Yes        | Yes          | This Reliability TypeID sends datagrams guaranteed to be delivered in the order they were sent and ensures that they are delivered sequentially |
| UnreliableWithAckReceipt | 5   | No          | No         | No           | This Reliability TypeID sends datagrams without any guarantees that they will arrive at the destination, but the receiver sends an acknowledgement receipt upon receipt of this datagram |
| ReliableWithAckReceipt   | 6   | Yes         | No         | No           | This Reliability TypeID sends datagrams guaranteed to be delivered in the order they were sent, and the receiver sends an acknowledgement receipt upon receipt of this datagram |
| ReliableArrangedWithAckReceipt | 7 | Yes   | Yes        | No           | This Reliability TypeID sends datagrams guaranteed to be delivered in the order they were sent but all datagrams before it that have not been acknowledged, and the receiver sends an acknowledgement receipt upon receipt of this datagram |

### Reliability definitions

Here you can find every reliability definition which is used in other places at the documentation.

1. Reliable - This is when the reliability is of any type that is reliable
2. Sequenced - This is when the reliability is both unreliable sequenced and reliable sequenced
3. Sequenced and arranged - This is when the reliability is `Sequenced` and reliable arranged and reliable arranged with ack recepit
4. Reliable or in sequence - This is when the reliability is reliable or reliable sequenced or reliable arranged

### Retransmission
RakNet uses selective repeat retransmission to ensure reliable delivery of datagrams. When a datagram is sent, it is assigned a sequence number. If a datagram is not acknowledged within a certain timeout period, RakNet will retransmit the datagram using the same sequence number. When the receiver receives a duplicate datagram with the same sequence number, it can discard it, since it has already acknowledged that sequence number.

### AckQueue / NackQueue
The AckQueue and NackQueue are used to keep track of which datagrams have been acknowledged and which have not. The AckQueue stores a list of datagram sequence numbers that have been successfully acknowledged, while the NackQueue stores a list of datagram sequence numbers that have not been acknowledged and need to be retransmitted. When a datagram is received with a sequence number that has already been acknowledged, it can be discarded.

### PacketPair
PacketPair is used to improve the efficiency of datagram retransmissions. When a datagram is acknowledged, RakNet sends the next datagram in the sequence as well. This allows the receiver to begin processing the next datagram immediately, reducing latency and improving throughput.

### ContinuousSend
ContinuousSend allows datagrams to be sent continuously without waiting for acknowledgement. This can improve performance in some cases, but can also lead to packet loss and retransmissions, since the sender does not wait for acknowledgement before sending the next datagram.

### Reassembly
Mechanism to reconstruct segmented datagrams. When a datagram is segmented, each segment is assigned a unique identifier. When the receiver receives a segment, it is buffered until all segments with the same identifier have been received(maximum number is `uint16` limit and if reached set the value back to zerk). Once all segments have been received, they are reassembled into the original datagram.

### Flow Control
Mechanism used to manage the rate of data transmission between sender and receiver. It ensures that the receiver can handle the incoming data at a pace it can process, preventing overwhelming or overflowing the receiver's buffer. Flow control helps maintain a balance between the sender's transmission speed and the receiver's processing capability, optimizing the overall efficiency and stability of the communication.

### Congestion Control
Congestion control is used to prevent network congestion by balancing data transmission rates. Techniques like TCP congestion control, packet dropping, rate limiting, traffic shaping, QoS, and load balancing are used. These techniques ensure reliable data delivery and efficient transmission in RakNet.

### Segment
Segmentation is used in RakNet to ensure data delivery if exceeds the number required by dividing large messages into smaller segments. These segments, with headers indicating position and size to ensure successful reassembly on the receiver's end if done correctly. This mechanism prevents data loss, manages large payloads, and guarantees reliable transmission in networked applications.

### Capsule Size
To determine the size of the capsule, you can follow these steps:
1. Increment the byte by 1 step to represent the reliability.
2. Increment the byte by 2 steps to represent the size of the buffer.
3. If the reliability is any type of reliable, increment the byte by 3 steps to represent the `reliableCapsuleIndex`.
4. If the reliability is sequenced, increment the byte by 3 steps to represent the `sequencedCapsuleIndex`.
5. If the reliability is sequenced and arranged increment the byte by 3 steps for the `arrangedCapsuleIndex`, and then by 1 step for the `arrangementChannel`.
6. If the capsule is segmented, increment the byte by 4 steps for the `size`, 2 steps for the `id`, and 4 steps for the `index` of the segment.

### UserPacketEnum
The UserPacketEnum ID is `0x86`, which marks the beginning of where you can start using your custom packet IDs.

What is recommended is to create a `PacketAggregator` considering you have a completed implementation then send and receive it.

You can put an id of your choice, like: `UserPacketEnumID` + your own id (it must not make the `UserPacketEnumID` surpass the `uint8 limit`)

For example: `UserPacketEnumID` + 22 = 0x9c

### Sending a Non-RakNet Packet
To send a non-RakNet packet, first determine if segmentation is needed by comparing if the capsule buffer size is greater than the MTU size minus 2, plus 3, plus 4 times 1 (for the datagram's data header byte length), and subtracting 11 if security is in use. Then, subtract the given value with the capsule size. If segmentation is necessary, segemnt the capsule's stream before adding it to the datagram queue for transmission. If no segmentation is required, add it directly to the queue. Remember, segmented packets must not be unreliable; if they are, convert them to reliable packets to guarantee successful and ordered delivery of all packet parts.

The way to segment it into parts:

get the capsule size and also the capsule buffer size then the size that you used to check if is greater than the capsule buffer size and define them.

to get the count of how many segments is needed to be made: subtract the capsule buffer size with 1 then divide it with the size that you used to check if is greater than the capsule buffer size then sum them with 1 (not two times) or you can ceil instead of subtracting 1 and adding 1 at the end

then define a `current segment index` outside the loop and is 0 by default

then do a loop that will use the count of how many segments then the segmentation process will start:

after that define a variable representing `start offset` and it is equals to `current segment index` multiplied by the size that you used to check if is greater than the capsule buffer size

after that define a variable representing `bytes to send` that is equals to capsule buffer size subtracted by `start offset`

check if `bytes to send` is greater than the size that you used to check if is greater than the capsule buffer size and if true set the `bytes to send` value to the size that you used to check...

after that create a new variable representing the `end offset`

check if the `bytes to send` is not the size that you used to check if is greater than the capsule buffer size then set the `end offset` to the capsule buffer size subtracted by the `current segment index` multiplied by the size that you used to check if is greater than the capsule buffer size

if the check fails then the `end offset` will be the `bytes to send`

after that slice the capsule buffer with the `start offset` and `end offset` then you can do the feilds in the capsule.

you will need an array that has the capsules that contains the segments that will then be sent in queue.

the segment id will be the `sender last segment id` property that will increment outside the loop every time.

## Resources
Here are a list of resources to help you better understand the RakNet protocol:

- <a href="https://github.com/facebookarchive/RakNet">Original RakNet</a>: Contains information on packets and all else but you will need to read and understand the code.
