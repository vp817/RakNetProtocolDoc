# RakNet Protocol Documentation

This is a raknet protocol documentation. It has everything needed to create a server/client(the default raknet not other types).

Not everything might be correct so you can pull request a fix to them if they really are wrong or you can create an issue.

## Data Types

| Type         | Size    | Note |
| ------------ | ------- | ---- |
| uint8        | 1 byte  |      |
| uint16       | 2 bytes |      |
| uint24       | 3 bytes | Unsigned 24-bit integer with a minimum value of 0 and a maximum value of (2^24)-1 |
| uint32       | 4 bytes |      |
| uint64       | 8 bytes |      |
| string       | variable | UTF-8 encoded string preceding with 2 bytes(uint16-little endian) its length |
| magic        | 16 bytes | An array of unsigned 8-bit integers with a specific sequence `[0x00, 0xFF, 0xFF, 0x00, 0xFE, 0xFE, 0xFE, 0xFE, 0xFD, 0xFD, 0xFD, 0xFD, 0x12, 0x34, 0x56, 0x78]` used to identify offline packets |
| zero-padding | variable | Null bytes used for padding with a size of your choice |
| bool         | 1 byte | Written or read as a single uint8, with a value of 0 or 1 (Zero represents false, and One represents true) |
| address      | 7-29 bytes | <table>
<thead> <th>Field/Condition</th> <th>Body</th> </thead> <tbody> <tr> <td>address version</td> <td>DataType: uint8.</br>Field will be referenced as "version" below</td> </tr> <tr> <td>version is 4</td> <td> <table> <thead> <th>Field</th> <th>Type</th> <th>Endianness</th> <th>Note</th> </thead> <tbody> <tr> <td>address</td> <td>uint32</td> <td>Big Endian</td> <td>Decoding: ~(address) then do endian stuff with it</td> </tr> <tr> <td>port</td> <td>uint16</td> <td>Big Endian</td> <td></td> </tr> </tbody> </table> </td> </tr> <tr> <td>version is 6</td> <td> <table> <thead> <th>Field</th> <th>Type</th> <th>Endianness</th> </thead> <tbody> <tr> <td>address family</td> <td>uint16</td> <td>Little Endian</td> </tr> <tr> <td>port</td> <td>uint16</td> <td>Big Endian</td> </tr> <tr> <td>flow info</td> <td>uint32</td> <td>Big Endian</td> </tr> <tr> <td>address</td> <td>uint8[16]</td> <td></td> </tr> <tr> <td>scope id</td> <td>uint32</td> <td>Big Endian</td> </tr> </tbody> </table> </td> </tr> </tbody> </table> |
| bit          | 1 bit   | 1/2/3 bits cannot be written inside the buffer if not 8bits which means 1 bit into buffer would be 0b10000000 written as uint8 |
| float        | 4 bytes | IEEE 754 single-precision floating-point number |

## Minecraft

This documentation isn't related to minecraft but it's possible to follow it while aiming to implement a minecraft-only raknet.

The things that changes in "General Constants":
- `MtuSize`: 1400
- `NumberOfLocalAddresses`: 20
- `DefaultProtocolVersion`: 11 (this isn't always the same and i will not be always updating it)
- datagram(not acks/nacks) sent packets are continous sent

Find the "motd" format from somewhere else and apply it as the "UnconnectedPong" `message` field.

> One way is to log it by creating a raknet client(not really since it will just read 1 packet and send 1 packet) and then getting the `message` field from a bedrock dedicated server.

## General Constants

| Name                      | Value     |
| ------------------------- | --------- |
| MtuSize                   | 1492      |
| UdpHeaderSize             | 28        |
| PublicKeySize             | 294       |
| RequstChallengeSize       | 64        |
| RespondingEncryptionKey   | 128       |
| MaxNumberOfLocalAddresses | 10        |
| IdentityProofSize         | 294       |
| ClientProofSize           | 32        |
| DefaultProtocolVersion    | 6         |
| NumberOfOrderedStreams    | 32(2 ^ 5) |

## Packets

### Identifiers

| Name                           | ID   | Type    |
| ------------------------------ | ---- | ------- |
| UnconnectedPing                | 0x01 | OFFLINE |
| UnconnectedPingOpenConnections | 0x02 | OFFLINE |
| UnconnectedPong                | 0x1c | OFFLINE |
| ConnectedPing                  | 0x00 | ONLINE  |
| ConnectedPong                  | 0x03 | ONLINE  |
| OpenConnectionRequest1         | 0x05 | OFFLINE |
| OpenConnectionReply1           | 0x06 | OFFLINE |
| OpenConnectionRequest2         | 0x07 | OFFLINE |
| OpenConnectionReply2           | 0x08 | OFFLINE |
| ConnectionRequest              | 0x09 | ONLINE  |
| RemoteSystemRequiresPublicKey  | 0x0a | ONLINE  |
| OurSystemRequiresSecurity      | 0x0b | OFFLINE |
| ConnectionAttemptFailed        | 0x11 | BOTH    |
| AlreadyConnected               | 0x12 | OFFLINE |
| ConnectionRequestAccepted      | 0x10 | ONLINE  |
| NewIncomingConnection          | 0x13 | ONLINE  |
| DisconnectionNotification      | 0x15 | BOTH    |
| ConnectionLost                 | 0x16 | BOTH    |
| IncompatibleProtocolVersion    | 0x19 | OFFLINE |

### Send and Receive Sequence

The first packets before a connection(not the socket connection) was craeted(still on unconnected packets) are the packets of the offline type, which means that those are the packets that is handled outside of datagrams(and the opposite holds true).

There isn't any info on why this and that below but the packet name says all that is needed to be said.

And the "Datagrams" to be handled and sent is all possible datagram types, where the `ValidDatagram` holds the online packets.

- Client: UnconnectedPing
   - Server: UnconnectedPong
- Client: OpenConnectionRequest1
   - Server: OpenConnectionReply1 / IncompatibleProtocolVersion
- Client: OpenConnectionRequest2
   - Server: OpenConnectionReply2 & Create connection.
- Client: Datagrams [if connection exists & is server]
   - Server: Handle accordingly.

### UnconnectedPing / UnconnectedPingOpenConnections

This packet is used to determine if a server is online or not.

For unconnected ping open connections: the server will only send a reply if the client's connection to the server is currently open. This helps to prevent sending responses to clients that have closed their connections.

| Field            | Type             | Endianness |
| ---------------- | ---------------- | ---------- |
| id               | uint8            | N/A        |
| client send time | uint64           | Big Endian |
| magic            | uint8[16]{magic} | N/A        |
| client guid      | uint64           | Big Endian |

### UnconnectedPong

This packet is the response to an unconnected ping packet.

| Field            | Type             | Endianness | Note |
| ---------------- | ---------------- | ---------- | ---- |
| id               | uint8            | N/A        |      |
| client send time | uint64           | Big Endian |      |
| server guid      | uint64           | Big Endian |      |
| magic            | uint8[16]{magic} | N/A        |      |
| message          | string           | Big Endian | Response data mostly used for server information. |

### ConnectedPing

This packet is used to keep the connection alive between the client and the server.

| Field            | Type   | Endianness |
| ---------------- | ------ | ---------- |
| id               | uint8  | N/A        |
| client send time | uint64 | Big Endian |

### ConnectedPong

This packet is the response to a connected ping packet.

| Field            | Type   | Endianness |
| ---------------- | ------ | ---------- |
| id               | uint8  | N/A        |
| client send time | uint64 | Big Endian |
| server send time | uint64 | Big Endian |

### OpenConnectionRequest1

This packet is used to initiate the handshake process between a client and a server.

| Field            | Type             | Endianness | Note |
| ---------------- | ---------------- | ---------- | ---- |
| id               | uint8            | N/A        |      |
| magic            | uint8[16]{magic} | N/A        |      |
| protocol version | uint8            | N/A        | Protocol version supported by the client |
| mtu size         | zero-padding     | N/A        |      |

> `mtu size` value when using zero-padding:
```
reading from stream:
   - mtu size = (zero-padding + reading position) + UDP Header Size
writing into stream:
   - mtu size = (zero-padding - writing position) - UDP Header Size
```

### OpenConnectionReply1

This packet is the response to an open connection request one packet.

| Field               | Type             | Endianness |
| ------------------- | ---------------- | ---------- |
| id                  | uint8            | N/A        |
| magic               | uint8[16]{magic} | N/A        |
| server guid         | uint64           | Big Endian |
| server has security | bool             | N/A        |
|   | &nbsp;&nbsp;&nbsp;**True & Libcat:**</br><table> <thead> <tr> <th>Field</th> <th>Type</th> <th>Endianness</th> </tr> </thead> <tbody> <tr> <td>hasCookie</td> <td>bool</td> <td>N/A</td> </tr> <tr> <td>cookie</td> <td>uint32</td> <td>Big Endian</td> </tr> <tr> <td>serverPublicKey</td> <td>uint8[294]</td> <td>N/A</td> </tr> </tbody> </table></br>&nbsp;&nbsp;&nbsp;&nbsp;**True & Nothing:**</br><table> <thead> <tr> <th>Field</th> <th>Type</th> <th>Endianness</th> </tr> </thead> <tbody> <tr> <td>cookie</td> <td>uint32</td> <td>Big Endian</td> </tr> </tbody> </table> |   |
| mtu size            | uint16           | Big Endian |

The `server has security` shall have a global variable that specifies if the server has security for later usage(if libcat).

### OpenConnectionRequest2

This packet is used to complete the handshake process between a client and a server.

| Field          | Type             | Endianness |
| -------------- | ---------------- | ---------- |
| id             | uint8            | N/A        |
| magic          | uint8[16]{magic} | N/A        |
| server address | uint8[7-29]      | N/A        |
| mtu size       | uint16           | Big Endian |
| client guid    | uint64           | Big Endian |

### OpenConnectionRequest2 If server has security & libcat

| Field              | Type             | Endianness | Note |
| ------------------ | ---------------- | ---------- | ---- |
| id                 | uint8            | N/A        |      |
| magic              | uint8[16]{magic} | N/A        |      |
| cookie             | uint32           | Big Endian |      |
| contains challenge | bool             | N/A        | Whether the system requires handshake challenge |
| challenge          | uint8[64]        | N/A        | The system handshake challenge bytes |
| server address     | uint8[7-29]      | N/A        |      |
| mtu size           | uint16           | Big Endian |      |
| client guid        | uint64           | Big Endian |      |

> if the server has security but this packet does not contain a challenge, then the client should immediately send a RemoteSystemRequiresPublicKey packet to notify the server that there was no challenge in the packet.

**ConnectionOutcome**:

If the `client guid` of the client address does not exists in list then you may mark this connection as a new connnection. if both already exists(even if address isn't same but guid is already used or vice versa) then it shall not connect.

If it can connect then the sent packet would be the `OpenConnectionReply2` packet.

If it cannot connect then the sent packet would be the `AlreadyConnected` packet.

### OpenConnectionReply2

This packet is the response to an open connection request two packet.

| Field               | Type             | Endianness | Note |
| ------------------- | ---------------- | ---------- | ---- |
| id                  | uint8            | N/A        |      |
| magic               | uint8[16]{magic} | N/A        |      |
| server guid         | uint64           | Big Endian |      |
| client address      | uint8[7-29]      | N/A        |      |
| mtu size            | uint16           | Big Endian |      |
| requires encryption | bit              | N/A        |      |
| encryption key      | uint8[128]       | N/A        | The encryption key of the client - it is only used if the `requiresEncryption` field is set to true. |

### ConnectionRequest

This packet is used to establish a connection between a client and a server with security enabled or disabled.

| Field            | Type       | Endianness | Note |
| ---------------- | ---------- | ---------- | ---- |
| id               | uint8      | N/A        |      |
| client guid      | uint64     | Big Endian |      |
| client send time | uint64     | Big Endian |      |
| do security      | bool       | N/A        |      |
| client proof     | uint8[32]  | N/A        | Proof of client authentication |
| do identity      | bool       | N/A        |      |
| identity proof   | uint8[294] | N/A        | Proof of client identity |

> If the `identity proof` is invalid and `do identity` is set to true, immediately send a `RemoteSystemRequiresPublicKey` packet with a type ID of `ClientIdentityIsInvalid`. If set to false and there is no `identity proof`, send a `RemoteSystemRequiresPublicKey` packet with a type ID of `ClientIdentityIsMissing`.

### RemoteSystemRequiresPublicKey

This packet is used to throw the errors related to public key requests for client authentication and identification.

| Field   | Type  | Endianness |
| ------- | ----- | ---------- |
| id      | uint8 | N/A        |
| type id | uint8 | N/A        |

#### Type Ids:

| Name                     | ID |
| ------------------------ | -- |
| ServerPublicKeyIsMissing | 0  |
| ClientIdentityIsMissing  | 1  |
| ClientIdentityIsInvalid  | 2  |

### OurSystemRequiresSecurity

This packet is sent when the server does not need security (libcat) but it is still required.

| Field          | Type        | Endianness |
| -------------- | ----------- | ---------- |
| id             | uint8       | N/A        |
| client address | uint8[7-29] |            |
| server guid    | uint64      | Big Endian |

### ConnectionAttemptFailed

This packet is sent when the attempt count trying to join the server is higher than a certain amount (depends on your implementation) or the client does not contain an assigned address; this is what you check and send if the requirements are met before sending the `OpenConnectionRequest1` packet.

| Field | Type  | Endianness |
| ----- | ----- | ---------- |
| id    | uint8 | N/A        |

### AlreadyConnected

This packet is sent when the client is already connected.

| Field       | Type             | Endianness |
| ----------- | ---------------- | ---------- |
| id          | uint8            | N/A        |
| magic       | uint8[16]{magic} | N/A        |
| client guid | uint64           | Big Endian |

### ConnectionRequestAccepted

This packet is the response to a connection request.

| Field                | Type        | Endianness | Note |
| -------------------- | ----------- | ---------- | ---- |
| id                   | uint8       | N/A        |      |
| client address       | uint8[7-29] | N/A        |      |
| client index         | uint16      | Big Endian | Current client index in list |
| server net addresses | address[10] | N/A        | Server local network addresses |
| client send time     | uint64      | Big Endian |      |
| server send time     | uint64      | Big Endian |      |

### NewIncomingConnection

This packet is sent from the client to the server .

| Field                | Type        | Endianness | Note |
| -------------------- | ----------- | ---------- | ---- |
| id                   | uint8       | N/A        |      |
| server address       | uint8[7-29] | N/A        |      |
| client net addresses | address[10] | N/A        | Client local network addresses |
| client send time     | uint64      | Big Endian |      |
| server send time     | uint64      | Big Endian |      |

Send the `ConnectedPing` packet or `ConnectedPong` packet depending on the side of the action.

### DisconnectionNotification

This packet is sent when a client disconnects from the server.

| Field | Type  | Endianness |
| ----- | ----- | ---------- |
| id    | uint8 | N/A        |

### ConnectionLost

| Field          | Type        | Endianness |
| -------------- | ----------- | ---------- |
| id             | uint8       | N/A        |
| client guid    | uint64      | Big Endian |
| client address | uint8[7-29] | N/A        |

### IncompatibleProtocolVersion

This packet is sent when a client attempts to connect to a server with an incompatible protocol version.

| Field            | Type             | Endianness | Note |
| ---------------- | ---------------- | ---------- | ---- |
| id               | uint8            | N/A        |      |
| protocol version | uint8            | N/A        | Protocol version supported by the server |
| magic            | uint8[16]{magic} | N/A        |      |
| server guid      | uint64           | Big Endian | Unique identifier of the server |

### Datagram

#### Dict

1. nann datagram means a datagram that is not ack nor nack(to simplify things in this doc).

#### Reliability

Each nann datagram sent is assigned a reliability that specifies how the data should be handled by the protocol. The following table lists the available reliability ids and their properties:

| Name                           | ID  | Is Reliable | Is Ordered | Is Sequenced |
| ------------------------------ | --- | ----------- | ---------- | ------------ |
| Unreliable                     | 0   | No          | No         | No           |
| UnreliableSequenced            | 1   | No          | Yes        | Yes          |
| Reliable                       | 2   | Yes         | No         | No           |
| ReliableOrdered                | 3   | Yes         | Yes        | No           |
| ReliableSequenced              | 4   | Yes         | Yes        | Yes          |
| UnreliableWithAckReceipt       | 5   | No          | No         | No           |
| ReliableWithAckReceipt         | 6   | Yes         | No         | No           |
| ReliableOrderedWithAckReceipt  | 7   | Yes         | Yes        | No           |

#### Reliability definitions

1. Reliable - This is when the reliability is of any type that is reliable
2. Sequenced - This is when the reliability is both unreliable sequenced and reliable sequenced
3. Sequenced and ordered - This is when the reliability is `Sequenced` and reliable ordered and reliable ordered with ack recepit
4. Reliable or in sequence - This is when the reliability is reliable or reliable sequenced or reliable ordered

#### Set of things required in your implemention

- Retransmission: retransmit a datagram if not acknowledged.
- Reassembly: reconstruct split packets into a valid normal packet.

Every datagram is and must be valid.

| Field    | Type | Endianness | Body |
| -------- | ---- | ---------- | ---- |
| is valid | bit  | N/A        | <table> <thead> <tr> <th>Field</th> <th>Type</th> <th>Endianness</th> <th>Note</th> </tr> </thead> <tbody> <tr> <td>is packet pair</td> <td>bit</td> <td>N/A</td> <td></td> </tr> <tr> <td>is continuous send</td> <td>bit</td> <td>N/A</td> <td></td> </tr> <tr> <td>requires B and AS</td> <td>bit</td> <td>N/A</td> <td></td> </tr> <tr> <td>range number</td> <td>uint24</td> <td>Little Endian</td> <td>The sequence number of the datagram</td> </tr> <tr> <td>capsules</td> <td>DatagramCapsule[]</td> <td>N/A</td> <td>Array of capsules in the datagram</td> </tr> </tbody> </table> |
| is ack   | bit  | N/A        | <table> <thead> <tr> <th>Field</th> <th>Type</th> <th>Endianness</th> <th>Note</th> </tr> </thead> <tbody> <tr> <td>requires B and AS</td> <td>bit</td> <td>N/A</td> <td>If true, the packet includes B and AS values</td> </tr> <tr> <td>B</td> <td>float</td> <td>Big Endian</td> <td>Not used(Bandwidth)</td> </tr> <tr> <td>AS</td> <td>float</td> <td>Big Endian</td> <td>Data arrival rate</td> </tr> <tr> <td>ranges</td> <td>RangeList</td> <td>N/A</td> <td>Acknowledged datagrams range numbers</td> </tr> </tbody> </table> |
| is nack  | bit  | N/A        | <table> <thead> <tr> <th>Field</th> <th>Type</th> <th>Endianness</th> <th>Note</th> </tr> </thead> <tbody> <tr> <td>ranges</td> <td>RangeList</td> <td>N/A</td> <td>Non-acknowledged datagrams range numbers</td> </tr> </tbody> </table> |

nann datagram header size: `2 + 3 + 4 * 1`

> Note: in a nann datagram every new range number must be removed from nack list/queue since the datagram was received.

The "finding" this and that below is all for the nann datagram.

**Finding `skipped ranges count` for the datagram to check if there was some packets missing**:

Psuedo code:

```py
global:`expected range number` := 0
method:`skipped msg count` := 0

if current datagram:`range number` is not equals to global:`expected range number` then
   method:`skipped msg count` := current datagram:`range number` - global:`expected range number`
   if method:`skipped msg count` is greater than 1000 then
      if method:`skipped msg count` is greater than 50000 then
         # nat related and the check can be removed if not dealing with that
      end
      method:`skipped msg count` := 1000
   end
end

function:`insert into ack list/queue`(`range number`: current datagram:`range number`)
global:`expected range number` := current datagram:`range number` + 1

if method:`skipped msg count` is greater than 0 then
   for tmp:`skipped msg offset` := method:`skipped msg count`; tmp:`skipped msg offset` is greater than 0; tmp:`skipped msg offset` -= 1 then
      function:`insert into nack list/queue`(`range number`: current datagram:`range number` - tmp:`skipped msg offset`)
   end
end
```

**Checking for corrupt ordering channels**:
(the datagram must be `Sequenced And Ordered (No ack receipt)`)

if `ordering channel` is greater or equals to the `NumberOfOrderedStreams` available then it is correupted(skip and do what is needed).

every datagram ordering type of an array max value is the `NumberOfOrderedStreams` and must not be greater or equals to.

**Finding hole in received reliable datagrams**:
(the datagram must be `Reliable or in sequence`)

it's basically an index order validator where the current reliable index is checked with the last reliable index and the hole is whats in between(over simplified)
and if whats in between is more than than 0 then the capsule/datgram/capsule is skipped.

### Range

This structure is used to represent the ranges of AckedDatagrams and the missing ranges of NackedDatagrams.

| Field     | Type   | Endianness | Note |
| --------- | ------ | ---------- | ---- |
| is single | bool   | N/A        | If min == max, then this is set to true |
| min       | uint24 | Little Endian | Minimum value in the range|
| max       | uint24 | Little Endian | Maximum value in the range - Is not wrote if is single |

#### RangeList

The range list contains an array of nodes of the min index and the max index for each datagram range number that needs to be inserted.

| Field     | Type    | Endianness | Note |
| --------- | ------- | ---------- | ---- |
| size      | uint16  | Big Endian | Nodes count |
| nodes     | Range[] | N/A        |      |

when inserting you should reduce the amount of nodes there is that is why min and max exists.

for example lets say you have an array of min and max nodes where each of them is single:

```
[
   <min 0, max: 0>
   <min: 2, max: 2>
   <min: 3, max: 3>
   <min: 5, max: 5>
   <min: 6, max: 6>
   <min: 8, max: 8>
   <min: 8, max: 8> - there shouldn't be any duplicates anyway but this is an example to show what it should do so it's here
   <min: 8, max: 8>
   <min: 10, max: 10>
   <min: 12, max: 12>
   <min: 13, max: 13>
   <min: 14, max: 14>
   <min: 15, max: 15>
]
```

It shall turn into what is below(remember they must always be sorted):

```
[
   <min: 0, max: 0>
   <min: 2, max: 3>
   <min: 5, max: 6>
   <min: 8, max: 8>
   <min: 10, max: 10>
   <min: 12, max: 15>
]
```

### DatagramCapsule

This structure represents a capsule in a ValidDatagram.

| Field                   | Type    | Endianness    | Note |
| ----------------------- | ------- | ------------- | ---- |
| reliability             | 3 bits  | Big Endian    |      |
| is split                | bit     | N/A           | If true, the packet is a split packet |
| size                    | uint16  | Big Endian    | Size of the buffer field down there in bits |
| reliable capsule index  | uint24  | Little Endian | Index used for reliable packets (it means use reliability for this) |
| sequenced capsule index | uint24  | Little Endian | Index used for sequenced packets (it means use reliability for this) |
| order                   | <table> <thead> <tr> <th>Field</th> <th>Type</th> <th>Endianness</th> </tr> </thead> <tbody> <tr> <td>ordering index</td> <td>uint24</td> <td>Little Endian</td> </tr> <tr> <td>ordering channel</td> <td>uint8</td> <td>N/A</td> </tr> </tbody> </table> |  | Order of the capsule used for sequenced and ordered packets (it means use reliability for this) |
| split packet            | <table> <thead> <tr> <th>Field</th> <th>Type</th> <th>Endianness</th> <th>Note</th> </tr> </thead> <tbody> <tr> <td>size</td> <td>uint32</td> <td>Big Endian</td> <td>Size of the packet</td> </tr> <tr> <td>id</td> <td>uint16</td> <td>Big Endian</td> <td>Id of the split</td> </tr> <tr> <td>index</td> <td>uint32</td> <td>Big Endian</td> <td>Index of the current split</td> </tr> </tbody> </table> |  | Segment of the capsule used when capsule is segmented |
| buffer                  | uint8[] | N/A           |      |

## Datagram related

### Capsule Size
To determine the size of the capsule, you can follow these steps:
1. Increment the byte by 1 to represent the reliability.
2. Increment the byte by 2 to represent the size of the buffer.
3. If the reliability is any type of reliable, increment the byte by 3 to represent the `reliable capsule index`.
4. If the reliability is sequenced, increment the byte by 3 to represent the `sequenced capsule index`.
5. If the reliability is sequenced and ordered increment the byte by 3 for the `ordering index`, and then by 1 step for the `ordering channel`.
6. If the capsule is segmented, increment the byte by 4 for the `size`, 2 for the `id`, and 4 for the `index` of the segment.

### UserPacketEnum
The UserPacketEnum id is `0x86`, which marks the beginning of where you can start using your custom packet IDs.

What is recommended is to create a `PacketAggregator` considering you have a completed implementation then send and receive it.

You can put an id of your choice, like: `UserPacketEnumId` + your own id (it must not make the `UserPacketEnumId` surpass the `uint8 limit`)

For example: `UserPacketEnumId` + 22 = 0x9c

### Sending a Non-RakNet Packet

Psuedo code:

This follows the default raknet splitting algorithm(you can write your own if you want).

```py
global/method(depends on your choice):`max payload size` := client mtu size - nann datagram header size
if libcat & server has security then
   global/method:`max payload size` -= libcat:auth:enc:OVERHEAD_BYTES or 11
end

method:`max block size` := global/method:`max payload size` - capsule:`size` # the size is not the buffer

method:`packet buffer len` := len(capsule:`buffer`)

method:`split packet` := method:`packet buffer len` is greater than method:`max block size`

# split packets must not be unreliable; if they are, convert them to reliable packets so they are successfully sent and each part is in order to reassemble.
# once the converting is done you can do stuff like sequenced index and reliable index and ordering stuff and so on.
# then you can split

if method:`split packet` then
   method:`split packet count` := int(math.ceil(method:`packet buffer len` / method:`max block size`))

   for tmp:`i` := 0; tmp:`i` is lower than method:`split packet count`; tmp:`i` += 1 then
      tmp:`start offset` := tmp:`i` * method:`max block size`
      tmp:`bytes to send` := method:`packet buffer len` - tmp:`start offset`
      if tmp:`bytes to send` > method:`max block size` the
         tmp:`bytes to send` = method:`max block size`
      end
      tmp:`end offset` := tmp:`bytes to send`
      if tmp:`bytes to send` is not equals to method:`max block size` then
         tmp:`end offset` = method:`packet buffer len` - tmp:`i` * method:`max block size`
      end
      # construct split packet capsule and the buffer inside would be the buffer that was going to be sent sliced with start offset and end offset of the values defined before.
   end
end
```

## Resources
Here are a list of resources to help you better understand the RakNet protocol:

- <a href="https://github.com/facebookarchive/RakNet">Original RakNet</a>: Contains information on packets and all else but you will need to read and understand the code.
