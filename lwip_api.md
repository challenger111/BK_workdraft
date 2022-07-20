# lwip的API和移植
## 移植的概念
   移植，简单地来讲，就是诸如操作系统、代码库的跨平台使用。   
  >移植的需求产生：  
  > + 代码的生产者无法预见性地为代码适配特定的平台，或者是无法为主流平台做相应地适配工作（意味着大量的，繁琐的操作）。  
  > + 代码的使用者希望根据自身的工程环境与需求的功能，灵活地使用代码。
  
## lwip的三种移植方式及其对应的API
  lwip提供了三种编程接口，分别为raw/callback API、 sequential API、 socket API。其中：
  > + raw/callback API无需操作系统的支持，移植工作比较简单。
  > + sequential API和sockte API分别以前一种移植为基础，需要操作系统的支持，除此之外还要完成 操作系统模拟层的编写。
  
  需要注意的是，虽然以上提及的三种API作者都已经实现其基本功能（或者框架），但是如何与硬件层对接、如何在系统中平稳地  
  运行，仍然是我们需要关心的内容。
  ### 一、raw/callback API
  该API移植工作的主要内容是网卡的配置以及实现通过网卡接发数据地功能。
>***若干头文件的定义：cc.h lwipopts.h***   
>>```C
>>CC.h
>>......
>>#ifndef LWIP_SIMPLE_TYPES
>>#define LWIP_SIMPLE_TYPES 1
>>typedef uint8_t   u8_t;
>>typedef int8_t    s8_t;
>>typedef uint16_t  u16_t;
>>typedef int16_t   s16_t;
>>typedef uint32_t  u32_t;
>>typedef int32_t   s32_t;
>>#endif
>>......
>>```
>>显而易见地，该文件描述了协议栈内部使用的数据类型和一些宏定义。  

>>```C
>>lwipopts.h
>>......
>>#define NO_SYS                        1           //无操作系统
>>#define LWIP_SOCKET                   0           //不编译socket API
>>#define LWIP_NETCONN                  0           //不编译sequential API
>>#define MEM_ALLIGNMENT                4           //系统对齐字节
>>#define MEM_SIZE                      1024*32     //内存堆大小
>>#endif
>>......
>>```
>>与CC.h不同的是，在lwip源文件中其实存在opt.h描述了lwipopts.h中所描述值的默认值，如果我们不在lwipopts中对值进行改  
>>动，则会保持默认。

>***ethernetif.c中函数的改写***  
>  
>  ethernetif.c实际上起到一个与网卡对接的作用，它主要负责初始化网卡和管理数据包在网卡上的收发。  
以下是其中已经写好框架的五个函数的具体职责
>>`static void low_level_init(struct netif *netif)`  
>> low_level_init为网卡初始化函数，主要完成网卡复位以及参数初始化，同时初始化netif部分字段。
>>`static err_t low_level_output(struct netif *netif, struct pbuf *p)`    
>> low_level_output为网卡数据包发送函数，它将会把数据包以pbuf的形式发送出去。
>>`err_t ethernetif_init(struct netif *netif)`   
>> ethernetif_init是对接上层网络接口结构的函数，它会初始化netif部分字段并调用low_level_init.  
>>`static struct pbuf *low_level_input(struct netif *netif)`    
>> low_level_input是网卡数据包接收函数，接收到的数据将被包装为pbuf形式。
>>`void ethernetif_input(int iface, struct pbuf *p)`     
>> ethernetif_input是数据包递交函数，负责将数据包递交至api层处理。
>>
>>
>>
>>
  
