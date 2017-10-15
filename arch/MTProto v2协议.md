# MTProto V2 协议

## 编码层(Encoding)

MTProto的数据类型是从ProtoBuf借鉴过来的.

varint - unsigned integer

Each byte in a varint, except the last byte, has the most significant bit (msb) set – this indicates that there are further bytes to come. The lower 7 bits of each byte are used to store the two's complement representation of the number in groups of 7 bits, least significant group first.

int - signed 32bit integer

int value is encoded as 4-byte big endian signed integer

long - signed 64bit intenger

long value is encoded as 8-byte big endian signed integer

byte - unsigned byte

byte value is encoded as single big endian byte

bytes - byte array

bytes is encoded as varint of bytes length and raw bytes

longs - long array

bytes is encoded as varint of longs length and then raw long values

string - byte array that contains UTF-8 text

string is encoded in the same way as bytes

## 链接层(Connection Level)

客户端链接到服务端之后, 首要任务是进行握手操作, 握手成功之后链接成功建立. Client端无需等待握手结果即可发送数据(*服务端会缓存数据吗, 还是直接drop??* -服务端会把msg stash起来, 然后在逐步).

通道传输单位是Frame. Frame中的header字段标记了不同的Frame类型, 如果header为0, payload需要发送到transport level. 其他的Frame都是信号帧(signaling message). Frame结构的定义是:

```java
Frame {

  // Index of package starting from zero.

  // If packageIndex is broken connection need to be dropped.

  packageIndex: int

  // Type of message 16进制

  header: byte

  // Package payload

  body: bytes

  // CRC32 of body 循环冗余校验码, 32位整数

  crc32: int

}

```



Signaling Message:

```

// header = 1

// Ping message can be sent from both sides

Ping {

  randomBytes: bytes

}



// header = 2

// Pong message need to be sent immediately after receving Ping message

Pong {

  // Same bytes as in Ping package

  randomBytes: bytes

}



// header = 3

// Notification about connection drop

Drop {

  messageId: long

  errorCode: byte

  errorMessage: string

}



// header = 4

// Sent by server when we need to temporary redirect to another server

Redirect {

  host: string

  port: int

  // Redirection timeout

  timeout: int

}



// header = 6

// Proto package is received by destination peer. Used for determening of connection state

Ack {

  receivedPackageIndex: int

}



// header == 0xFF

Handshake {

  // Current MTProto revision

  // For Rev 2 need to eq 1

  protoRevision: byte

  // API Major and Minor version

  apiMajorVersion: byte

  apiMinorVersion: byte

  // Some Random Bytes (suggested size is 32 bytes)

  randomBytes: bytes

}



// header == 0xFE

HandshakeResponse {

  // return same versions as request, 0 - version is not supported

  protoRevision: byte

  apiMajorVersion: byte

  apiMinorVersion: byte

  // SHA256 of randomBytes from request

  sha1: byte[32]

}

```

## 传输层(Transport Level)

目标:

- Fast connection recreation(链接快速重建)\
- Minimize traffic amount(流量优化)\
- Connection-agnostic package transmition(链接无关的消息传递)\
- Recovering after connection die(链接go die恢复)\
- Recovering after server die(服务器go die恢复)\
- Work in networks that work on buggy hardware that breaks connection in random situations(适应弱网环境)

传输层消息结构定义如下:

```

// Message Container

Message {

  // message identifier

  messageId: long

  // message body

  message: byte

}



Package {

  // unique identifier that is constant thru all application lifetime

  authId: long

  // random identifier of current session

  sessionId: long

  // message

  message: Message

}

```



Requesting Auth ID 消息定义:

Requesting Auth ID, Auth ID是有Diffie-Hellman算法生成. 在Auth ID生成之前, authId/sessionId应该为0.

```

RequestAuthId {

  HEADER = 0xF0

}



ResponseAuthId {

  HEADER = 0xF1

  authId: long

}

```



RPC and Push 消息定义:

客户端通过ProtoRpcRequest消息向后端发送rpc请求, 服务端响应为ProtoRpcRequest. 服务端主动发起的Push为ProtoPush.



消息优先级(how???): When implementing protocol it is useful to have ability to mark some RPC Requests as high priority and send them always as separate packages and before any other packages. This technic can reduce message send latency.



```

ProtoRpcRequest {

 HEADER = 0x03

 // Request body

 payload: bytes

}



ProtoRpcResponse {

 HEADER = 0x04

 // messageId from Message that contains ProtoRpcRequest

 messageId: long

 // Response body

 payload: bytes

}



ProtoPush {

 HEADER = 0x05

 // Push body

 payload: bytes

}



```



Service Package:

前后端信息同步?



ACKNOWLEDGE:

ProtoRpcResponse自动确认ProtoRpcRequest, 不需要额外确认一次.



Server端在收到消息之后会尽快的发送MessageAck, 并且尽量尝试批量确认.



Client 端发送confirmation逻辑:

  When new connection is created: first message is always MessageAck if available

  When unconfirmed buffer contains more than 10 messages

  When client need to send some normal priority packages: client group it with MessageAck

  When uncorfirmed buffer is older than 1 minute



```

MessageAck {

 HEADER = 0x06

 // Message Identificators for confirmation

 messageIds: longs

}

```



RESEND

如果链接的一端超出1min都没有收到对方的Ack消息, 发送方可以尝试重发消息或提醒对方确认.

如果发送的Package大于1K, 优先提醒对方确认, 对方收到提醒之后应该回复MessageAck或者RequestResend.

```

// Notification about unsent message (usually ProtoRpcRequest or ProtoPush)

UnsentMessage {

 HEADER = 0x07

 // Sent Message Id

 messageId: long

 // Size of message in bytes

 len: int

}



// Notification about unsent ProtoRpcResponse

UnsentResponse {

 HEADER = 0x08

 // Sent Message Id

 messageId: long

 // Request Message Id

 requestMessageId: long

 // Size of message in bytes

 len: int

}



// Requesting resending of message

RequestResend {

 HEADER = 0x09

 // Message Id for resend

 messageId: long

}

```



** NEWSESSIONCREATED ** ???有点不是特别明白, NewSessionCreated到底是Client端发起还是Server端发起???

NewSessionCreated是服务端向客户端发起的Session重建提醒. 当客户端收到该消息之后, 应该重发所有的ProtoRpcRequests, 重新订阅所有的events.

```

NewSession {

 HEADER = 0x0C

 // Created Session Id

 sessionId: long

 // Message Id of Message that created session

 messageId: long

}

```



SESSIONHELLO

Session重建之后, client可以发送任何的request或者SessionHello

```

SessionHello {

  HEADER = 0x0F

}

```



SESSIONLOST

Server端向客户端发起的session丢失提醒, Client端需要发起一个请求或者SessionHello.

```

SessionLost {

  HEADER = 0x10

}

```



AUTHIDINVALID

客户端AuthId不对, 这条消息发送之后链接会断开.

```

AuthIdInvalid {

  HEADER = 0x11

}

```



DROP

Server端主动断开链接

```

Drop {

 HEADER = 0x0D

 // Message Id of message that causes Drop. May be zero if not available

 messageId: long

 // Error Message

 errorMessage: String

}

```



Container:

批量发送消息

```

Container {

 HEADER = 0x0A

 // Messages count

 count: varint

 // Messages in container

 data: Message[]

}

```

## 同步层(Basic Sync Level)

同步层是和后端Rpc API对应的, 向后端的api对应的是RpcRequest中的methodId字段. 同步层的消息会被封装到ProtoRpcRequest/ProtoRpcResponse(Transport层消息结构).

Rpc Call:

```

RpcRequest {

    HEADER = 0x01

    // ID of API Method Request

    methodId: int

    // Encoded Request

    body: bytes

}



// Successful RPC

RpcOk {

    HEADER = 0x01

    // ID of API Method Response

    methodResponseId: int

    // Encoded response

    body: bytes

}



// RPC Error

RpcError {

    HEADER = 0x02

    // Error Code like HTTP Error code

    errorCode: int

    // Error Tag like "ACCESS_DENIED"

    errorTag: string

    // User visible error

    userMessage: string

    // Can user try again

    canTryAgain: bool

    // Some additional data of error

    errorData: bytes

}



// RPC Flood Control. (流量控制)

// Client need to repeat request after delay

RpcFloodWait {

    HEADER = 0x03

    // Repeat delay on seconds

    delay: int

}



// Internal Server Error

// Client may try to resend message

RpcInternalError {

    HEADER = 0x04

    canTryAgain: bool

    tryAgainDelay: int

}

```



Server Push:

Basic Sync Level contains wrapper for Push. This objects are encoded as body in ProtoPush.

```

Push {

  // Push Entity Id

  updateId: int

  // Encoded Push body

  body: bytes

}

```