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

>***netif结构体解析***<br/><br>
>netif网络接口是这一层的API中重要的结构体，它实际上抽象了网卡，方便与应用层的对接。  
>>```C
>>struct netif
>>{
>>	struct netif *next;
>>	/* IP地址、子网掩码、默认网关配置 */
>>	ip_addr_t ip_addr;
>>	ip_addr_t netmask;
>>	ip_addr_t gw;
>>	
>>	//从网卡上取得一个数据包
>>	err_t (* input)(struct pbuf *p, struct netif *inp);
>>	
>>	//IP层向网卡发送一个数据包
>>	err_t (* output)(struct netif *netif, struct pbuf *p,struct ip_addr *ipaddr);   
>>
>>	//ARP发送数据包
>>	err_t (* linkoutput)(struct netif *netif, struct pbuf *p);
>>	
>>	u8_t hwaddr[NETIF_MAX_HWADDR_LEN];      //MAC 地址
>>	u16_t mtu;             					   //一次可以传送的最大字节数
>>	char name[2]; 							      //网络接口使用的设备驱动类型的种类
>>	u8_t num;        						      //用来标示使用同种驱动类型的不同网络接口
>>	......
>>}
>>```
>><br/>
>>结构体过于抽象，我们来看一个初始化的具体例子  
>>
>>```C
>>struct netif netif_demo;			      //创建一个虚拟网卡的实例                         
>>struct ip_addr ipaddr, netmask, gw;	//用于存放地址
>>
>>/*给三个地址赋值*/
>>IP4_ADDR(&gw, 192,168,0,1);   
>>IP4_ADDR(&ipaddr, 192,168,0,60); 
>>IP4_ADDR(&netmask, 255,255,255,0);
>> 
>>netif_init(); //初始化存放netif实例的链表
>>
>>/*完善netif_demo的字段，并将其添加至链表中。*/   
>>netif_add(&netif_demo, &ipaddr, &netmask, &gw, NULL, ethernetif_init, tcpip_input);                                          
>>```
>>以上代码段展示了一个名为netif_demo的网卡实例被加载到链表中的过程，其中ethernetif_init、tcpip_input分别指明了  
>>函数指针init和input的具体值。



>***ethernetif.c中函数的改写***  
>  
>  ethernetif.c实际上起到一个与网卡对接的作用，它主要负责初始化网卡和管理数据包在网卡上的收发。  
以下是其中已经写好框架的五个函数的预定职责
>> `static void low_level_init(struct netif *netif)`
>> ```C
>>.....
>>netif->hostname = (char*)&wlan_name[vif_entry->index];          //初始化接口名
>>netif->hwaddr_len = ETHARP_HWADDR_LEN;                          //MAC硬件地址长度      
>>os_memcpy(netif->hwaddr, macptr, ETHARP_HWADDR_LEN);            //MAC硬件地址
>>netif->mtu = 1500;                                              //最大传输长度
>>......
>>```  
>> low_level_init为网卡初始化函数，主要完成网卡复位以及参数初始化，同时初始化netif部分字段。<br/>  
>> `static err_t low_level_output(struct netif *netif, struct pbuf *p)`    
>> low_level_output为网卡数据包发送函数，在发送数据的同时还要处理pbuf数据区和网卡发送位数的对齐问题。<br/>  
>> `err_t ethernetif_init(struct netif *netif)`   
>> ethernetif_init是对接上层网络接口结构的函数，它会初始化netif部分字段并调用low_level_init。<br/><br/>
>> `static struct pbuf *low_level_input(struct netif *netif)`
>> ```C
>> ......
>> p=pbuf_alloc(PBUF_RAW,packetlength,PBUF_POOL);
>> if(p!=NULL){
>>    memcpy(p->payload,PDHeader+4,14);
>>    for(q=p;q!=NULL;q=q->next){
>>       payload=q->payload;
>>       len=q->len;
>>       if(q==p){
>>          payload+=14;
>>          len-=14;
>>       }
>>       ......                  //从网卡拷贝数据
>>     }
>>   ......                      //失败处理
>>   return p
>>   ```
>> low_level_input是网卡数据包接收函数，接收到的数据将被包装为pbuf形式。<br/>  
>> `void ethernetif_input(struct netif *netif)`    
>> ```C
>> {
>>    ......            //获取netif   
>>    ethhdr = p->payload;          //获取数据区指针字段
>>
>>    switch (htons(ethhdr->type))  //根据字段指示的类型进行操作
>>    {
>>        /* IP or ARP packet? */
>>    case ETHTYPE_IP:
>>    #if LWIP_IPV6
>>    case ETHTYPE_IPV6:
>>    #endif
>>    case ETHTYPE_ARP:
>>    #if PPPOE_SUPPORT
>>        /* PPPoE packet? */
>>    case ETHTYPE_PPPOEDISC:
>>    case ETHTYPE_PPPOE:
>>    #endif /* PPPOE_SUPPORT */
>>        /* full packet send to tcpip_thread to process */
>>        if (netif->input(p, netif) != ERR_OK)    // ethernet_input
>>        {
>>            LWIP_DEBUGF(NETIF_DEBUG, ("ethernetif_input: IP input error\r\n"));
>>            pbuf_free(p);
>>            p = NULL;
>>        }
>>        break;
>>		
>>    case ETHTYPE_EAPOL:
>>	 	ke_l2_packet_tx(p->payload, p->len, iface);
>>		pbuf_free(p);
>>		p = NULL;
>>        break;
>>		
>>    default:
>>        pbuf_free(p);
>>        p = NULL;
>>        break;
>>}
>>```
>> ethernetif_input是数据包递交函数，通常由硬件层调用,负责将数据包递交至api层处理。  

>***demo展示***  
>下面这个demo主要结构为:
> + 先通过lwip_input_task()来初始化内核和网卡
> + 再通过GetPacket()来接受并递交数据包
> + 最后在循环中运行,周期性检测递交数据包
>```C
>void lwip_init_task(void){
>  struct ip_addr ipaddr,netmask,gw;         //声明三个ip地址变量
>  ......                                    //并且赋值
>  netif_add(&netif_demo, &ipaddr, &netmask, &gw, NULL, ethernetif_init, ethernet_input);
>  netif_set_default(&netif_demo);
>  netif_set_up(&netif_demo);
>}
>**********************
>void Getpacket(void){
>  ......                  //根据网卡硬件属性的不同,使读取位置就绪
>  /*循环读取网卡中的数据包并递交*/
>  while(...){
>     ethernetif_input(&netif_demo)
>     ......
>   }
> }
> *********************
> void lwip_demo(){
>   lwip_init_task();
>   while(1){
>     Getpacket();               //读取一个数据包并递交
>     OSTimeDly(4);              //延迟一段时间,不占用资源
>     /*TCP和ARP定时器的超时恢复*/
>   }
> }
> ```  

### Sequential API
   Sequential API建立在raw/callback API的基础上,包含以下内容：
   + 头文件的修改和定义
   + 操作系统模拟层的实现
   + API的底层工作流程
> 头文件的修改和定义
>>```C
>>lwipopts.h
>>......
>>#define NO_SYS                        1           //无操作系统
>>#define LWIP_SOCKET                   0           //不编译socket API
>>#define LWIP_NETCONN                  0           //不编译sequential API
>>#define MEM_ALLIGNMENT                4           //系统对齐字节
>>#define MEM_SIZE                      1024*32     //内存堆大小
>>......
>>```
>>与raw/callback API类似，主要是有关操作系统，API使能的编译选项。
>>```C
>>sys_arch.h中主要是关于操作系统模拟相关的一些宏、结构体、数据类型的定义。

>操作系统模拟层的实现
>>***全局变量与初始化***
>>```C
>> sys_arch.c
>> static OS_MEM *MboxMem   // 邮箱内存管理结构 
>> static char MboxMemory Area[最大邮箱数目*邮箱结构体字节大小]  //邮箱内存区域 
>> /*任务堆栈的定义*/
>> ......  
>> ```
  
>>***信号量函数的定义***
>>```C
>> err_t sys_sem_new(sys_sem_t *sem, u8_t count)            //创建一个信号量
>> void sys_sem_free(sys_sem_t *sem)                        //删除一个信号量
>> void sys_sem_signal(sys_sem_t *sem)                      //释放一个信号量
>> 
>> /*根据timeout的值等信号量，若为0则阻塞直到信号量到来*/
>> /*若不为0，则最多阻塞timeout的毫秒数*，并返回等待的毫秒数/
>> u32_t sys_arch_sem_wait(sys_sem_t *sem, u32_t timeout)
>>```

>>***邮箱函数的定义***
>>```C
>> err_t sys_mbox_new(sys_mbox_t *mbox, int size)
>> void sys_mbox_free(sys_mbox_t *mbox)
>> void sys_mbox_post(sys_mbox_t *mbox, void *data)
>> err_t sys_mbox_trypost(sys_mbox_t *mbox, void *msg)
>> u32_t sys_arch_mbox_fetch(sys_mbox_t *mbox, void **msg, u32_t timeout)
>> u32_t sys_arch_mbox_tryfetch(sys_mbox_t *mbox, void **msg)
>> int sys_mbox_valid(sys_mbox_t *mbox)          
>> void sys_mbox_set_invalid(sys_mbox_t *mbox)
>> ```
