# RakNetProtocolDoc
latest doc about raknet protocol

# DataTypes
|Type|Size|Note|
|----|----|----|
|uint8|1||
|uint16|2||
|uint16-string|uint16 size + the string size|That is an utf-8 encoded string the uint16 as the length of it|
|uint32|4||
|uint64|4-8|4 for 32bit systems 8 for 64 bit systems|
|magic|16|The magic is an uint8 array and its contents is: [0x00, 0xFF, 0xFF, 0x00, 0xFE, 0xFE, 0xFE, 0xFE, 0xFD, 0xFD, 0xFD, 0xFD, 0x12, 0x34, 0x56, 0x78]|
|pad-with-zero|your choice|You will fill null byte with the size of your choice as the size of the fill and on reading just get the remaining bytes that are zero size but if the pad-with-zero is what you wrote at the last place just get the remaining bytes length|
|bool|0-1|read uint8 and if you get 0x00 then return false. 0x01 = true|
|address|7|1 uint8 for address version 4 uint8's is for the address ip 1 uint16 for the address port|

# Packets
### Identifiers
|Name|ID|
|----|--|
|UnconnectedPing|0x01|
|UnconnectedPingOpenConnections|0x02|
|UnconnectedPong|0x1c|
|ConnectedPing|0x00|
|ConnectedPong|0x03|
|OpenConnectionRequestOne|0x05|
|OpenConnectionReplyOne|0x06|
|OpenConnectionRequestTwo|0x07|
|OpenConnectionReplyTwo|0x08|
|ConnectionRequest|0x09|
|RemoteSystemRequiresPublicKey|0x0a|
|ConnectionRequestAccepted|0x10|
|NewIncomingConnection|0x13|
|DisconnectionNotification|0x15|
|ConnectionLost|0x16|
|IncompatibleProtocolVersion|0x19|
|FrameSet|0x80..0x8d|
|Nack|0xa0|
|Ack|0xc0|

### UnconnectedPing || UnconnectedPingOpenConnections
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|onlyReplyOnOpenConnections|bool|None|
|id|uint8|None|
|clientSendTime|uint64|Big Endian|
|magic|uint8[16]|None|
|clientGuid|uint64|Big Endian|

> Note about the "onlyReplyOnOpenConnections":
>
> it isnt read / written inside the binary.
>
> if its true then the id should be the unconnected ping open connections id and if false it should be the unconnected ping id

### UnconnectedPong
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|serverSendTime|uint64|Big Endian|
|serverGuid|uint64|Big Endian|
|magic|uint8[16]|None|
|responseData|uint16-string|Big Endian|

> Note that the responseData is what you will get as the motd take mcbe|mcee data as example:
>
> (MCPE|MCEE);(Motd);(Current MCBE|MCEE Protocol Version);(Current MCBE|MCEE Version);(Online Players Count);(Max Players Count);(UniqueID (20 digits recommended));(Default World Name);(Default GameMode String);(Default GameMode Int);(Ipv4);(Ipv6);
>
> the "Default Gamemode Int" to "Ipv6" is not necessary

### ConnectedPing
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|clientSendTime|uint64|Big Endian|

### ConnectedPong
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|clientSendTime|uint64|Big Endian|
|serverSendTime|uint64|Big Endian|

### OpenConnectionRequestOne
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|magic|uint8[16]|None|
|protocolVersion|uint8|None|
|mtuSize|pad-with-zero|None|

> Note about the mtuSize and the pad-with-zero of open connection request one is:
>
> you will need to put when padding with zero the mtuSize plus the current available bytes size plus 28 (Udp Header Size)
>
> if you want to check if the size of the current available bytes is valid check if the available bytes size is 18 before you pad-with-zero

### OpenConnectionReplyOne
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|magic|uint8[16]|None|
|serverGuid|uint64|Big Endian|
|serverHasSecurity|bool|None|
|mtuSize|uint16|Big Endian|

### OpenConnectionReplyOne With Security
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|magic|uint8[16]|None|
|serverGuid|uint64|Big Endian|
|serverHasSecurity|bool|None|
|hasCookieON|bool|None|
|cookie|uint32|Big Endian|
|serverPublicKey|uint8[]|None|
|mtuSize|uint16|Big Endian|

### OpenConnectionRequestTwo
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|magic|uint8[16]|None|
|serverAddress|uint8[7]|None|
|mtuSize|uint16|Big Endian|
|clientGuid|uint64|Big Endian|

### OpenConnectionReplyTwo
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|magic|uint8[16]|None|
|serverGuid|uint64|Big Endian|
|clientAddress|uint8[7]|None|
|mtuSize|uint16|Big Endian|
|requiresEncryption|bool|None|

### ConnectionRequest
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|clientGuid|uint64|Big Endian|
|serverSendTime|uint64|Big Endian|
|doSecurity|bool|None|

### ConnectionRequest With Security
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|clientGuid|uint64|Big Endian|
|serverSendTime|uint64|Big Endian|
|doSecurity|bool|None|
|clientProof|uint8[32]|None|
|doIdentity|bool|None|
|identityProof|uint8[]|None|

> immediately send the RemoteSystemRequiresPublicKey packet with the typeID of "ClientIdentityIsInvalid" if the identity is invalid and doIdentity is set to be true if its false the typeID should be "ClientIdentityIsMissing"

### RemoteSystemRequiresPublicKey
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|typeID|uint8|None|

#### The RemoteSystemRequiresPublicKey TypeID's
|Name|ID|
|----|------|
|ServerPublicKeyIsMissing|0|
|ClientIdentityIsMissing|1|
|ClientIdentityIsInvalid|2|

### ConnectionRequestAccepted
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|clientAddress|uint8[7]|None|
|clientIndex|uint16|Big Endian|
|clientSystemAddresses|address[10]|None|
|clientSendTime|uint64|Big Endian|
|serverSendTime|uint64|Big Endian|

### NewIncomingConnection
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|serverAddress|uint8[7]|None|
|clientIndex|uint16|Big Endian|
|clientSystemAddresses|address[10]|None|
|clientSendTime|uint64|Big Endian|
|serverSendTime|uint64|Big Endian|

### DisconnectionNotification
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|

### ConnectionLost
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|clientGuid|uint64|Big Endian|
|clientAddress|uint8[7]|None|
|wasGeneratedLocaly|bool|None|

> Note that the connection lost is sent when you dont want to send the disconnection notification

### IncompatibleProtocolVersion
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|protocolVersion|uint8|None|
|magic|uint8[16]|None|
|serverGuid|uint64|Big Endian|

### FrameSet
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|
|sequenceNumber|uint24|Little Endian|
|frames|Frame[]|None|

#### The "Frame" Contents
|Field|Field Type|Field Type Endianness|Explanation|
|-----|----------|---------------------|----|
|flags|uint8|None|we use the bitwise "and" operation with the mask 0xe0 to isolate the top 3 bits and then then extract 5 bits out of the flags and you will get the reliablitiy. to know if its fragmented you will use the bitwise "and" operation again to with the mask 0x10 to check if the 4th bit is set to 1 if the result is equal to 0x10 it is fragmented|
|size|uint16|Big Endian|this is the size in bits to get it as bytes extract 3 bits out of it|
|reliableFrameIndex|uint24|Little Endian|Only if reliable|
|sequencedFrameIndex|uint24|Little Endian|Only if sequenced|
|order|FrameOrder|None|Only if ordered|
|fragment|FrameFragment|None|Only if fragmented|
|buffer|Buffer|None|The buffer can be whatever language you are working on type of buffer you would basically read out of it the "size" you got up there|

#### The "FrameOrder" Contents
|Field|Field Type|Field Type Endianness|
|-----|----------|---------------------|
|orderedFrameIndex|uint24|Little Endian|
|orderChannel|uint8|None|

#### The "FrameFragment" Contents
|Field|Field Type|Field Type Endianness|
|-----|----------|---------------------|
|compoundSize|uint32|Big Endian|
|compoundID|uint16|Big Endian|
|compoundIndex|uint32|Big Endian|

#### The Reliablitiy TypeID's
|Name|ID|Is Reliable|Is Ordered|Is Sequenced|
|----|--|-----------|----------|------------|
|Unreliable|0|No|No|No|
|UnreliableSequenced|1|No|Yes|Yes|
|Reliable|2|Yes|No|No|
|ReliableOrdered|3|Yes|Yes|No|
|ReliableSequenced|4|Yes|Yes|Yes|
|UnreliableWithAckReceipt|5|No|No|No|
|ReliableWithAckReceipt|6|Yes|No|No|
|ReliableOrderedWithAckReceipt|6|Yes|Yes|No|

### Nack || Ack
|Field|Field Type|Field Type Endianness|
|-----|---------|-------------------|
|id|uint8|None|

> The Nack and Ack are the same the only difference is the id so we extends them with another class that have the reading and writing of them we call that class Acknowledge since Ack means Acknowledgement or Acknowledged and Nack means NegativeAckowledgement or NotAcknowledged
>
> the sequenceNumbers is an array of sequence numbers is is used on "Acknowledge"  and it must be sorted. when reading from buffer you will get them sorted because it is written sorted to the bufer you are using

#### The "Acknowledge" Contents
|Field|Field Type|Field Type Endianness|
|-----|----------|---------------------|
|recordCount|uint16|Big Endian|
|records|AcknowledgeRecord[]|None|

#### The "AcknowledgeRecord" Contents
|Field|Field Type|Field Type Endianness|Note|
|-----|----------|---------------------|----|
|isSingleSequenceNumber|bool|None|True for with range false for without range|

#### The "AcknowledgeRecord" Contents with range
|Field|Field Type|Field Type Endianness|
|-----|----------|---------------------|
|isSingleSequenceNumber|bool|None|
|receivedSquenceNumber|uint24|Little Endian|

#### The "AcknowledgeRecord" Contents without range
|Field|Field Type|Field Type Endianness|
|-----|----------|---------------------|
|isSingleSequenceNumber|bool|None|
|startSequenceNumber|uint24|Little Endian|
|endSequenceNumber|uint24|Little Endian|
