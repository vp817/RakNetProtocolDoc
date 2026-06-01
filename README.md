# RakNet Protocol Documentation

This is a raknet protocol documentation. It has everything required to create a raknet server or a client(the default raknet and not the other types).

If there is something that was written related to raknet which the reader still has no info on, then the reader shall not panic and start searching as it will eventually be mentioned later on what it is(or it is possible to just search within this document, if there was no other mention then the reader can search outside of it).

If anything was incorrect or misspelled, you can pull request a fix to them if they really are wrong(and a pull request for other stuff would also be appreciated)

## DataTypes

| Type         | Size       | Desc |
| ------------ | ---------- | ---- |
| uint8        | 1 byte     |      |
| uint16       | 2 bytes    |      |
| uint24       | 3 bytes    | Unsigned 24-bit integer with a minimum value of 0 and a maximum value of (2^24)-1 |
| uint32       | 4 bytes    |      |
| uint64       | 8 bytes    |      |
| string       | variable   | UTF-8 encoded string preceding with 2 bytes(uint16, little endian) that represents its length |
| magic        | 16 bytes   | A uint8 array with a specific sequence `[0x00, 0xFF, 0xFF, 0x00, 0xFE, 0xFE, 0xFE, 0xFE, 0xFD, 0xFD, 0xFD, 0xFD, 0x12, 0x34, 0x56, 0x78]` that remains unchanged which is used to identify offline/unconnected packets |
| zero-padding | variable   | A single zero value uint8 recorded in sequence until the required size. |
| bool         | 1 byte     | Written or read as a single uint8, with a value of 0 or 1 (0 represents false, and 1 represents true). |
| address      | 7-29 bytes | See below(Address DataType). |
| bit          | 1 bit      | a bit can be 1 or 0 where it is written in buffer as a single uint8 extended by 0s until it achieves 8 bits if not all of the bits in the uint8 are used (the bits follow the MSb order) |
| float        | 4 bytes    | IEEE 754 single-precision floating-point number |

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
each part shall be recorded from the MSB to the LSB then the address would be the result with all of its bits inverted.

converting an ipv4 address into an ipv4 string:
the value would be the uint32 address with all of its bits inverted that is going to be used to extract each part into its place in the string from the MSB to the LSB.

> Note: alternatively it is possible to use inet functions for address encoding or decoding for both an ipv4 or an ipv6 (specifically inet_pton, inet_ntop for ipv4).

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

Note: Minecraft does not use libcat encryption(as of now).

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
| ConnectionAttemptFailed        | 0x11 | OFFLINE |
| AlreadyConnected               | 0x12 | OFFLINE |
| ConnectionRequestAccepted      | 0x10 | ONLINE  |
| NewIncomingConnection          | 0x13 | ONLINE  |
| DisconnectionNotification      | 0x15 | BOTH    |
| ConnectionLost                 | 0x16 | BOTH    |
| IncompatibleProtocolVersion    | 0x19 | OFFLINE |

### Offline send and receive sequence

The first packets before a connection(not the socket connection) was craeted(still on unconnected packets) are the packets of the offline type, which means that those are the packets that is handled outside of datagrams(and the opposite holds true).

Below, there is no information on why this and that, but the packet name says all that is needed to be said.

The "Datagrams" to be handled and sent is all possible datagram types, where the `ip dgram` holds the online packets.

Offline:

- Client: UnconnectedPing
   - Server: UnconnectedPong
- Client: OpenConnectionRequest1
   - Server: OpenConnectionReply1 / IncompatibleProtocolVersion
- Client: OpenConnectionRequest2
   - Server: OpenConnectionReply2 & Create connection.
- Client: Datagrams
   - Server: Handle accordingly.

Online:

- Client: ConnectionRequest
   - Server: ConnectionRequestAccepted
- Client: NewIncomingConnection
   - Server: ConnectedPing

Before sending the ConnectedPing after handling the NewIncomingConnection packet in raknet port the reader should check if the server port is same as the binding port before connecting (optional).

After every 5 seconds the ConnectedPing is sent to keep the connection alive.

Client <-> Server: user messages after connecting through NewIncomingConnection.

### UnconnectedPing / UnconnectedPingOpenConnections

This packet is used to determine if a server is online or not.

For unconnected ping open connections: the server will only send a reply if the client's connection to the server is currently open. This helps to prevent sending responses to clients that have closed their connections.

| Field            | Type             | Endianness | Note |
| ---------------- | ---------------- | ---------- | ---- |
| id               | uint8            | N/A        |      |
| send ping time   | uint64           | Big Endian | Client current time in milliseconds. |
| magic            | magic            | N/A        |      |
| client guid      | uint64           | Big Endian |      |

> Note: everything time related in raknet is high resolution(like for example `client send time` is the current high resolution time in milliseconds, doesn't matter if milliseconds, microseconds or whatever else its all high resolution).

### UnconnectedPong

This packet is the response to an unconnected ping packet.

| Field            | Type             | Endianness | Note |
| ---------------- | ---------------- | ---------- | ---- |
| id               | uint8            | N/A        |      |
| send ping time   | uint64           | Big Endian | Shall be set to the time received from UnconnectedPing. |
| server guid      | uint64           | Big Endian |      |
| magic            | magic            | N/A        |      |
| message          | string           | Big Endian | Usually used for server information. |

### ConnectedPing

This packet is used to keep the connection alive between the client and the server.

| Field            | Type   | Endianness | Note |
| ---------------- | ------ | ---------- | ---- |
| id               | uint8  | N/A        |      |
| send ping time | uint64 | Big Endian | Client current time in milliseconds. |

### ConnectedPong

This packet is the response to a connected ping packet.

| Field            | Type   | Endianness | Note |
| ---------------- | ------ | ---------- | ---- |
| id               | uint8  | N/A        |      |
| send ping time   | uint64 | Big Endian | Shall be set to the time received from ConnectedPing. | 
| send pong time   | uint64 | Big Endian | Current time in milliseconds. |

### OpenConnectionRequest1

This packet is used to initiate the handshake process between a client and a server.

| Field            | Type             | Endianness | Note |
| ---------------- | ---------------- | ---------- | ---- |
| id               | uint8            | N/A        |      |
| magic            | magic            | N/A        |      |
| protocol version | uint8            | N/A        | Protocol version of the client. |
| mtu size         | zero-padding     | N/A        |      |

if protocol version isn't the same as the server then connection fails by `IncompatibleProtocolVersion`.


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
         s_MtuSizeIndex := s_rcs->RequestsMade / (s_rcs->SendConnectionAttemptCount / num of mtu sizes);
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

if server and OpenConnectionRequest1's mtu size is greater than the MaximumMtuSize then the OpenConnectionReply1 mtu shall be MaximumMtuSize, but if not then it shall be the mtu size given by previously said received packet.

**`....1` shall be recorded as:**

if ServerHasSecurity & Libcat:

| Field               | Type             | Endianness |
| ------------------- | ---------------- | ---------- |
| has cookie          | bool             | N/A        |
| cookie              | uint32           | Big Endian |
| server public key   | uint8[294]       | N/A        |

if ServerHasSecurity & Nothing:

| Field               | Type             | Endianness |
| ------------------- | ---------------- | ---------- |
| cookie              | uint32           | Big Endian |

The `server has security` shall have a global variable that specifies if the server has security for later usage in the implemention.

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

If server has security

| Field              | Type             | Endianness | Note |
| ------------------ | ---------------- | ---------- | ---- |
| cookie             | uint32           | Big Endian |      |
| contains challenge | bool             | N/A        | Whether the system requires handshake challenge |
| challenge          | uint8[64]        | N/A        | The system handshake challenge bytes |

> if the server has security but this packet does not contain a challenge, then the client shall send a RemoteSystemRequiresPublicKey packet to notify the server that there was no challenge in the packet.

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

| Field              | Type       | Endianness | Note |
| ------------------ | ---------- | ---------- | ---- |
| id                 | uint8      | N/A        |      |
| client guid        | uint64     | Big Endian |      |
| incoming timestamp | uint64     | Big Endian | Current send time in milliseconds. |
| do security        | bool       | N/A        |      |
| client proof       | uint8[32]  | N/A        | Proof of client authentication |
| do identity        | bool       | N/A        |      |
| identity proof     | uint8[294] | N/A        | Proof of client identity |

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
| send ping time       | uint64      | Big Endian | Shall be the incoming timestamp of ConnectionRequest. |
| send pong time       | uint64      | Big Endian | Current send time in milliseconds. |

ConnectedPong payload at the end of the buffer.

### NewIncomingConnection

This packet is sent from the client to the server .

| Field                | Type        | Endianness | Note |
| -------------------- | ----------- | ---------- | ---- |
| id                   | uint8       | N/A        |      |
| server address       | address     | N/A        |      |
| client net addresses | address[10] | N/A        | Client local network addresses |
| send ping time       | uint64      | Big Endian | Shall be the send pong time of ConnectionRequestAccepted. |
| send pong time       | uint64      | Big Endian | Current send time in milliseconds. |

ConnectedPong payload at the end of the buffer.

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
1. ip dgram: a datagram that is not ack nor nack; used for internal packets (short for InternalPacket Datagram, not to be confused with something else).
2. ack dgram: an acked datagram.
3. nack dgram: a nacked datagram.

#### 2. Reliability
1. reliable - the reliability is of any type that is reliable.
2. sequenced - the reliability is unreliable sequenced or reliable sequenced.
3. ordered - the reliability is reliable ordered or reliable ordered with ack receipt
3. sequenced or ordered - the reliability is sequenced or ordered.

### Reliability

Each ip dgram sent is assigned a reliability that specifies how the data should be handled by the protocol. The following table lists the available reliability ids and their properties:

| Name                           | ID  | Is Reliable | Is Ordered | Is Sequenced |
| ------------------------------ | --- | ----------- | ---------- | ------------ |
| Unreliable                     | 0   | False       | False      | False        |
| UnreliableSequenced            | 1   | False       | True       | True         |
| Reliable                       | 2   | True        | False      | False        |
| ReliableOrdered                | 3   | True        | True       | False        |
| ReliableSequenced              | 4   | True        | True       | True         |
| UnreliableWithAckReceipt       | 5   | False       | False      | False        |
| ReliableWithAckReceipt         | 6   | True        | False      | False        |
| ReliableOrderedWithAckReceipt  | 7   | True        | True       | False        |

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

| Field              | Type              | Endianness    | Note |
| ------------------ | ----------------- | ------------- | ---- |
| is packet pair     | bit               | N/A           |      |
| is continuous send | bit               | N/A           | See below. |
| needs B and AS     | bit               | N/A           |      |
| sequence number    | uint24            | Little Endian |      |
| packets            | InternalPacket[]  | N/A           | There is no length but internal packets written one after another. |

When the outgoing packet queue has more than 0 packets then `bandwidth has exceeded statistic`; therefore, `is continous send` is set to true.

**`....2` shall be recorded as:**

| Field              | Type      | Endianness |
| ------------------ | --------- | ---------- |
| has B and AS       | bit       | N/A        |
| AS                 | float     | N/A        |
| ranges             | RangeList | N/A        |

There would be a float `B` field below the `has B and AS` and not the `AS` but is commented out in the original raknet.

The `has B and AS` is set to true if ip dgram of the ack needs B and AS.

**`....3` shall be recorded as:**

| Field              | Type      | Endianness |
| ------------------ | --------- | ---------- |
| ranges             | RangeList | N/A        |

The information that are not used/handled by default in raknet and the stuff that can be ignored(in other words if its not UDT congestion management) are: B and AS related stuff, packet pair. However, they are still required to be present.

</br>

dgram header size: `2 + 3 + 4 * 1` in bytes which seems to be: `2 (nack rangelist size field) + 3 (sequence number of a ip dgram) + 4 * 1 (AS of ack dgram, would be * 2 instead of * 1 if B field was not commented out)` so the 1 byte that informs dgram type is excluded.

### RangeNode

This structure is used to represent the ranges of datagrams that is acknowledged and the missing ranges in non-acknowledged datagrams.

| Field     | Type   | Endianness    | Note |
| --------- | ------ | ------------- | ---- |
| is single | bool   | N/A           | If min == max, then it is a single range node |
| min       | uint24 | Little Endian | Minimum value in the range |
| max       | uint24 | Little Endian | Maximum value in the range - Is not wrote if is single |

### RangeList

The range list contains an array of range nodes for each datagram sequence number that needs to be inserted.

| Field     | Type        | Endianness | Note |
| --------- | ----------- | ---------- | ---- |
| size      | uint16      | Big Endian | The number of how many nodes exist in the array. |
| nodes     | RangeNode[] | N/A        |      |

See the `Algorithms` section for more details.

### InternalPacket

This structure represents an internal packet in an ip dgram.

| Field                   | Type    | Endianness    | Note |
| ----------------------- | ------- | ------------- | ---- |
| reliability             | 3 bits  | Big Endian    |      |
| is split                | bit     | N/A           | If true, the packet is a split packet |
| buffer size             | uint16  | Big Endian    | Size of the buffer field below in bits |
| reliable index          | uint24  | Little Endian | Index used for reliable packets (requires reliability check) |
| sequencing index        | uint24  | Little Endian | Index used for sequenced packets (requires reliability check) |
| order                   | ....1   | .....         |      |
| split packet info       | ....2   | .....         |      |
| buffer                  | uint8[] | N/A           | The data to be sent over or received. |

**`....1` shall be recorded as:**

| Field            | Type      | Endianness    |
| ---------------- | --------- | ------------- |
| ordering index   | uint24    | Little Endian |
| ordering channel | uint8     | N/A           |

**`....2` shall be recorded as:**

| Field            | Type      | Endianness | Note |
| ---------------- | --------- | ---------- | ---- |
| count            | uint32    | Big Endian | number of how many splits there are of the packet |
| id               | uint16    | Big Endian | id of the split packet |
| index            | uint32    | Big Endian | index of the current split |

The split id is basically a number that is used to reassemble/reconstruct the split packet; The split id is incremented by 1 after every datagram is completely split and pushed into the outgoing packet queue, and reset after reaching the uint16 limit.

### InternalPacket Message Size
The header size of the internal packet that would later be used for calculations.

1. Increment the byte by 1 to represent the reliability. (always there)
2. Increment the byte by 2 to represent the size of the buffer. (always there)
3. If the reliability is any type of reliable, increment the byte by 3 to represent the `reliable index`.
4. If the reliability is sequenced, increment the byte by 3 to represent the `sequencing index`.
5. If the reliability is sequenced or ordered increment the byte by 3 for the `ordering index`, and then by 1 step for the `ordering channel`.
6. If the internal packet is a split packet, increment the byte by 4 for the `count`, 2 for the `id`, and 4 for the `index` of the split packet.

### UserPacketEnum
The UserPacketEnum id is `0x86`, which marks the beginning of where you can start using your packet ids (for user packets).

The id sent over network would be `UserPacketEnumId` + user packet id (it must not make the `UserPacketEnumId` surpass the `uint8 limit`; it can start from 0, 1, ...).

It is recommended is to create a packet that contains the compressed payload that will then be decompressed when decoding the packet considering you have a completed implementation to not exceed the `uint8 limit` while sending smaller data compared to normal (may be slower than a raw payload for smaller data but shouldnt be for big depending on the compressor).

## Algorithms

### RangeList
when inserting into the nodes, the the amount of nodes in there must be reduced to not waste the buffer, that is why min and max exists.

In the original RakNet, RangeList has its own DataStructure with an insert function that would do what was said below automatically, whereas some implementations insert the ip dgram sequence numbers into a simple dynamic array, that would then be sorted and converted like shown above when it's time that the dgram is sent or received.

Let's say there is an array of nodes where all of them are single:

```
[
   <min 0, max: 0>
   <min: 2, max: 2>
   <min: 3, max: 3>
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
   <min: 0, max: 0> (single)
   <min: 2, max: 3> (not single)
   <min: 8, max: 8> (single)
   <min: 10, max: 10> (single)
   <min: 12, max: 15> (not single)
]
```

### Data splitting

A user packet is an internal packet containing the user message as its buffer. When sending a user packet the message shall not exceed the `maximum mtu size excluding header bytes`, where that field is equals to the `client mtu size excluding the udp header size` - dgram header size, because if it does then that packet will never be received due to its large size(if with libcat and server has security then the `maximum payload` shall be decremented by 11 or `libcat_OVERHEAD_BYTES`).

The user message shall be split into separate internal packets which are all reliable(if the user packet was sent as unreliable then it shall be converted into a reliable type to not mess up the reassembling process, or atleast that's how it is done in the original raknet).

The way to determine whether a data requires splitting is done by checking if the user message length is greater than the `maximum allowed user message size`, which is equals to the `maximum mtu size excluding header bytes` - the `maximum message header size`.

The `maximum message header size` is the internal packet size(not to be confused with the user data size) of an internal packet with the reliability type of RELIABLE_SEQUENCED and a dummy split packet info to sum all fields.

If the user packet requires splitting then it is possible to know how many splits there will be for the user packet by ceiling the quotient of the user message length and the `maximum allowed user message size`.

The way to split the user message(atleast how it is done in the original RakNet) would be to first iterate over the number of how many split packets there will be.
In each iteration, a specific start offset and end offset/how many the bytes to send in this iteration is calculated, with the start offset being the product of the current index of this iteration and the `maximum allowed user message size`. while the number of bytes being sent in this iteration would be the difference between the user message length and the start offset. If the number of bytes being sent in this iteration is greater than the `maximum allowed user message size` then it shall be set to that said maximum value. If the number of bytes being sent in this iteration is not equals to the `maximum allowed user message size` then it shall be set to be the difference between the user message length and the product of the current index of this iteration and the `maximum allowed user message size`. The split internal packet shall then be constructed with the split packet info being [The number of how many splits there, the id of the split, the current index of this iteration]. The buffer of the split internal would be the original user message sliced by the start offset and the end offset that was calculated, and the rest of the fields shall be the same as the ones of the original user packet.

### Congestion Management
Congestion occurs when the packets are sent too fast or in a too high volume, thats when congestion management comes into hand.

Congestion management shall only happen in the online phase of a connection(where the handshake process has been completed and is now connected with the other side of the action) due to trivial reasons.

The congestion manager used in raknet by default without any option set is called `SlidingWindow`; while other congestion managers exist in raknet such as "UDT", the only one that will be documented below is the default one as the UDT congestion control algorithm is already documented in "raknet/help/congestioncontrol.html".

-------------------

SlidingWindow is some kind of flow control used within various protocols, such as TCP and UDT.

There are multiple phases in SlidingWindow, which are: In flight, 

#### Definitions
- SYN: A static 10000 (10ms in μs) time constant used as a delay threshold used so that the acks can be buffered until necessary.

- RTT(Round Trip Time): The total time in μs for a data packet to travel from a source and destination and back.

- RTO(Retransmission Time Out): The time that an acknowledgement is expected after a data packet is sent out. If there is no acknowledgement after this amount of time elapsed, a timeout event should be triggered.

- CWND(Congestion window): Maximum bytes allowed on wire at once.

- MaxDatagramPayload: Maximum amount of bytes that the user can send, e.g. the mtu size.

- UnacknowledgedBytes: A number that is a sum of the complete length(internal packet message size + user buffer length) of every internal packet that are in flight.

- RetransmissionBandwidth: Represents the total volume of data currently in flight. in other words, the UnacknowledgedBytes.

- TransmissionBandwidth: Represents the remaining capacity available to send new data without overwhelming the connection.

The CWND and "MaxMtuExcludingUdpHeader" shall both initially be the same as the MaxDatagramPayload value.

#### When an ACK is Received

Before anything, there must be a way to know the time of a datagram that was sent(the tick time in microseconds of the update cycle where the datagram was sent in), like some kind of history to later determine the RTT.

When an ack is received and its RangeList is iterated over, the RTT can be calculated if the sequence number in this iteration exists in the previously said ip dgram history concept through:

RTT = ReceiveTime - SendTime

Where the ReceiveTime is the time when the ACK was received(in microseconds), and the SendTime is the time that the datagram of this sequence number was sent at(in microseconds). This RTT value shall be known as the LastRTT. If the sequence number does not exist in the history then the RTT shall be 0.

There should also be a way to know the expected rtt, the deviation rtt. where the expected rtt is the running averate of rtts, while deviation rtt measures how much those trip times fluctuate. The expected rtt, deviation rtt by default is unset and when an ack comes and it is unset then they should just be the same as LastRTT. But if not, then calculate them by first calculating the difference:

Difference = RTT - EstimatedRTT

Then to compute the EstimatedRTT:

EstimatedRTT = EstimatedRTT + d * Difference

Where d is a smoothing factor between 0 and 1, and it is equals to 0.05 in RakNet.

To compute the DeviationRTT/RTTVAR:

Deviation = Deviation + d * (|Difference| - Deviation)

These values can later be then used if packets are continous sent/bandwidth has exceeded statistic(is is specified above on what this is, outside of congestion management). If bandwidth has exceeded statistics, then everything that is said below shall happen.

Firstly, there must be a way to track the congestion control period, where if the sequence number has passed the next congestion control block, then next congestion control block shall be updated to be the value of the next ip dgram sequence number(the exact sequence number of the last ip datagram sent + 1), then we shall not back off in this control block so that it can be used for other stuff later on.

Slow Start is a network traffic control technique that prevents a connection from overloading a network with too much data at once. To achieve that, the connection shall start at a very small speed then increase its speed to the maximum until it finds the network's maximum limit. (However, its not slow at all, because it keeps increasing the packets that is allowed to be sent every time an ack is received). There must be a slow start threshold(which will later be used for retransmission or if a packet was not received by the other side), which will then be used to determine whether it's currently the slow start phase or not, by checking if:

CWND <= SlowStartThreshold || SlowStartThreshold is unset

If its currently the slow start phase then the CWND shall be recomputed as:

CWND = CWND + MaxMtuExcludingUdpHeader

If CWND is greater than the SlowStartThreshold and SlowStartThreshold is set, then the CWND shall be recomputed as:

CWND = SlowStartThreshold + MaxMtuExcludingUdpHeader * MaxMtuExcludingUdpHeader / CWND

If it's not the Slow Start phase and is a new congestion control period, then the CWND shall be recomputed as:

CWND = CWND + (MaxMtuExcludingUdpHeader * MaxMtuExcludingUdpHeader / CWND)

#### When a Packet is Retransmitted

If packets are continous sent/bandwidth has exceeded statistic and if we shouldn't back off this congestion control block(see When an ACK is Received) and if CWND is greater than the MaxMtuExcludingUdpHeader multiplied by 2 then the Slow Start Threshold shall be computed through:

Slow Start Threshold = CWND / 2

If the Slow Start Threshold is greater than the MaxMtuExcludingUdpHeader it shall be capped back to MaxMtuExcludingUdpHeader.

The CWND shall be reset back to MaxMtuExcludingUdpHeader, and then we can back off this congestion control block and update the next congestion control block to be the value of the next ip dgram sequence number(the exact sequence number of the last ip datagram sent + 1).

#### When a NACK is Received

If packets are continous sent/bandwidth has exceeded statistic and if we shouldn't back off this congestion control block(see When an ACK is Received) then the Slow Start Threshold shall be recomputed through:

Slow Start Threshold = CWND / 2

#### RTO for Retransmission

This is going to later be used for something called the next action time for internal packet.

The maximum threshold shall be 2000000 and the additional variance shall be 30000 (both in microseconds).

If the ExpectedRTT is unset(which means its most likely the first packet) then the RTO shall be the maximum threshold.

The threshold shall be computed through:

Threshold = (u * EstimatedRTT + q * DeviationRTT) + Additional Variance

Where u is to multiply the average travel time and q is to multiply the variance; u = 2 and q = 4.

The Threshold must be an integer. If the Threshold is greater than the maximum then it shall be the maximum instead.

That Threshold is the RTO for Retransmission.

#### When Should ACKs be Sent

ACKs do not have to be sent immediately. Instead, they can be buffered up such that groups of acks are sent at a time. This reduces the overall bandwidth usage. How long they can be buffered depends on the retransmit time of the sender.

Firstly, the rto/retranmit time of the sender must be calculated if the LastRTT is set:

RTO = LastRTT + SYN

And to determine whether they should be sent or not:

Time >= oldest unsent ack + SYN

"Time" denotes the current time of the connection update cycle in microseconds.

If not, then there is no acks to buffer, so they can just be sent immediately.

#### When ACKs are Sent

The oldest unsent ack shall be reset to its intial value that is not a time.

#### When a Packet Comes

When an ip datagram is received(and only if an ip dgram, because only they contain sequence numbers), the congestion manager calculates whether there are any skipped messages or not. If the oldest unsent ack is not set to a time, it shall be set to the current time that the datagram has been received in microseconds.

A sequence number hole/when there are skipped messages is when an ip dgram arrives that is not in sequence(where the sequence number is greater than expected). To determine whether there are any holes, there shall be a way to know what the expected next sequence number is, and with that it is possible to know how big is the hole by calculating the difference between the current ip datagram sequence number and the expected next sequence number. If the sequence number is the same as the expected next sequence number, then there is no hole/skipped messages. but if it is greater than the expected next sequence number then there are skipped messages. The number of skipped messages can then be iterated over in descending order and the sequence number in this iteration that will be inserted into the nack range list is equals to the difference between the current ip dgram sequence number and the current index of this iteration. Once they are inserted into the nack range list the current ip datagram shall not be skipped because it is a genuine ip dgram. If the number of skipped messages is more than 1000 then it shall be capped to 1000 and if it is more than 50000 then the current ip dgram shall be ignored without inserting anything into the nack range list. The expected next sequence number shall be updated if there is a hole and if there isn't, where in both cases, the expected next sequence number shall be the current ip dgram sequence number + 1.

#### Transmission Bandwidth and Retransmission Bandwidth

The transmission bandwidth and the retransmission bandwidth are both used to know whether the connection shall transmit or retransmit the packets in the retransmission queue, outgoing packet queue. Getting the transmission bandwidth is simple, if UnacknowledgedBytes <= CWND then it is equals to CWND-UnacknowledgedBytes, if not then it shall be set to 0, and the reason for that should be obvious.

Retransmission only happens for reliable packets. A retransmission queue is a dynamic array with its indexes being a reliable index and with its value being the internal packet, with the index masked by the RETRANSMISSION_QUEUE_MASK(511).
``RetransmissionQueue[reliable_index & RETRANSMISSION_QUEUE_MASK] = internal packet``

A send queue is a dynamic array of internal packets.

If there are packets to be retransmitted or if there are anything in the outgoing packet queue then what is said below can happen:

If both the retransmission bandwidth and transmission bandwidth is greater than 0 then retransmission is possible. If so, then it is possible to send the packets that are in the retransmission queue.

the complete internal packet size shall be equals to the internal packet message size + the internal packet user buffer size.

Below is how RakNet sends its datagrams not not overload the other side, but it depends on how the reader wants to implement it after reading and comprehending it.

There should be a way to detect how much bytes(the complete internal packet size) is going to be sent individually when retransmitting and when transmitting, and there should be something like a total number for both of them that is going to be used later on(which shall be known as the total dgrams to be sent size), which is going to be reset every single time that a condition related to it below occurs and is true.

After iterating through the retransmission queue while the total dgrams to be sent also being is lower than the retransmission bandwidth, then the first thing that shall happen is checking whether the current time of this update cycle(in microseconds) is greater than the next action time of the internal packet(what this is will be said will be said below and can then be concluded intuitively), and if it is then there shall be a check that checks if the number of how many bytes is going to be retransmitted(after the retransmission queue iteration is complete) but has been pushed into the send queue(the same thing as what was said above) + the complete internal packet of this iteration size is greater than the `maximum datagram size excluding header bytes`(see the data splitting section on what this is). If that condition is achieved, then the iterator shall be broken out of; but if not then the internal packet of this iteration shall be pushed into the send queue and the total datagrams to be sent with the number of how many bytes is going to be transmitted then after that is when the congestion management "when a packet is retransmitted" happens. After that the internal packet next action shall be updated to be the "RTO for retransmission" + the current time of this update cycle(in microseconds) and then pushed back into the retransmission queue, however without modifying the unacknowledged bytes. If the internal packet of this iteration  next action time is not greater than the current time of this update cycle(in microseconds), then the iterator should be broken out of. The reason for everything happening here can also be concluded intuitively.

If the total datagrams to be sent is lower than the transmission bandwidth then the outgoing packet queue can be iterated through with that condition also being a reason if the iterator ends. After iterating through and if the user internal packet buffer size is more than 0(which means is not an empty buffer) it shall then be eligible to be sent. There shall be the same check and behavior for when retransmitting and the number of bytes retransmit(however in this case, the number of bytes transmit) is greater than the `maximum datagram size excluding header bytes`. The internal packet of this iteration shall then be removed from the outgoing packet queue and then if it is reliable it shall have its own reliable index that keeps increasing for each reliable packet and also a next action time where it shall be set to be the same way it is set when retransmitting. If the next action time cannot be greater than 10000000, and if it is in the retransmission queue then an overflow likely happened. if not then it shall be added to the retransmission queue(where the reliable index is the one of the reliable and not the next reliable index). The reason why its going to be added to the retransmission queue should be clear and the unacknowledged bytes number shall be updated to include the complete internal packet size of this packet. Regardless if its reliable or not, it shall be pushed into the send queue with the behavior being the same as the one previously said in retransmission(only for pushing and nothing else).

The send queue can be sent by then with its datagram containing header info and whatever else(and it is possible to do stuff like datagram history from here); Needs B And AS is when congestion mangement is on slow start, however B and AS is unused in SlidingWindow congestion management. The total datagrams to be sent shall be reset to 0 in here too so that it does not break other stuff later on. (Datagrams should be sent after ACKs if possible)

--------------------------

When an ACK comes and interated through and whatever else happens(like congestion related stuff and whatever), it shall remove from the datagram history the sequence number of this iteration(at last). and if the internal packets of that datagram is in the retransmission queue then they shall be removed if reliable and the reliable index is the same as the one of the internal packet of this iteration(where the acknowledged bytes number is also decremented by the complete size of that internal packet).

When a NACK comes and iterated through and whatever else happen and if the sequence number of this iteration exists in the datagram history and if an internal packet of its internal packets is reliable and if it exists in the retransmission queue and if the the next action time is not 0(of the internal packet in the retransmission queue) then it shall be updated to be the time of this update cycle(in microseconds).

### Reliable Window

Reliable window is for received internal packets where it is possible to check if there are duplicates or out of bounds reliable indexes if reliable.

if reliable and is not a split internal packet and the reliable index is lower than the reliable window start(0 by default) or greater than the reliable window end(2048 by default), then it is out of range. if it exists in the reliable window array then it is a duplicate internal packet.

If both of what was stated above are true, then the internal packet shall be ignored. If it is none of what was stated above, then it shall be added into the reliable window array where the reliable window array is a dynamic array. It is unnecessary to put the internal packet as a value, the only thing necessary is knowing whether the reliable index is in the reliable window array.

if the reliable index start is the same as the reliable index of this reliable index, then it shall be incremented until it is not a value that exists in the reliable window(with the reliable window end also increasing), while also removing from the reliable window the reliable window start value in the iteration.

### Receiveing Ordering And Sequencing

Note: The internal packet must be reliable or sequenced for this section.

Firstly before anything, the ordering channel must be validated whether it was valid or not, and the way to do that is by checking if the ordering channel is more than the NumberOfOrderedStreams or less than 0 then it shall be ignored as it is corrupted.

There shall be a fixed array with the size of the NumberOfOrderedStreams for the ordering channels and an index for that channel, where the index grows ascendingly depending on the situation.

There shall also be another fixed array with the size of the NumberOfOrderedStreams, where each element contains the highest sequencing index for that channel (the last received sequencing index + 1).

There also shall be another fixed array or a heap with the size of the NumberOfOrderedStreams for internal packets that were received out of order. e.g. reliable index 1, reliable index 2, reliable index 5, reliable index 3 reliable index 4.

There shall be a fixed size array with the size of NumberOfOrderedStreams for packets that were received out of order but this time it only contains the index/heap index offsets for the ordering channel of that internal packet.

If the current internal packet ordering channel exist in the array that contains the indexes, then the current index in that array shall be checked whether it was the same as the current internal packet ordering index.

If it is the same:

-------------------------------------------------------
if the internal packet is sequenced:

If the ordering channel exists in the highest sequencing indexes array(those should always be true due to the fact that the arrays are fixed and there was a check before that checks for channel corruption) then:

If the highest sequencing index is less than the current internal packet sequencing number then the internal packet shall be dropped as its out of sequence. If not then the highest sequencing index shall be updated and the internal packet shall be handled accordingly.

if the internal packet is not sequenced:

The current internal packet shall be handled then the array that contains the ordering indexes shall be incremented (but not the same as the highest sequencing indexes). The reason for all of that is obvious, that's why they are not explained and only the way it shall be done is written. The highest sequencing index in the channel of the current internal packet shall be set back to 0 as the sequencing was broken, that is if there was any.

The ordered internal packets in this channel that were out of order eariler can/should also be handled in this place where they are handled in a heap-like way(if it is a fixed array and not a heap data structure that tries to imitate a heap data structure). Then way to handle it shall be implementation specific. e.g. In RakNet, it iterates through the heap if its not empty and if the first element ordering index is the same as the one stored in the ordering indexes array, then it pops the first element consecutively, then pushes it into the array that handles the packets then if its reliable ordered it increments the ordering index for the channel in the array of that popped element. If not then it sets the highest sequencing index to be the same as the one of that popped element.

-------------------------------------------------------

If it is not the same and the current ordering index is greater than that stored then:

-------------------------------------------------------
If the ordering channel of the current internal packet does not exist within the heap, then the heap index offsets array with for this ordering channel shall be updated with its value being the last one of the ordering indexes array.

The ordering hole count can be obtained by checking the difference between the ordering index of the current internal packet and the one in the heap index offsets for this channel.

The weight for each element that will be pushed into heap shall be: the ordering hole count * 1048576(that is how it is in raknet, i'm unsure of what this number is, however it does not matter if the reader knows how a heap works and the stuff that will be said later on. Note that this is the base weight)

If the current internal packet is sequenced then the weight shall be incremented by the sequencing index, if not then it is likely ordered so the weight shall be incremented by 1048576 - 1. So the weight for ordered packets is higher than sequenced, which means they will be processed before sequenced packets later on. and the reason of why the sequencing indexes are placed as weight is so that they are in sequence later on when they are going to be processed.

-------------------------------------------------------

If not any of those conditions stated before then the internal packet shall be dropped due to it being out of order.

## Sending a Packet

### Sending an Online Packet

The online packets(internal packets sent over ip dgram) are all reliable ordered except the ping/pong; all online packets to be sent are immediate sent but not user data(see splitting buffers for data packets for sending user data).

### Splitting buffers for data packets

Unrelated to sending an online packet;

Buffers must be split to not exceed the mtu size and fail the sending of the data packets(in other words user packets).

If no splitting is required then it shall directly be pushed into the outgoing packet queue. If it is, then split the message and the split internal packets shall also be pushed into the outgoing packet queue.

## Source

- <a href="https://github.com/facebookarchive/RakNet">Original RakNet</a>: everything.
