# RakNet Protocol Documentation

This is a raknet protocol documentation. It has everything needed to create a server/client(the default raknet not other types).

If anything was incorrect or misspelled, you can pull request a fix to them if they really are wrong.

## DataTypes

| Type         | Size       | Note |
| ------------ | ---------- | ---- |
| uint8        | 1 byte     |      |
| uint16       | 2 bytes    |      |
| uint24       | 3 bytes    | Unsigned 24-bit integer with a minimum value of 0 and a maximum value of (2^24)-1 |
| uint32       | 4 bytes    |      |
| uint64       | 8 bytes    |      |
| string       | variable   | UTF-8 encoded string preceding with 2 bytes(uint16-little endian) its length |
| magic        | 16 bytes   | A uint8 array with a specific sequence `[0x00, 0xFF, 0xFF, 0x00, 0xFE, 0xFE, 0xFE, 0xFE, 0xFD, 0xFD, 0xFD, 0xFD, 0x12, 0x34, 0x56, 0x78]` used to identify offline packets |
| zero-padding | variable   | A single zero value uint8 recorded in sequence until the required size. |
| bool         | 1 byte     | Written or read as a single uint8, with a value of 0 or 1 (0 represents false, and 1 represents true). |
| address      | 7-29 bytes | See below(Address DataType). |
| bit          | 1 bit   | a bit can be 1 or 0 where it is written as uint8 extended by 0s until 8 bits (the bits follow the MSb order) |
| float        | 4 bytes | IEEE 754 single-precision floating-point number |

### Address DataType

| Field   | Type  |
| ------- | ----- |
| version | uint8 |
| .....   | ....1 |

**`....1` shall be recorded as:**

#### If version is equals to 4:

| Field   | Type   | Endianness |
| ------- | ------ | ---------- |
| address | uint32 | Big Endian |
| port    | uint16 | Big Endian |

an ipv4 string address always has 4 parts "part1(MSB).part2.part3.part4(LSB)".

converting an ipv4 string to an ipv4 address:
each part shall be recorded from the MSB to the LSB then the address would be ~(the result).

converting an ipv4 address into an ipv4 string:
the value to extract each part into its place in the string from the MSB to the LSB would be ~address.

#### If version is equals to 6:

| Field          | Type      | Endianness    |
| -------------- | --------- | ------------- |
| address family | uint16    | Little Endian |
| port           | uint16    | Big Endian    |
| flow info      | uint32    | Big Endian    |
| address        | uint8[16] | N/A           |
| scope id       | uint32    | Big Endian    |

## Minecraft

This documentation isn't related to minecraft but it's possible to follow it while aiming to implement a minecraft-only raknet.

The things that changes in "General Constants":
- `MaximumMtuSize` is 1400.
- `NumberOfLocalAddresses`: 20
- `DefaultProtocolVersion`: 11 (this isn't always the same but the protocol does not seem to change)

Datagram(not acks/nacks) sent packets are continous sent.

Find the "motd" format from somewhere else and apply it as the "UnconnectedPong" `message` field.

> One way is to log it by sending the unconnected ping packet to a bds(bedrock dedicated server) and then logging the `message` field from a bds.
> It is also possible to log the protocol version of the minecraft client through the incompatible protocol version packet if it was ever updated.

## General Constants

| Name                      | Value     |
| ------------------------- | --------- |
| MaximumMtuSize            | 1492      |
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

Below, there is no information on why this and that, but the packet name says all that is needed to be said.

The "Datagrams" to be handled and sent is all possible datagram types, where the `valid dgram` holds the online packets.

- Client: UnconnectedPing
   - Server: UnconnectedPong
- Client: OpenConnectionRequest1
   - Server: OpenConnectionReply1 / IncompatibleProtocolVersion
- Client: OpenConnectionRequest2
   - Server: OpenConnectionReply2 & Create connection.
- Client: Datagrams
   - Server: Handle accordingly.

### UnconnectedPing / UnconnectedPingOpenConnections

This packet is used to determine if a server is online or not.

For unconnected ping open connections: the server will only send a reply if the client's connection to the server is currently open. This helps to prevent sending responses to clients that have closed their connections.

| Field            | Type             | Endianness |
| ---------------- | ---------------- | ---------- |
| id               | uint8            | N/A        |
| client send time | uint64           | Big Endian |
| magic            | magic            | N/A        |
| client guid      | uint64           | Big Endian |

### UnconnectedPong

This packet is the response to an unconnected ping packet.

| Field            | Type             | Endianness | Note |
| ---------------- | ---------------- | ---------- | ---- |
| id               | uint8            | N/A        |      |
| client send time | uint64           | Big Endian |      |
| server guid      | uint64           | Big Endian |      |
| magic            | magic            | N/A        |      |
| message          | string           | Big Endian | Response data usually used for server information. |

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
| magic            | magic            | N/A        |      |
| protocol version | uint8            | N/A        | Protocol version supported by the client |
| mtu size         | zero-padding     | N/A        |      |


The `mtu size` value when using zero-padding:
```
reading from buffer:
   - mtu size = zero-padding + reading position + UDP Header Size
writing into buffer:
   - mtu size = (mtu size - writing position) - UDP Header Size
```

**client mtu size discovery:**

in the original raknet the mtu sizes is an integer array with the size of 3 (it would be refered as number of mtu sizes below).

the mtu sizes are {`MaximumMtuSize`, 1200, 576}.

Maximum mtu size chart:
```
1500. The largest Ethernet packet size. This is the typical setting for non-PPPoE, non-VPN connections. The default value for NETGEAR routers, adapters and switches.
1492. The size PPPoE prefers.
1472. Maximum size to use for pinging. (Bigger packets are fragmented.)
1468. The size DHCP prefers.
1460. Usable by AOL if you don't have large email attachments, etc.
1430. The size VPN and PPTP prefer.
1400. Maximum size for AOL DSL.
576. Typical value to connect to dial-up ISPs.
```

The one used in raknet by default is 1492(known as the size PPPoE prefers), so the `MaximumMtuSize` is equals to said value. But it can always be changed through user preferences.

the UDP Header Size is always the size of an ipv4 udp header + actual udp header size in the original raknet.
Which can be found by calculating the udp ipv4 header size(`(32 (src addr) + 32 (dst addr) + 8 (zeroes) + 8 (protocol) + 16 (udp len) + 16 (src port) + 16 (dest port) + 16 (len) + 16 (checksum)) / 8`) which is equals to 20 then plus the the udp header size (`(16 (src port) + 16 (dest port) + 16 (len) + 16 (checksum)) / 8`) which would be 8, so 20 + 8 would be 28.

the 576 would be known as the minimum of an ipv4 udp packet and the 1200 may be for the ipv6 udp (usually its 1280 so not sure what happened to the remaining 80, it may have been removed to keep the rest of the calculations consistent while still using the ipv4 values but can still serve as a middle point in ipv4).

the default mtu size is mtu sizes[number of mtu sizes - 1, with said value being the default index].

the reason why the mtu size is zero padding is because that if size of buffer is more than the currently choosen mtu size it wont be sent, so the client will know that and then change into another mtu size depending on the implemention.

Psuedo code through the original raknet way:
```cpp
RequestingConnection: global
{
   SystemAddress: address;
   RequestsMade;
   SendConnectionAttemptCount defaults to 15;
   IsConnecting;
   NextRequestTime;
   TimeBetweenConnectionAttempts defaults to 500;
};

g_RequestedConnections: DynamicArray;

i := 0;

loop i < g_NumOfRequestedConnections then
   s_rcs := g_RequestedConnections[i];
   s_timeNow := current time ms;
   if s_rcs->NextRequestTime < s_timeNow then
      s_unsetAddr := s_rcs->SystemAddress is unset;
      s_tooManyRequests := s_rcs->RequestsMade is s_rcs->SendConnectionAttemptCount + 1
      if s_unsetAddr or s_tooManyRequests then
         if (s_tooManyRequests and !s_unsetAddr and s_rcs->IsConnecting) then
            // send ConnectionAttemptFailed packet.
         end
         complete_unset s_rcs;
      else
         s_MtuSizeIndex := s_rcs->RequestsMade / (rcs->SendConnectionAttemptCount / num of mtu sizes);
         if s_MtuSizeIndex > num of mtu sizes then
            s_MtuSizeIndex = index of default mtu size;
         end
         s_rcs->RequestsMade = s_rcs->RequestsMade + 1;
         s_rcs->NextRequestTime = s_timeNow + s_rcs->TimeBetweenConnectionAttempts;

         // send OpenConnectionRequest1 packet with mtu size being the mtu sizes[s_MtuSizeIndex].

         s_SendToStart := current time ms;

         if unable to send packet due to it being too big (error code: 10040) then
            // "don't use this mtu size again" said in the original.
            s_rcs->RequestsMade = (s_MtuSizeIndex + 1) * (s_rcs->SendConnectionAttemptCount / num of mtu sizes);
            s_rcs->NextRequestTime = s_timeNow;
         else
            s_SendToEnd := current time ms;
            if s_SendToStart - s_SendToEnd < 100 then
               // "drop to the lowest mtu" said in the original.
               s_LowestMtuIndex := s_rcs->SendConnectionAttemptCount / num of mtu sizes * default mtu index;
               if s_LowestMtuIndex > s_rcs->RequestsMade then
                  s_rcs->RequestsMade = s_LowestMtuIndex;
                  s_rcs->NextRequestTime = s_timeNow;
               else
                  s_rcs->RequestsMade = s_rcs->SendConnectionAttemptCount + 1;
               end
            end
         end
         i = i + 1;
      end
   else
      i = i + 1;
   end
end
```

### OpenConnectionReply1

This packet is the response to an open connection request one packet.

| Field               | Type             | Endianness |
| ------------------- | ---------------- | ---------- |
| id                  | uint8            | N/A        |
| magic               | magic            | N/A        |
| server guid         | uint64           | Big Endian |
| server has security | bool             | N/A        |
| .....               | ....1            | ....       |
| mtu size            | uint16           | Big Endian |

**`....1` shall be recorded as:**

#### if ServerHasSecurity & Libcat:

| Field               | Type             | Endianness |
| ------------------- | ---------------- | ---------- |
| has cookie          | bool             | N/A        |
| cookie              | uint32           | Big Endian |
| server public key   | uint8[294]       | N/A        |

#### if ServerHasSecurity & Nothing:

| Field               | Type             | Endianness |
| ------------------- | ---------------- | ---------- |
| cookie              | uint32           | Big Endian |

The `server has security` shall have a global variable that specifies if the server has security for later usage(if libcat) in the implemention.

### OpenConnectionRequest2

This packet is used to complete the handshake process between a client and a server.

| Field          | Type             | Endianness |
| -------------- | ---------------- | ---------- |
| id             | uint8            | N/A        |
| magic          | magic            | N/A        |
| .....          | ....1            | .....      |
| server address | address          | N/A        |
| mtu size       | uint16           | Big Endian |
| client guid    | uint64           | Big Endian |

**`....1` shall be recorded as:** 

If server has security(Libcat)

| Field              | Type             | Endianness | Note |
| ------------------ | ---------------- | ---------- | ---- |
| cookie             | uint32           | Big Endian |      |
| contains challenge | bool             | N/A        | Whether the system requires handshake challenge |
| challenge          | uint8[64]        | N/A        | The system handshake challenge bytes |

> if the server has security but this packet does not contain a challenge, then the client will be required to send a RemoteSystemRequiresPublicKey packet to notify the server that there was no challenge in the packet.

**Connection outcome**:

If the `client guid` of the client address does not exists in list then you may mark this connection as a new connnection. if both already exists(even if address isn't same but guid is already used or vice versa) then it shall not connect.

If it can connect then the sent packet would be the `OpenConnectionReply2` packet.

If vice versa it would be the `AlreadyConnected` packet.

### OpenConnectionReply2

This packet is the response to an open connection request two packet.

| Field               | Type             | Endianness | Note |
| ------------------- | ---------------- | ---------- | ---- |
| id                  | uint8            | N/A        |      |
| magic               | magic            | N/A        |      |
| server guid         | uint64           | Big Endian |      |
| client address      | address          | N/A        |      |
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

| Field   | Type  |
| ------- | ----- |
| id      | uint8 |
| type id | uint8 |

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
| client address | address     |            |
| server guid    | uint64      | Big Endian |

### ConnectionAttemptFailed

This packet is sent when the attempt count trying to join the server is higher than a certain amount (depending on your implementation) or the client does not contain an assigned address; this is what to check and send if the requirements are met before sending the `OpenConnectionRequest1` packet.

| Field | Type  | Endianness |
| ----- | ----- | ---------- |
| id    | uint8 | N/A        |

### AlreadyConnected

This packet is sent when the client is already connected.

| Field       | Type             | Endianness |
| ----------- | ---------------- | ---------- |
| id          | uint8            | N/A        |
| magic       | magic            | N/A        |
| client guid | uint64           | Big Endian |

### ConnectionRequestAccepted

This packet is the response to a connection request.

| Field                | Type        | Endianness | Note |
| -------------------- | ----------- | ---------- | ---- |
| id                   | uint8       | N/A        |      |
| client address       | address     | N/A        |      |
| client index         | uint16      | Big Endian | Current client index in list |
| server net addresses | address[10] | N/A        | Server local network addresses |
| client send time     | uint64      | Big Endian |      |
| server send time     | uint64      | Big Endian |      |

### NewIncomingConnection

This packet is sent from the client to the server .

| Field                | Type        | Endianness | Note |
| -------------------- | ----------- | ---------- | ---- |
| id                   | uint8       | N/A        |      |
| server address       | address     | N/A        |      |
| client net addresses | address[10] | N/A        | Client local network addresses |
| client send time     | uint64      | Big Endian |      |
| server send time     | uint64      | Big Endian |      |

Send the `ConnectedPing` packet or `ConnectedPong` packet depending on the side of the action.

### DisconnectionNotification

This packet is sent when a client disconnects from the server.

| Field | Type  |
| ----- | ----- |
| id    | uint8 |

### ConnectionLost

| Field          | Type        | Endianness |
| -------------- | ----------- | ---------- |
| id             | uint8       | N/A        |
| client guid    | uint64      | Big Endian |
| client address | address     | N/A        |

### IncompatibleProtocolVersion

This packet is sent when a client attempts to connect to a server with an incompatible protocol version.

| Field            | Type             | Endianness | Note |
| ---------------- | ---------------- | ---------- | ---- |
| id               | uint8            | N/A        |      |
| protocol version | uint8            | N/A        | Protocol version supported by the server |
| magic            | magic            | N/A        |      |
| server guid      | uint64           | Big Endian | Unique identifier of the server |

### Datagram

---

#### Partial Terminology

#### 1. Dgram
1. valid dgram: a datagram that is not ack nor nack.
2. ack dgram: an acked datagram.
3. nack dgram: a nacked datagram.

#### 2. Reliability
1. reliable - the reliability is of any type that is reliable.
2. sequenced - the reliability is both unreliable sequenced and reliable sequenced.
3. sequenced and ordered - the reliability is `Sequenced` and reliable ordered and reliable ordered with ack recepit.
4. reliable or in sequence - the reliability is reliable or reliable sequenced or reliable ordered.

### Reliability

Each valid dgram sent is assigned a reliability that specifies how the data should be handled by the protocol. The following table lists the available reliability ids and their properties:

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

### Set of things required in your implemention

- Retransmission: retransmit a datagram if not acknowledged.
- Reassembly: reconstruct split packets into a valid normal packet.

Every datagram is and must be valid.

| Field    | Type | Endianness | Body    |
| -------- | ---- | ---------- | ------- |
| is valid | bit  | N/A        | `....1` |
| is ack   | bit  | N/A        | `....2` |
| is nack  | bit  | N/A        | `....3` |

**`....1` shall be recorded as:**

| Field              | Type              | Endianness    |
| ------------------ | ----------------- | ------------- |
| is packet pair     | bit               | N/A           |
| is continuous send | bit               | N/A           |
| requires B and AS  | bit               | N/A           |
| range number       | uint24            | Little Endian |
| capsules           | DatagramCapsule[] | N/A           |

The `range number` is known as the sequence number in other words.

**`....2` shall be recorded as:**

| Field              | Type      | Endianness |
| ------------------ | --------- | ---------- |
| requires B and AS  | bit       | N/A        |
| AS                 | float     | N/A        |
| ranges             | RangeList | N/A        |

There would be a float `B` field below the `requires B and AS` but is commented out in the original raknet.

**`....3` shall be recorded as:**

| Field              | Type      | Endianness |
| ------------------ | --------- | ---------- |
| ranges             | RangeList | N/A        |

</br>

valid dgram header size: `2 + 3 + 4 * 1` in bytes which seems to be: `2 (nack rangelist size field) + 3 (range number of a valid dgram) + 4 * 1 (AS of ack dgram, would be * 2 instead of * 1 if B field was not commented out)` so the 1 byte that informs dgram type is excluded.

> Note: in a valid dgram every new range number must be removed from the nack range list since the datagram was received and added to the ack range list.

The "finding" this and that below is all for the valid dgram.

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

This structure is used to represent the ranges of datagrams that is acknowledged and the missing ranges in non-acknowledged datagrams.

| Field     | Type   | Endianness | Note |
| --------- | ------ | ---------- | ---- |
| is single | bool   | N/A        | If min == max, then this is set to true |
| min       | uint24 | Little Endian | Minimum value in the range|
| max       | uint24 | Little Endian | Maximum value in the range - Is not wrote if is single |

### RangeList

The range list contains an array of nodes of the min index and the max index for each datagram range number that needs to be inserted.

| Field     | Type    | Endianness | Note |
| --------- | ------- | ---------- | ---- |
| size      | uint16  | Big Endian | Nodes count |
| nodes     | Range[] | N/A        |      |

when inserting into the nodes, the the amount of nodes in there must be reduced to not waste the buffer, that is why min and max exists.

for example lets say you have an array of min and max nodes where each of them is single:

```
[
   <min 0, max: 0>
   <min: 2, max: 2>
   <min: 3, max: 3>
   <min: 5, max: 5>
   <min: 6, max: 6>
   <min: 8, max: 8>
   <min: 8, max: 8> - there shouldn't be any duplicates in a real-world implementation.
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
| buffer size             | uint16  | Big Endian    | Size in bits |
| reliable capsule index  | uint24  | Little Endian | Index used for reliable packets (requires reliability check) |
| sequenced capsule index | uint24  | Little Endian | Index used for sequenced packets (requires reliability check) |
| order                   | ....1   | .....         |      |
| split packet info       | ....2   | .....         |      |
| buffer                  | uint8[] | N/A           |      |

**`....1` shall be recorded as:**

| Field            | Type      | Endianness    |
| ---------------- | --------- | ------------- |
| ordering index   | uint24    | Little Endian |
| ordering channel | uint8     | N/A           |

**`....2` shall be recorded as:**

| Field            | Type      | Endianness | Note |
| ---------------- | --------- | ---------- | ---- |
| size             | uint32    | Big Endian | size of the split packet |
| id               | uint16    | Big Endian | id of the split packet |
| index            | uint32    | Big Endian | index of the current split |

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
The UserPacketEnum id is `0x86`, which marks the beginning of where you can start using your packet ids (for user packets).

The id sent over network would be `UserPacketEnumId` + user packet id (it must not make the `UserPacketEnumId` surpass the `uint8 limit`; it can start from 0, 1, ...).

It is recommended is to create a packet that contains the compressed payload that will then be decompressed when decoding the packet considering you have a completed implementation to not exceed the `uint8 limit` while sending smaller data compared to normal (may be slower than a raw payload depending on the compressor).

### Sending a RakNet Packet

#### Sending an online packet

The online packets(internal packets sent over valid dgram) are all reliable except the ping/pong; the online packets are not sequenced or ordered, just reliable or unreliable.

#### Sending a user packet

Psuedo code:

This follows the default raknet splitting algorithm.

```py
global/method(depends on your choice):`max payload size` := client mtu size - valid dgram header size
if libcat & server has security then
   global/method:`max payload size` -= libcat:auth:enc:OVERHEAD_BYTES or 11
end

method:`max block size` := global/method:`max payload size` - capsule:`size` # the size is not the buffer size but the calculated capsule size.

method:`packet buffer len` := len(capsule:`buffer`)

method:`split packet` := method:`packet buffer len` is greater than method:`max block size`

# split packets must not be unreliable. if they are, convert them to reliable packets so they are successfully sent and each part is in order to reassemble.
# once the converting is done you can do stuff like sequenced index and reliable index and ordering stuff and so on.
# then you can split the data.

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

## Info Sources

Where the information was gathered from.

- <a href="https://github.com/facebookarchive/RakNet">Original RakNet</a>: nearly everything.
- <a href="http://www.jenkinssoftware.com/raknet/manual/programmingtips.html">Jenkins software programming tips</a>: mtu size chart.
