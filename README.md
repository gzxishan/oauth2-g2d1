# oauth2-g2d1
在oauth2协议的基础上进行了适当的修改，同时（为了平台级应用）增加了一个认证服务器主动向客户端服务器发起授权登陆的协议。
* 官方:[Oauth2.0](https://oauth.net/2/)

#协议流程图

## 一、普通授权流程
```
     +----------+
     | Resource |
     |   Owner  |
     |          |
     +----------+
          ^
          |
         (B)
     +----|-----+          Client Identifier      +---------------+
     |         -+----(A)-- &      type       ---->|               |
     |  User-   |                                 | Authorization |
     |  Agent  -+----(B)-- User authenticates --->|     Server    |
     |          |                                 |               |
     |         -+----(C)-- Authorization Code ---<|               |
     +-|----|---+                                 +---------------+
       |    |                                         ^      v
      (A)  (C)                                        |      |
       |    |                                         |      |
       ^    v                                         |      |
     +---------+                                      |      |
     |         |>---(a)-- Client Identifier  ---------'      |
     |  Client |          & secret                           |
     |         |                                             |
     |         |<---(b)----- Access Token -------------------'
     +---------+
       ^    v   
       |    |   
      (E)  (D)  
       |    |   
       ^    v   
     +---------+
     |         |
     | Resource|
     |  Server |
     |         |
     +---------+
```
### 说明：
* 步骤(a)与(b)为Client服务器单独获取access\_token，即可以提前获取（同时也作为访问其他API的凭证），且对所有资源共用一个access\_token。
* 步骤(A)的type为登陆类型，如type可表示微信登陆、小程序登陆、qq登陆、公共平台自身的帐号密码登陆。
* 步骤(D)和(E)的说明：
1. (D)第一次通过access\_token与code获取资源，code只能使用一次；(E)获取了保护资源，同时包括coder,expires\_in与refresh\_coder\_token等。
2. (D)以后通过access\_token与coder获取资源，coder可使用多次；(E)获取了保护资源。
3. 可通过refresh\_coder\_token获取一个新的coder。

### 特别说明：
步骤(A)与(C)需要对参数进行签名，且包括参数名，(A)与(C)分别由Client与认证服务器进行签名，如：
```
1、原始请求参数：appid=123,type=global,scope=base,state=abc,response_type=code,customer1=xyz,customer2=xyz2
2、增加的参数:signature_names=encode(customer1+','+customer1),timestamp,nonce
3、通过与认证服务器共有的token，对以上参数进行签名，得到signature，最终得到参数：
appid=123,type=global,scope=base,state=abc,response_type=code,customer1=xyz,customer2=xyz2,
signature_names=encode(customer1+','+customer1),timestamp,nonce,signature
4、客户端服务器或认证服务器进行同样算法的认证即可，另外在获取自定义参数时、应该获取参与了签名的参数。
```

## 二、（client服务器通过）认证服务器主动向clientB服务器传送资源流程
```
     +----------+                               
     | Resource |                               
     |   Owner  |             +----------+                         
     |          |             |          |>------(1)-----------   
     +----------+             |  Client  |                    |   
          ^   ----------(A)--<|          |<------(2)----      |   
          |   |               +----------+             |      |   
         (B)  v                                        ^      v   
     +----|-----+                                 +---------------+
     |         -+----(A)--- transmit state   ---->|               |
     |  User-   |                                 | Authorization |
     |  Agent  -+----(B)-- User authenticates --->|     Server    |
     |          |                                 |               |
     |         -+----(C)-- Authorization Code ---<|               |
     +-|----|---+                                 +---------------+
            |                                         ^      v
           (C)                                        |      |
            |                                         |      |
            v                                         |      |
     +---------+                                      |      |
     |         |>---(a)-- Client Identifier  ---------'      |
     | ClientB |          & secret                           |
     |         |                                             |
     |         |<---(b)----- Access Token -------------------'
     +---------+
       ^    v   
       |    |   
      (E)  (D)  
       |    |   
       ^    v   
     +---------+
     |         |
     | Resource|
     |  Server |
     |         |
     +---------+
```
### 说明：
1. Client之前已经获取了资源，已有：refresh\_coder\_token,ClientB Identifier。
2. 在步骤(1)与(2)中，Client通过transmit\_fun、transmit\_data、refresh\_coder\_token与ClientB Identifier获取redirect\_uri，并重定向的认证服务器。
3. 步骤(A)中包括一个transmit\_state，由认证服务器生成的redirect\_uri中携带。
4. 之后ClientB的步骤同（一）中的后续流程。
5. 特别地，步骤(C)中不含state，包含from、transmit\_fun,transmit\_data、code、type

### 特别说明：
* 步骤(A)与(C)需要对参数进行签名,且包括参数名，(A)与(C)由认证服务器进行签名，具体说明见（一）的特别说明。
* from由认证服务器传给ClientB，用于标识Client。
* transmit\_fun表示Client调用ClientB的功能说明，transmit\_data为传递的数据。