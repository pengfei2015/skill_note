# 推送

### 推送流程
- 申请证书发送给服务端
- 推送的设备向APNs服务器注册，获得device_token
- 将token发给自己的应用服务器
- 服务器将指定格式的消息打包，结合设备的device_token一起发送APNs服务器
- APNs和我们的应用维持有一个基于TCP的长连接

