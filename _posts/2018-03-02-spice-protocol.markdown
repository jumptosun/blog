---
layout: post
title:  "spice协议翻译"
date:   2018-03-02 10:40:24 +0800
categories: spice
---

根据官方 spice 协议文档翻译 [https://www.spice-space.org/static/docs/spice_protocol.pdf](https://www.spice-space.org/static/docs/spice_protocol.pdf)。文档比较老，仅做参考。最新细节阅读 spice-protocol 项目。

第一、二章参照博客　[http://blog.csdn.net/sjin_1314/article/details/41720159](http://blog.csdn.net/sjin_1314/article/details/41720159)。
如有侵权请发邮件至 jumptosun@live.com 


# 1 SPICE协议简介

SPICE协议定义了一组协议消息来访问、控制、和接收通过网络从远程计算机设备（如：键盘、视频、鼠标）的操作，并回复发送输出。控制设备既可以在客户端，也可以在服务端。另外，协议定义了一组支持远程服务器从一个网络地址迁移到另一个网络地址。加密传输数据，有一个例外，在选择加密方法上比较灵活。SPICE使用简单的消息传递和不依赖于任何RPC标准或特定的传输层。
SPICE通信会话分为多种沟通通道道（每个通道针对一个远程设备）为了有能力控制通信和执行根据通道类型的消息（如QOS加密），并在运行时添加和删除通道（支持SPICE自定义）。
- 1）  主通道作为主要的SPICE会话通道
- 2）  显示通道接收远程显示更新
- 3）  输入通道发送鼠标和键盘事件
- 4）  光标通道接收指针形状和位置
- 5）  播放通道接收音频流
- 6）  录音通道发送客户端音频输入。
随着协议的发展将添加更多的通道类型，SPICE还定义了一组协议同步信道在远程站点上执行。

# 2  普通协议定义
# 2.1 字节顺序
除非另有规定，所有数据结构封装和字节位顺序是小端字节序格式。

# 2.2 数据类型

- a) UINT8 – 8 bits unsigned integer
- b) INT16 – 16 bits signed integer
- c) UINT16 – 16 bits unsigned integer
- d) UINT32 – 32 bits unsigned integer
- e) INT32 - 32 bits signed integer
- f) UINT64 – 64 bits unsigned integer
- g) ADDRESS - 64 bits unsigned integer, value is the offset of the addressed data from the beginning of spice protocol     message body (i.e., data following RedDataHeader orRedSubMessage).  

    从 message( 2.12 ) body 开始的地址偏移.

- h) FIXED28_4 – 32 bits fixed point number. 28 high bits are signed integer. Low 4 bits is
    unsigned integer numerator of a fraction with denominator 16.   
    
    高28位有符号整数，低4位为分母为16的分数。

- i) POINT
   INT32 x
   INT32 y

- j) POINT16
   INT16 x
   INT16 y

- k) RECT 矩形
   INT32 top
   INT32 left
   INT32 bottom
   INT32 right

- l) POINTFIX
   FIXED28_4 x
   FIXED28_4 y

## 2.3 协议中 magic number
    RED_MAGIC = { 0x52, 0x45, 0x44, 0x51}

## 2.4 协议版本
协议版本定义为两个UINT32值，主要协议版本和次要协议版本。服务器和客户端拥有相同的主要版本，为了保持兼容性不管次要版本号。    

    RED_VERSION_MAJOR = 1
    RED_VERSION_MINOR = 0

## 2.6 通道类型-UINT8

    RED_CHANNEL_MAIN    = 1
    RED_CHANNEL_DISPLAY = 2
    RED_CHANNEL_INPUTS  = 3
    RED_CHANNEL_CURSOR  = 4
    RED_CHANNEL_PLAYBACK= 5
    RED_CHANNEL_RECORD  = 6

## 2.11 通道链接: 建立一个通道连接

- a) 连接过程
通道连接过程是由客户端发起。客户端发送RedLinkMess，作为回应服务器发送RedLinkReply。当客户端接收到RedLinkReply时，它检查返回错误代码，如果没有错误它会加密公钥密码在RedLinkReply并将其发送到服务器。服务器接收密码并将链接的结果发给客户端。客户端检查连接的结果，如果结果为RED_ERROR_OK，建立一个有效的链接。除了RED_CHANNEL_MAIN以外其他的通道类型只允许在客户端主动连接RED_CHANNEL_MAIN。仅仅在RED_CHANNEL_MAIN通道被允许后，才会建立起与远程服务器的会话。

- b) Ticketing
Ticketing是SPICE协议的一种实现机制目的是为了确保只允许有授权的来源连接。为了确保这种机制，Ticketing是一个在SPICE服务器端有密码和有效时间段的组成。时间过期后，这个Ticketing也就过期了。这个Ticketing是加密的。为了使用加密，服务器生成一个1024位的RSA密钥并向客户端发送公钥（通过RedLinkInfo）。客户端使用这个公钥加密密码并将其发回服务器（RedLinkMess之后）。服务器解密密码，比较它的ticketing和确保在允许的时间框架内。

- c) RedLinkMess定义    
```
    typedef structSPICE_ATTR_PACKED SpiceLinkHeader {  
        uint32_t magic;          //必须为RED_MAGIC  
        uint32_t major_version;  //客户端主版本号  
        uint32_t minor_version;  //客户端次版本号  
        uint32_t size;           //后续数据的大小  
    }SpiceLinkHeader;  

    typedef structSPICE_ATTR_PACKED SpiceLinkMess {  
        uint32_t connection_id;    //针对一个新的会话（例如RED_CHANNEL_MAIN）这个数值被设置为0，服务器将分配会话ID并在RedLinkMess消息中发给客户端。其他通道类型。使用分配的会话ID。  
        uint8_t channel_type;      //通道类型  
        uint8_t channel_id;        //通道ID，同一个ID可能有多个通道。  
        uint32_t num_common_caps;  //普通客户端通道功能词数量  
        uint32_t num_channel_caps; //特殊客户端通道功能词数量  
        uint32_t caps_offset;      //这个结构体的大小。后续还有能力集数值  
    } SpiceLinkMess;  
```
- d) RedLinkReply定义    

       typedef structSPICE_ATTR_PACKED SpiceLinkHeader {  
          uint32_t magic;           //必须为RED_MAGIC  
           uint32_t major_version;  //服务器主版本号  
           uint32_t minor_version;  // 服务器次版本号  
           uint32_t size;           //后续数据的大小  
       }SpiceLinkHeader;  

       typedef structSPICE_ATTR_PACKED SpiceLinkReply {  
           uint32_t error;            //版本号协商结果，错误码  
           uint8_t pub_key[SPICE_TICKET_PUBKEY_BYTES];//公钥  
           uint32_t num_common_caps;  //普通服务器通道功能词数量  
           uint32_t num_channel_caps; //普通服务器通道功能词数量  
           uint32_t caps_offset;      //偏移量  
       }SpiceLinkReply;  


- e) 加密密码
客户端使用公钥加密密码发送回服务器

- f) 验证结果
接收验证结果

## 2.12 Protocol message definition
在连接建立后，所有的消息传输都遵循下面的格式。开始于RedDataHeader描述一个主要消息和一个可选的子消息。在实际传输中使用SpiceMiniDataHeader。

- a) SpiceMiniDataHeader    
```
    typedef structSPICE_ATTR_PACKED SpiceMiniDataHeader {  
        uint16_t type; //消息类型，根据通道类型选择各自的处理函数。  
        uint32_t size;//数据大小  
    } SpiceMiniDataHeader;  
```
- b)  SpiceDataHeader    
      在实际传输中并未用到这个数据头而使用上面的mini_header    
```
    typedef structSPICE_ATTR_PACKED SpiceDataHeader {  
        uint64_t serial; //通道内，消息的序列号。开始于1，后续逐渐增加  
        uint16_t type; //消息类型  
        uint32_t size; //消息体的大小，如果sublist不是0，则sub_list为消息的实际大小  
        uint32_t sub_list;//offset to SpiceSubMessageList[]//后续数据的大小  
    }SpiceDataHeader;  
 ```

- c)  SpiceSubMessage    
```    
    typedef structSPICE_ATTR_PACKED SpiceSubMessage {  
        uint16_t type;  
        uint32_t size;  
    }SpiceSubMessage;  
```
- d) SpiceSubMessageList
```
    typedef structSPICE_ATTR_PACKED SpiceSubMessageList {  
        uint16_t size;  
        uint32_t sub_messages[0]; //offsets toSpicedSubMessage  
    }SpiceSubMessageList;  
```

##  2.13  Common messages and messaging naming convention
通用消息类型，以及其命名规则。    
消息和消息体结构类型前缀描述了消息的来源。具体如下：    
服务器 --> 客户端：RED    
客户端 --> 服务器：REDC    
    
这个命名是SPICE协议文档中的描述，在实际源码中，可能是下面的定义：    
服务器 --> 客户端：SPICE_MSG    
客户端 --> 服务器：SPICE_MSGC    

##  2.14  server message，定义对所有的通道都适用。
代码中一般code为SPICE_MSG

    RED_MIGRATE         = 1 // 迁移
    RED_MIGRATE_DATA    = 2
    RED_SET_ACK         = 3
    RED_PING            = 4
    RED_WAIT_FOR_CHANNELS = 5
    RED_DISCONNECTING   = 6
    RED_NOTIFY          = 7
    RED_FIRST_AVAIL_MESSAGE = 101

7~101保留

## 2.15  client message 类型，定义对所有的通道都适用。
代码中一般code为SPICE_MSGC 

    REDC_ACK_SYNC   = 1
    REDC_ACK        = 2
    REDC_PONG       = 3
    REDC_MIGRATE_FLUSH_MARK = 4
    REDC_MIGRATE_DATA= 5
    REDC_DISCONNECTING = 6
    REDC_FIRST_AVAIL_MESSAGE = 101

6~101保留

## 2.16  Messages acknowledgment （消息确认）
        server                            client        
          |-----------RED_SET_ACK----------->|         // 服务端发送多少个消息，客户端回复一个确认     
          |<---------REDC_ACK_SYNC-----------|        // 客户端对RED_SET_ACK的确认    
                              :  
                              :  
          |<---------REDC_ACK----------------|    


## 2.17 Channel migration (通道迁移)
Spice协议支持 Spice 服务器间的迁移。 迁移过程通过 main 通道来传输 message。 
将这些迁移过程中涉及到的服务器称为源端(source)和目的端(destination)。 main   
通道用于启动和控制迁移过程。 以下描述了实际的信道迁移过程。  


## 2.19 Channel synchronization (通道同步)
SPICE协议提供了消息在客户端执行同步机制。服务器发送RED_WAIT_FOR_CHANNELS消息包含一个频道列表的消息等待（RedWaitForChannels）。
SPICE客户端等待完成这个列表的所有消息在其他的所有消息执行之前。

## 2.20 Disconnect reason (链接关闭原因)
UINT64 time_stamp – time stamp of disconnect action on the server.
UINT32 reason – disconnect reason, RED_ERROR_?

    typedef structSpiceMsgDisconnect {  
        uint64_t time_stamp; //服务器或客户端断开的时间戳  
        uint32_t reason;     // SPICE_ERR_?  
    }SpiceMsgDisconnect;  

## 2.21 Server notification
消息按严重性和可见性(visibility ?)分类。可以隐式的决定用户后续显示消息的方式。  
例如高可见性的通知将触发消息框，低可见度的通知将会被指向日志。  

    typedef structSpiceMsgNotify {  
        uint64_t time_stamp; //这个消息在server端的时间戳  
        uint32_t severity;    //消息的严重程度  
        uint32_t visibilty;    //消息的可见程度  
        uint32_t what;                     //消息的类别，错误、警告、帮助等等  
        uint32_t message_len;//消息长度  
        uint8_t message[0];   //消息数据  
    }SpiceMsgNotify;  


# 3. Main Channel definition 主通道定义

##  3.1. Server messages

    RED_MAIN_MIGRATE_BEGIN = 101
    RED_MAIN_MIGRATE_CANCEL = 102
    RED_MAIN_INIT = 103
    RED_MAIN_CHANNELS_LIST = 104
    RED_MAIN_MOUSE_MODE = 105
    RED_MAIN_MULTI_MEDIA_TIME = 106
    RED_MAIN_AGENT_CONNECTED = 107
    RED_MAIN_AGENT_DISCONNECTED = 108
    RED_MAIN_AGENT_DATA = 109
    RED_MAIN_AGENT_TOKEN = 110

## 3.2. Client messages  

    REDC_MAIN_RESERVED = 101
    REDC_MAIN_MIGRATE_READY = 102
    REDC_MAIN_MIGRATE_ERROR = 103
    REDC_MAIN_ATTACH_CHANNELS = 104
    REDC_MAIN_MOUSE_MODE_REQUEST = 105
    REDC_MAIN_AGENT_START = 106
    REDC_MAIN_AGENT_DATA = 107
    REDC_MAIN_AGENT_TOKEN = 108

## 3.3 Migration control  
spice Migration 使用 main channel 消息来控制。 spice server
通过发送 RED_MAIN_MIGRATE_BEGIN 消息来启动迁移过程。 一旦
客户端已完成其迁移准备过程(pre-migrate)，向 server 发送
REDC_MAIN_MIGRATE_READY 消息来通知 server。 迁移准备过程
出错的情况下，客户端则发送 REDC_MAIN_MIGRATE_ERROR。
一旦服务器收到 REDC_MAIN_MIGRATE_READY 即可以开始迁移过程。
服务器可以发送 RED_MAIN_MIGRATE_CANCEL 以指示客户端取消迁移
过程。

- a) RED_MAIN_MIGRATE_BEGIN    

        channel-main.c :2283
        #ifdef __GNUC__
        typedef struct __attribute__ ((__packed__)) OldRedMigrationBegin {
        #else
        typedef struct __declspec(align(1)) OldRedMigrationBegin {
        #endif
           uint16_t port;
           uint16_t sport;
           char host[0];
        } OldRedMigrationBegin;

- b) RED_MAIN_MIGRATE_CANCEL VOID    
- c) RED_MAIN_MIGRATE_READY VOID    
- d) RED_MAIN_MIGRATE_ERROR VOID   

## 3.4 Mouse modes　鼠标模式
spice 协议指定了两种鼠标模式,客户端模式和服务器模式。
在客户端模式下，affective 鼠标是客户端鼠标：客户端发送鼠标位置显示，服务器发送鼠标
形状显示。    
在服务器模式下，客户端发送相对鼠标移动，服务器发送位置和形状显示。    
鼠标模式控制用主通道传送。    

- a) Modes    
 RED_MOUSE_MODE_SERVER = 1    
 RED_MOUSE_MODE_CLIENT = 2    

- b) RED_MAIN_MOUSE_MODE     
```
    typedef struct RedMouseModePipeItem {
        RedPipeItem base;
        SpiceMouseMode current_mode;
        int is_client_mouse_allowed;
    } RedMouseModePipeItem;
```
- c) REDC_MAIN_MOUSE_MODE_REQUEST
    客户端返回选择的鼠标模式
    UINT32 – requested mode, one of RED_MOUSE_MODE_?

## 3.5 Main channel init message
Spice 服务器必须发送 RedInit 作为第一个发送的消息，并且不允许
在任何其他时候发送 RedInit 。

- a) RED_MAIN_INIT, RedInit
```
    main-channel-client.c col:86:
    typedef struct RedInitPipeItem {
        RedPipeItem base;
        int connection_id;
        int display_channels_hint; // 预计有多少 display channel, 0 不能作为有效值被设置
        int current_mouse_mode;
        int is_client_mouse_allowed; //　与current_mouse_mode 的顺序，和文档上相反
        int multi_media_time;
        int ram_hint;   // LZ 压缩字典大小
    } RedInitPipeItem;
```

## 3.6 Server side channels notification 服务器端通道通知
为了有能力动态地到加入服务器端的渠道，Spice 协议设计了
RED_MAIN_CHANNELS_LIST 消息。 该消息通知客户服务器端的可用通道。 
响应这个消息，客户端可以决定连接新的可用通道。
服务器首先要收到 REDC_MAIN_ATTACH_CHANNELS ,才能发送
RED_MAIN_CHANNELS_LIST。

- a) RED_MAIN_CHANNELS_LIST, RedChannels 
UINT32 num_of_channels -- 列表中 channel 的数量
RedChanneID[] channels -- channel 的列表

- b) RedChannelID
    代码中未找到结构体定义
    uint8 type 
    uint8 id

## 3.7 Multimedia time 多媒体时间

Spice 定义了用于设置多媒体时间以同步视音频的消息
流。 更新多媒体时间有两种方法。 
第一种方法使用在播放频道上到达数据携带的时间戳。
第二种方法使用主频道 RED_MAIN_MULTI_MEDIA_TIME 。
active playback channel(没有音频通道) 存在时使用
第二种方法。

- a) RED_MAIN_MULTI_MEDIA_TIME, UINT32
    UINT32 -- 多媒体时间戳

## 3.8 Spice agent

Spice协议定义了一组信息, 用于spice客户端和远程服务器上的
spice client agent 双向通信。 spice 只提供了一个通信通道，
实际传输的数据内容是不透明的协议。 此通道可用于各种用途，
例如客户端剪贴板共享，认证和显示配置。

Spice 客户端接收远程站点的代理连接请求，通过 RED_MAIN_INIT 中携带的信息 或
RED_MAIN_AGENT_CONNECTED 消息来交互。远程代理断线用 RED_MAIN_AGENT_DISCONNECTE
来通知 。使用双向令牌机制来防止主通道被 agent message 堵塞（例如，代理不再消费数据）。
每一边发送都不允许超出另一方令牌所分配消息的数量。使用 RED_MAIN_INIT 消息来初始化令牌的分配，
并使用 RED_MAIN_AGENT_TOKEN 完成令牌的进一步分配。REDC_MAIN_AGENT_START　中携带了服务器令牌初始计数。
这个消息必须是客户端发送到服务器的第一个消息。使用 REDC_MAIN_AGENT_TOKEN  完成服务器端的更多的分配。
实际数据包的传送使用RED_MAIN_AGENT_DATA 和 REDC_MAIN_AGENT_DATA。

消息定义待翻译

# 4. Inputs channel definition
控制鼠标和键盘。

## 4.1. Client messages

REDC_INPUTS_KEY_DOWN    = 101    
REDC_INPUTS_KEY_UP      = 102    
REDC_INPUTS_KEY_MODIFAIERS  = 103    

REDC_INPUTS_MOUSE_MOTION    = 111    
REDC_INPUTS_MOUSE_POSITION  = 112    
REDC_INPUTS_MOUSE_PRESS     = 113    
REDC_INPUTS_MOUSE_RELEASE   = 114    

## 4.2. Server messages

RED_INPUTS_INIT = 101    
RED_INPUTS_KEY_MODIFAIERS   = 102    

RED_INPUTS_MOUSE_MOTION_ACK = 111    

## 4.3. Keyboard messages

spice 支持发送键盘事件和键盘状态灯(大小写，小键盘)同步。 客户端使用
REDC_INPUTS_KEY_DOWN 和 REDC_INPUTS_KEY_UP 发送键盘事件。 
键值用PC AT扫描码表示（见KeyCode）。 键盘灯状态
由服务器发送 RED_INPUTS_KEY_MODIFAIERS 或
客户端发送 REDC_INPUTS_KEY_MODIFAIERS 消息
来完成键盘状态灯同步。RED_INPUTS_INIT 也可能携带键盘灯状态，
该消息必须作为服务器和客户端之间的第一个消息,
并且不能在其他时间点出现。

- a) Keybord led bits 键盘指示灯状态比特位
    RED_SCROLL_LOCK_MODIFIER    = 1    
    RED_NUM_LOCK_MODIFIER       = 2    
    RED_CAPS_LOCK_MODIFIER      = 4    

- b) RED_INPUTS_INIT, UINT32
    UINT32  led bits 的组合

- c) RED_INPUTS_KEY_MODIFAIERS, UINT32 同上

- d) REDC_INPUTS_KEY_MODIFAIERS, UINT32 同上

- e) KeyCode
     UINT8[4] - PC AT　扫描码

- f) REDC_INPUTS_KEY_DOWN, KeyCode
- g) REDC_INPUTS_KEY_UP, KeyCode

## 4.4. Mouse messages
Spice 支持两种鼠标操作模式：客户端鼠标和服务器鼠标（参见 3.4）。
在服务器鼠标模式下，客户端发送鼠标移动消息（即，REDC_INPUTS_MOUSE_MOTION）。
在服务器鼠标模式下，客户端发送鼠标模式位置消息（即，REDC_INPUTS_MOUSE_POSITION）。
位置消息指定了客户端鼠标在显示器上的位置和显示通道的ID(RedLinkMess.channel_id)。
为了防止频繁的发送鼠标移动和位置的事件，服务器每收到 RED_MOTION_ACK_BUNCH　消息，
都会恢复 RED_INPUTS_MOUSE_MOTION_ACK消息。这种机制允许客户端跟踪服务器的消息消费频率，
并更改事件推送策略。发送 REDC_INPUTS_MOUSE_PRESS 和 REDC_INPUTS_MOUSE_RELEASE 消息,
来更新鼠标按钮状态。

- a) Red Button ID, 鼠标按钮 ID    
REDC_MOUSE_LBUTTON = 1 鼠标左键    
REDC_MOUSE_MBUTTON = 2 鼠标中建    
REDC_MOUSE_RBUTTON = 3 鼠标右键    
REDC_MOUSE_UBUTTON = 4 向上滚动    
REDC_MOUSE_DBUTTON = 5 向下滚动    

- b) Buttons masks 按钮掩码    
REDC_LBUTTON_MASK = 1,    
REDC_MBUTTON_MASK = 2,    
REDC_RBUTTON_MASK = 4,    

- c) RED_MOTION_ACK_BUNCH = 4,　移动确认    

- d) REDC_INPUTS_MOUSE_MOTION, RedcMouseMotion 鼠标位移   
INT32 dx – 在x轴上移动了多少像素    
INT32 dy - 在y轴上移动了多少像素    
UINT32 buttons_state – 按钮掩码的组合。代表当前鼠标按钮的状态（按下置1,抬起置0）。    

- e) REDC_INPUTS_MOUSE_POSITION, RedcMousePosition    
UINT32 x – 鼠标x轴位置    
UINT32 y - 鼠标x轴位置    
UINT32 buttons_state - 按钮掩码的组合。代表当前鼠标按钮的状态（按下置1,抬起置0）。    
UINT8 display_id – 鼠标所在显示通道的ID    

- f) REDC_INPUTS_MOUSE_PRESS, RedcMousePress 鼠标按钮按压状态    
UINT32 button_id – REDC_MOUSE_?BUTTON 之一    
UINT32 buttons_state - 按钮掩码的组合。代表当前鼠标按钮的状态（按下置1,抬起置0）。    

- g) REDC_INPUTS_MOUSE_RELEASE, RedcMouseRelease    鼠标释放     
UINT32 button_id – REDC_MOUSE_?BUTTON 之一  
UINT32 buttons_state - 按钮掩码的组合。代表当前鼠标按钮的状态（按下置1,抬起置0）。    

# 5 Display channel definition
Spice协议定义了一组消息，用于支持远程客户端的渲染显示。 协议支持绘制图形基元（例如线，图像）
和视频流。 协议还支持在客户端上缓存图像和调色板。 spice display channel 支持多种图像压缩方式，
以减少网络带宽占用。

## 5.1  Server message    
RED_DISPLAY_MODE = 101
RED_DISPLAY_MARK = 102
RED_DISPLAY_RESET =103
RED_DISPLAY_COPY_BITS = 104

RED_DISPLAY_INVAL_LIST = 105
RED_DISPLAY_INVAL_ALL_IMAGES = 106
RED_DISPLAY_INVAL_PALETTE = 107
RED_DISPLAY_INVAL_ALL_PALETTES = 108

RED_DISPLAY_STREAM_CREATE = 122
RED_DISPLAY_STREAM_DATA = 123
RED_DISPLAY_STREAM_CLIP = 124
RED_DISPLAY_STREAM_DESTROY = 125
RED_DISPLAY_STREAM_DESTROY_ALL = 126

RED_DISPLAY_DRAW_FILL = 302
RED_DISPLAY_DRAW_OPAQUE = 303
RED_DISPLAY_DRAW_COPY = 304
RED_DISPLAY_DRAW_BLEND = 305
RED_DISPLAY_DRAW_BLACKNESS = 306
RED_DISPLAY_DRAW_WHITENESS = 307
RED_DISPLAY_DRAW_INVERS = 308
RED_DISPLAY_DRAW_ROP3 = 309
RED_DISPLAY_DRAW_STROKE = 310
RED_DISPLAY_DRAW_TEXT = 311
RED_DISPLAY_DRAW_TRANSPARENT = 312
RED_DISPLAY_DRAW_ALPHA_BLEND = 313

## 5.2 Client message 
REDC_DISPLAY_INIT = 101

## 5.3  Operation flow 工作流程
spice 服务器向客户端发送RED_DISPLAY_MODE 来指定当前的绘制区域大小和格式。
作为响应，客户端创建一个绘图区域,用于渲染后续服务器发来的所有渲染命令。 
客户只有在收到 mark (i.e., RED_DISPLAY_MARK) 命令后之后才暴露新的远程显
示区域内容。 服务器可以发送 RED_DISPLAY_RESET 命令来指示客户端删除其绘
制区域和调色板缓存。 仅当客户端没有激活的绘图区域时才允许发送 mode 消息。 
仅当客户端存在激活的绘图区域时才允许发送 reset 消息。在 mode 和 reset 
消息之间，只允许出现一次 mark 消息。

draw 命令, copy bit 命令和stream 命令只有存在有效绘画区域时,才能被发送。

当 channel 建立时,客户端可以选择性的发送 init (REDC_DISPLAY_INIT) 消息。
发送 REDC_DISCONNECTING   是为了开启图像缓存，和全局的压缩字典。该消息只能
发送一次。

调色板的缓存由服务器管理。插入新的调色板缓存作为渲染命令的一部分被发送。
删除调色板信息发送 RED_DISPLAY_INVAL_LIST 或 ???。通过发送 
RED_DISPLAY_INVAL_ALL_IMAGES 或  RED_DISPLAY_INVAL_ALL_PALETTES 消息来
重置客户端缓存。

## 5.4. 绘画区域控制
- a) RED_DISPLAY_MODE, RedMode
    UINT32 width -- 绘画区域宽度
    UINT32 height -- 绘画区域高度
    UINT32 depth -- 色彩深度, 16或32

- b) RED_DISPLAY_MARK, VOID

- c) RED_DISPLAY_RESET, VOID

## 5.5. Raster operation descriptor 光栅操作描述符

以下定义了一组可由于描述光栅操作的标志位,可用于图像，画笔，目标和渲染过程中。
根据这些标志的组合就可以确定渲染期间需要执行的操作。 在下面的渲染命令的定义中
ROPD 即 rop_descriptor。
    
- ROPD_INVERS_SRC = 1
  原图像在渲染之前需要转换。

- ROPD_INVERS_BRUSH = 2
  笔刷在渲染之前需要转换。

- ROPD_INVERS_DEST = 4
  目标区域需要变换。

- ROPD_OP_PUT = 8
  复制操作

- ROPD_OP_OR = 16
  或操作

- ROPD_OP_AND = 32
  与操作

- ROPD_OP_XOR = 64
  异或操作

- ROPD_OP_BLACKNESS = 128
  目标区域的像素需要被替换为黑色

- ROPD_OP_WHITENESS = 256
  目标区域的像素需要被替换为白色

- ROPD_OP_INVERS = 512
  目标区域的像素需要被转换

- ROPD_INVERS_RES = 1024
  Result of the operation needs to be inverted
  操作的结果需要转换

- OP_PUT, OP_OR, OP_AND, OP_XOR, OP_BLACKNESS, OP_WHITENESS, 
  OP_INVERS 操作之间是互斥的

- OP_BLACKNESS, OP_WHITENESS, and OP_INVERS 操作之间是是互斥的

## 5.6 原始光栅图像
以下部分描述了spice 原始光栅图像 ( pixmap, 像素图 ).
pixmap 属于 spice 协议用来传输图像的一种方式。

color table: 色彩表
palette: 调色板

- a) Pixmap format type 像素图格式类型
     
     PIXMAP_FORMAT_1BIT_LE = 1
     每一像素占1bit, bits序为小端。每个像素的值代表色彩表中的索引，
     色彩表大小为2。

     PIXMAP_FORMAT_1BIT_BE = 2
     每一像素占1bit, bits序为大端。每个像素的值代表色彩表中的索引，
     色彩表大小为2。

     PIXMAP_FORMAT_4BIT_LE = 3 
     每一像素占4bit, 字节内顺序为小端。每个像素的值代表色彩表中的索引，
     色彩表大小为16。

     PIXMAP_FORMAT_4BIT_BE = 4
     每一像素占4bit, 字节内顺序为大端。每个像素的值代表色彩表中的索引，
     色彩表大小为16。

     PIXMAP_FORMAT_8BIT    = 5
     每一像素占8bit。每个像素的值代表色彩表中的索引，色彩表大小为256。

     PIXMAP_FORMAT_16BIT   = 6
     像素格式为 RGB555 

     PIXMAP_FORMAT_24BIT   = 7
     像素格式为 RGB888

     PIXMAP_FORMAT_32BIT   = 8
     像素格式为 RGB888

     PIXMAP_FORMAT_RGBA    = 9
     像素格式为 ARGB8888, A代表透明度

- b) Palette 调色板
     UINT64 id - 唯一的调色板ID
     UINT16 table_size - 色彩表中的存储数量
     UINT32[] color_table - 表中每一项为 RGB555 或者 RGB888，具体依据
     当前的渲染模式。

- c) 像素图标志

     PIXMAP_FLAG_PAL_CACHE_ME = 1
     指示客户端添加调色板到缓存

     PIXMAP_FLAG_PAL_FROM_CACHE = 2
     指示客户端查找缓存中的调色板

     PIXMAP_FLAG_TOP_DOWN = 4
     像素扫描行为从上到下排列 ( 如: 第0行为图像最上面一行 )

- d) 像素图

     UINT8 format - PIXMAP_FORMAT_? 之一    
     UINT8 flags - PIXMAP_FLAG_? 的组合    
     UINT32 width - 宽度    
     UINT32 height - 高度    
     UINT32 stride - 每一行像素存储需要多少字节。第 n 行到第 n+1 行的字节位移。    
     
     union {    
        ADDRESS palette - 调色板起始地址。不存在要置0。    
        UINT64 palette_id - 调色板ID, FLAG_PAL_FROM_CACHE 被设置才有效    
     }    
     ADDRESS data - pixmap 的开始地址

## 5.7 LZ with palette

## 5.8 Spice image
