# Chat

- 基于Socket的长连接，使用第三方库cocoaAsyncSocket建立的连接。
- 数据传输使用的是ProtocolBuffer。ProtocolBuffer相比JSON的优势是解析迅速，占用内存小
- 聊天消息的协议是自己定制的
- 根据不同的位，标识不同的信息。type, header, packetId, length, payload
- type 分为连接请求，连接返回。ping请求，ping返回，消息返回，消息请求等
- 聊天消息   sqlId  msgId  type content  from to direction chatid   status。数据库存储的就是这样的消息

### RPC消息
- 发送消息的时候，将消息的packid当做key，存在一个字典中。value存的是封装的一个对象。存的是frame和相应的操作
- 发送消息的时候，开启一个定时器，当定时器触发的时候，看字典里有没有对应的操作
- 有对应消息说明，发送失败。调用失败的calllback
- 没有对应消息，说明发送成功。
- 当收到消息的时候，根据head判断当前消息是不是RPC的回复，如果是，根据packetID找到对应的消息。
- 断线重连后，将没有收到回复的RPC消息。重新发送。


