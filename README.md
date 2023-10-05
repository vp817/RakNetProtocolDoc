# RakNet Protocol Documentation

This is the latest RakNet protocol documentation. It includes information on the data types used in the protocol and details about each packet and their associated fields.

Feel free to join my discord server if you have any more questions: <a href="https://discord.gg/Ytwwgs5nFU">Join</a>

## Data Types

### Integer Data Types

| Type | Size | Note |
| ---- | ---- | ---- |
| uint8 | 1 byte | Unsigned 8-bit integer |
| uint16 | 2 bytes | Unsigned 16-bit integer |
| uint24 | 3 bytes | Unsigned 24-bit integer with a minimum value of 0 and a maximum value of 16777215 |
| uint32 | 4 bytes | Unsigned 32-bit integer |
| uint64 | 4/8 bytes | Unsigned 64-bit integer (4 bytes for 32-bit systems, 8 bytes for 64-bit systems) |

### String Data Types

| Type | Size | Note |
| ---- | ---- | ---- |
| uint16-string | variable | UTF-8 encoded string with a length of 2 bytes preceding the string |

### Other Data Types

| Type | Size | Note |
| ---- | ---- | ---- |
| magic | 16 bytes | An array of unsigned 8-bit integers with a specific sequence and it is: [0x00, 0xFF, 0xFF, 0x00, 0xFE, 0xFE, 0xFE, 0xFE, 0xFD, 0xFD, 0xFD, 0xFD, 0x12, 0x34, 0x56, 0x78] |
| pad-with-zero | variable | Null bytes used for padding with a size of your choice |
| bool | 1 byte | Write or read as a single unsigned 8-bit integer, with a value of 0 or 1 (Zero is used to represent false, and One is used to represent true) |
| address | 7-29 bytes | IPv4: 1 byte (address version), 4 bytes (IP address), 2 bytes (port), IPv6: 1 byte (address version), 2 bytes (address family), 2 bytes (port), 4 bytes (flow info), 16 bytes (IP address), 4 bytes (scope ID) |
| bit | 1 bit | Write or read the bit inside the buffer after you completed it |
| float | 4 bytes | IEEE 754 single-precision floating-point number |

## Packets

### Packet Identifiers

| Name | ID |
| ---- | -- |
| UnconnectedPing | 0x01 |
| UnconnectedPingOpenConnections | 0x02 |
| UnconnectedPong | 0x1c |
| ConnectedPing | 0x00 |
| ConnectedPong | 0x03 |
| OpenConnectionRequestOne | 0x05 |
| OpenConnectionReplyOne | 0x06 |
| OpenConnectionRequestTwo | 0x07 |
| OpenConnectionReplyTwo | 0x08 |
| ConnectionRequest | 0x09 |
| RemoteSystemRequiresPublicKey | 0x0a |
| ConnectionRequestAccepted | 0x10 |
| NewIncomingConnection | 0x13 |
| DisconnectionNotification | 0x15 |
| ConnectionLost | 0x16 |
| IncompatibleProtocolVersion | 0x19 |

### UnconnectedPing

This packet is used to determine if a server is online or not. It also include information about open connections.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| onlyReplyOnOpenConnections | bool | N/A | If set to true, the server will only send a reply if the client's connection to the server is currently open. This helps to prevent sending responses to clients that have closed their connections. This is especially useful in peer-to-peer networks where clients may come and go frequently. By setting this field to true, the client can avoid wasting network resources by only sending requests when it knows that the server will be able to respond. The resulting message ID for the request would be `UnconnectedPingOpenConnections`. If set to false, the default behavior is `UnconnectedPing`. |
| id | uint8 | N/A | Unique identifier for the ping |
| clientSendTime | uint64 | Big Endian | Client timestamp used to calculate the latency |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| clientGuid | uint64 | Big Endian | Unique identifier for the client |

### UnconnectedPong

This packet is the response to an unconnected ping packet.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier associated with the ping |
| serverSendTime | uint64 | Big Endian | Server timestamp used to calculate the latency |
| serverGuid | uint64 | Big Endian | Unique identifier for the server |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| responseData | uint16-string | Big Endian | Response data typically used for server information |

### ConnectedPing

This packet is used to keep the connection alive between the client and the server.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier for the ping |
| clientSendTime | uint64 | Big Endian | Client timestamp used to calculate the latency |

### ConnectedPong

This packet is the response to a connected ping packet.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier associated with the ping |
| clientSendTime | uint64 | Big Endian | Client timestamp from the ping |
| serverSendTime | uint64 | Big Endian | Server timestamp used to calculate the latency |

### OpenConnectionRequestOne

This packet is used to initiate the handshake process between a client and a server.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier for the request |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| protocolVersion | uint8 | N/A | Protocol version supported by the client |
| mtuSize | pad-with-zero | N/A | Maximum transmission unit (MTU) size of the client |

> Note: when using pad-with-zero to pad the MTU size, the total size of the buffer should be the MTU size if you are padding then its minus if you are reading then ite plus the current size of the buffer plus 28 bytes (used for the UDP header). You can also check if the packet buffer is valid by checking if the buffer size is 18 bytes so you know that what you are doing is correct

### OpenConnectionReplyOne

This packet is the response to an open connection request one packet.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier associated with the request |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| serverGuid | uint64 | Big Endian | Unique identifier for the server |
| serverHasSecurity | bool | N/A | Whether the server requires security or not |
| mtuSize | uint16 | Big Endian | Maximum transmission unit (MTU) size of the server |

### OpenConnectionReplyOne With Security

This packet is the response to an open connection request one packet with additional security information.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier associated with the request |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| serverGuid | uint64 | Big Endian | Unique identifier for the server |
| serverHasSecurity | bool | N/A | Whether the server requires security or not |
| hasCookieON | bool | N/A | Whether the packet includes a cookie |
| cookie | uint32 | Big Endian | Cookie value |
| serverPublicKey | uint8[294] | N/A | Public key used for encryption |
| mtuSize | uint16 | Big Endian | Maximum transmission unit (MTU) size of the server |

### OpenConnectionRequestTwo

This packet is used to complete the handshake process between a client and a server.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier for the request |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| serverAddress | uint8[7-29] | N/A | Server IP address and port combo |
| mtuSize | uint16 | Big Endian | Maximum transmission unit (MTU) size of the client |
| clientGuid | uint64 | Big Endian | Unique identifier for the client |

### OpenConnectionReplyTwo

This packet is the response to an open connection request two packet.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier associated with the request |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| serverGuid | uint64 | Big Endian | Unique identifier for the server |
| clientAddress | uint8[7-29] | N/A | Client IP address and port combo |
| mtuSize | uint16 | Big Endian | Maximum transmission unit (MTU) size of the server |
| requiresEncryption | bool | N/A | Whether the connection requires encryption or not |

### ConnectionRequest

This packet is used to establish a connection between a client and a server with security enabled.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier for the request |
| clientGuid | uint64 | Big Endian | Unique identifier for the client |
| serverSendTime | uint64 | Big Endian | Timestamp for the server |
| doSecurity | bool | N/A | Whether the connection requires security or not |
| clientProof | uint8[32] | N/A | Proof of client authentication |
| doIdentity | bool | N/A | Whether the packet requires an identity proof |
| identityProof | uint8[294] | N/A | Proof of client identity |

> Note: If the identity proof is invalid and `doIdentity` is set to true, immediately send a `RemoteSystemRequiresPublicKey` packet with a type ID of `ClientIdentityIsInvalid`. If `doIdentity` is set to false and there is no identity proof, send a `RemoteSystemRequiresPublicKey` packet with a type ID of `ClientIdentityIsMissing`.

### RemoteSystemRequiresPublicKey

This packet is used to request public keys for client authentication and identification.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier for the request |
| typeID | uint8 | N/A | Type of public key request |

#### RemoteSystemRequiresPublicKey Type IDs

| Name | ID |
| ---- | -- |
| ServerPublicKeyIsMissing | 0 |
| ClientIdentityIsMissing | 1 |
| ClientIdentityIsInvalid | 2 |

### ConnectionRequestAccepted

This packet is the response to a connection request with security enabled.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier associated with the request |
| clientAddress | uint8[7-29] | N/A | Client IP address and port combo |
| clientIndex | uint16 | Big Endian | Unique identifier assigned to the client |
| serverMachineAddresses | address[10] | N/A | Server machine addresses |
| clientSendTime | uint64 | Big Endian | Timestamp for the client |
| serverSendTime | uint64 | Big Endian | Timestamp for the server |

### NewIncomingConnection

This packet is sent to all other clients when a new client connects to the server.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier associated with the connection |
| serverAddress | uint8[7-29] | N/A | Server IP address and port combo |
| clientIndex | uint16 | Big Endian | Unique identifier assigned to the client |
| clientMachineAddresses | address[10] | N/A | Client machine addresses |
| clientSendTime | uint64 | Big Endian | Timestamp for the client |
| serverSendTime | uint64 | Big Endian | Timestamp for the server |

After you send or receive this packet to the server, you need to keep the connection alive by sending periodic `ConnectedPing` packets. These packets are essentially a way to say "hey, I'm still here and connected to the server." The server also sends `ConnectedPong` packets back in response to confirm that the connection is still active. This ping-pong process helps prevent the connection from timing out due to inactivity or network issues.

### DisconnectionNotification

This packet is sent when a client disconnects from the server.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier for the disconnection |

### ConnectionLost

This packet is sent when a connection to a client is lost.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier associated with the lost connection |
| clientGuid | uint64 | Big Endian | Unique identifier for the client |
| clientAddress | uint8[7-29] | N/A | Client IP address and port combo |
| wasGeneratedLocaly | bool | N/A | Whether the packet was generated locally |

> Note: If you're not sure what to set `wasGeneratedLocaly` to, just set it to `true`.

### IncompatibleProtocolVersion

This packet is sent when a client attempts to connect to a server with an incompatible protocol version.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| id | uint8 | N/A | Unique identifier associated with the connection |
| protocolVersion | uint8 | N/A | Protocol version supported by the server |
| magic | uint8[16] | N/A | Magic sequence to identify the packet |
| serverGuid | uint64 | Big Endian | Unique identifier for the server |

### Datagram

This packet is used for sending and receiving data between clients and the server. It can be one of three types: ValidDatagram, AckedDatagram, or NackedDatagram.

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
| ranges | Range | N/A | Array of range values that were received |

> Note: Prepare a new buffer for B and AS before proceeding further.

### NackedDatagram

This packet is a response to a ValidDatagram indicating that the server has not received all of the expected data.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| ranges | Range | N/A | Array of range values that were not received |

### Range

This structure is used to represent the missing ranges of a NackedDatagram.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| size | uint24 | Little Endian | Number of ranges in the array |
| isSingle | bool | N/A | If true, there is only one missing range |
| min | uint24 | Little Endian | Minimum value in the range |
| max | uint24 | Little Endian | Maximum value in the range |

### ValidDatagram

This packet is used for sending and receiving data between clients and the server.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| isPacketPair | bit | N/A | If true, the packet is one of two associated packets |
| isContinousSend | bit | N/A | If true, the packet is a continuous send packet |
| requiresBAndAS | bit | N/A | If true, the packet includes B and AS values |
| rangeNumber | uint24 | Little Endian | The sequence number of the datagram |
| capsuleLayers | DatagramCapsuleLayer[] | N/A | Array of capsule layers in the packet |

### DatagramCapsuleLayer

This structure represents a capsule layer in a ValidDatagram.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| reliability | 3 bits | Big Endian | Type of reliability used |
| isSegmented | bit | N/A | If true, the packet is segmented |
| size | uint16 | Big Endian | Size of the buffer in bits |
| reliableCapsuleIndex | uint24 | Little Endian | Index used for reliable packets |
| sequencedCapsuleIndex | uint24 | Little Endian | Index used for sequenced packets |
| arrangement | CapsuleArrangement | N/A | Arrangement of the capsule used for sequenced and arranged packets |
| segment | CapsuleSegment | N/A | Segment of the capsule used when capsule is segmented |
| buffer | Buffer | N/A | Buffer data |

### CapsuleArrangement

This structure represents the arrangement of a capsule in a ValidDatagram.

| Field | Type | Endianness | Note |
| ----- | ---- | ----------| ----- |
| arrangedCapsuleIndex | uint24 | Little Endian | Index of the arranged capsule |
| arrangementChannel | uint8 | N/A | Channel used for the arrangement |

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

| Name                     | ID  | Is Reliable | Is Arranged | Is Sequenced |
| ------------------------| --- | -----------| -----------|--------------|
| Unreliable               | 0   | No          | No         | No           |
| UnreliableSequenced      | 1   | No          | Yes        | Yes          |
| Reliable                 | 2   | Yes         | No         | No           |
| ReliableArranged         | 3   | Yes         | Yes        | No           |
| ReliableSequenced        | 4   | Yes         | Yes        | Yes          |
| UnreliableWithAckReceipt | 5   | No          | No         | No           |
| ReliableWithAckReceipt   | 6   | Yes         | No         | No           |
| ReliableArrangedWithAckReceipt | 7 | Yes   | Yes        | No           |

* **Unreliable:** This Reliability TypeID sends datagrams without any guarantees that they will arrive at the destination. They are not guaranteed to be delivered in any specific order or at all.

* **UnreliableSequenced:** This Reliability TypeID sends datagrams without any guarantees that they will arrive at the destination but ensures that they are delivered in the sequence they were sent.

* **Reliable:** This Reliability TypeID sends datagrams guaranteed to be delivered in the order they were sent. If a datagram is lost, RakNet will retransmit it until it is acknowledged by the receiver.

* **ReliableArranged:** This Reliability TypeID sends datagrams guaranteed to be delivered in the order they were sent. If a datagram is lost, RakNet will not retransmit it and all datagrams before it that have not been acknowledged. 

* **ReliableSequenced:** This Reliability TypeID sends datagrams guaranteed to be delivered in the order they were sent and ensures that they are delivered sequentially.

* **UnreliableWithAckReceipt:** This Reliability TypeID sends datagrams without any guarantees that they will arrive at the destination, but the receiver sends an acknowledgement receipt upon receipt of this datagram.

* **ReliableWithAckReceipt:** This Reliability TypeID sends datagrams guaranteed to be delivered in the order they were sent, and the receiver sends an acknowledgement receipt upon receipt of this datagram.

* **ReliableArrangedWithAckReceipt:** This Reliability TypeID sends datagrams guaranteed to be delivered in the order they were sent but all datagrams before it that have not been acknowledged, and the receiver sends an acknowledgement receipt upon receipt of this datagram.

For more information you can look at: <a href="https://datatracker.ietf.org/doc/html/rfc793">RFC 793</a>

### Reliability definitions
Here you can find every reliablity definition which is used in other places at the documentation.

1. Reliable - This is when the reliability is of any type that is reliable
2. Sequenced - This is when the reliability is both unreliable sequenced and reliable sequenced
3. Sequenced and arranged - This is when the reliablity is `Sequenced` and reliable arranged and reliable arranged with ack recepit

### Retransmission
RakNet uses selective repeat retransmission to ensure reliable delivery of datagrams. When a datagram is sent, it is assigned a sequence number. If a datagram is not acknowledged within a certain timeout period, RakNet will retransmit the datagram using the same sequence number. When the receiver receives a duplicate datagram with the same sequence number, it can discard it, since it has already acknowledged that sequence number.

For more information you can look at: <a href="https://datatracker.ietf.org/doc/html/rfc793">RFC 793</a>

### AckQueue / NackQueue
The AckQueue and NackQueue are used to keep track of which datagrams have been acknowledged and which have not. The AckQueue stores a list of datagram sequence numbers that have been successfully acknowledged, while the NackQueue stores a list of datagram sequence numbers that have not been acknowledged and need to be retransmitted. When a datagram is received with a sequence number that has already been acknowledged, it can be discarded.

For more information you can look at: <a href="https://datatracker.ietf.org/doc/html/rfc793">RFC 793</a>

### PacketPair
PacketPair is a technique used by RakNet to improve the efficiency of datagram retransmissions. When a datagram is acknowledged, RakNet sends the next datagram in the sequence as well. This allows the receiver to begin processing the next datagram immediately, reducing latency and improving throughput.

### ContinuousSend
ContinuousSend is a feature of RakNet that allows datagrams to be sent continuously without waiting for acknowledgement. This can improve performance in some cases, but can also lead to packet loss and retransmissions, since the sender does not wait for feedback before sending the next datagram.

### Reassembly
RakNet uses a reassembly mechanism to reconstruct segmented datagrams that may be received out of order. When a datagram is segmented, each segment is assigned a unique identifier. When the receiver receives a segment, it is buffered until all segments with the same identifier have been received. Once all segments have been received, they are reassembled into the original datagram.

For more information you can look at: <a href="https://datatracker.ietf.org/doc/html/rfc793">RFC 793</a>

### Flow Control
Flow Control is a RakNet mechanism used to manage the rate of data transmission between sender and receiver. It ensures that the receiver can handle the incoming data at a pace it can process, preventing overwhelming or overflowing the receiver's buffer. Flow control helps maintain a balance between the sender's transmission speed and the receiver's processing capability, optimizing the overall efficiency and stability of the communication.

For more information you can look at: <a href="https://datatracker.ietf.org/doc/html/rfc793">RFC 793</a>

### Congestion Control
Congestion control is a RakNet technique used to prevent network congestion by balancing data transmission rates. Techniques like TCP congestion control, packet dropping, rate limiting, traffic shaping, QoS, and load balancing are used. These techniques ensure reliable data delivery and efficient transmission in RakNet.

For more information you can look at: <a href="https://datatracker.ietf.org/doc/html/rfc5681">RFC 5681</a>

### Segment
Segmentation in RakNet enhances data delivery by dividing large messages into smaller segments. These segments, with headers indicating position and size, ensure successful reassembly on the receiver's end. By comparing the buffer size to the Maximum Transmission Unit (MTU) size (usually 1400), if the buffer exceeds the MTU, it is split into segments for transmission. This mechanism in RakNet prevents data loss, manages large payloads, and guarantees reliable transmission in networked applications.

For more information you can look at: <a href="https://datatracker.ietf.org/doc/html/rfc793">RFC 793</a>

### B
"B" represents the link capacity or the maximum amount of data that can be transmitted per second over the network link. The link capacity is determined by multiple factors, including the network infrastructure, the network configuration, and the available resources. By using a float value, the network capacity can be represented more accurately and precisely, enabling better utilization of the available resources.

### AS
"AS" represents the data arrival rate, which is the rate at which the data is generated and sent by the sender. The use of a float value allows for more precise representation of the arrival rate, which can vary based on the application requirements and the network conditions. By comparing the arrival rate with the link capacity, the sender can determine the amount of data that can be sent over the network link without causing congestion or degradation of performance.

### CapsuleLayer Size
To determine the size of the capsule layer, you can follow these steps:
1. Increment the byte by 1 step to represent the reliability.
2. Increment the byte by 2 steps to accommodate the size of the buffer.
3. If the reliability is any type of reliable, increment the byte by 3 steps to represent the `reliableCapsuleIndex`.
4. If the reliability is sequenced, increment the byte by 3 steps to represent the `sequencedCapsuleIndex`.
5. If the reliability is sequenced and arranged increment the byte by 3 steps for the `arrangedCapsuleIndex`, and then by 1 step for the `arrangementChannel`.
6. If the capsule is segmented, increment the byte by 4 steps for the `size`, 2 steps for the `id`, and 4 steps for the `index` of the segment.

### UserPacketEnum
The UserPacketEnum ID is `0x86`, which marks the beginning of where you can start using your custom packet IDs.

### Sending a Non-RakNet Packet
To send a non-RakNet packet, first determine if segmentation is needed by comparing the buffer size to the MTU size minus 2, plus 3, plus 4 times 1 (for the datagram's data header byte length), and subtracting 11 if security is in use. Then, subtract the given value with the capsule size. If segmentation is necessary, reassemble the packet before adding it to the datagram queue for transmission. If no segmentation is required, add it directly to the queue. Remember, segmented packets must not be unreliable; if they are, convert them to reliable packets to guarantee successful and ordered delivery of all packet parts.
