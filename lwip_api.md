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
>>struct netif netif_demo;			         //创建一个虚拟网卡的实例                         
>>struct ip_addr ipaddr, netmask, gw;	   //用于存放地址
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
>> ****low_level_init为网卡初始化函数，主要完成网卡复位以及参数初始化，同时初始化netif部分字段。****
>>.....
>>netif->hostname = (char*)&wlan_name[vif_entry->index];          //初始化接口名
>>netif->hwaddr_len = ETHARP_HWADDR_LEN;                          //MAC硬件地址长度      
>>os_memcpy(netif->hwaddr, macptr, ETHARP_HWADDR_LEN);            //MAC硬件地址
>>netif->mtu = 1500;                                              //最大传输长度
>>......
>>```  
>> `static err_t low_level_output(struct netif *netif, struct pbuf *p)`    
>> low_level_output为网卡数据包发送函数，在发送数据的同时还要处理pbuf数据区和网卡发送位数的对齐问题。<br/>  
>> `err_t ethernetif_init(struct netif *netif)`   
>> ethernetif_init是对接上层网络接口结构的函数，它会初始化netif部分字段并调用low_level_init。<br/><br/>
>> `static struct pbuf *low_level_input(struct netif *netif)`
>> ```C
>> **** low_level_input是网卡数据包接收函数，接收到的数据将被包装为pbuf形式。****
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
>>   
>> `void ethernetif_input(struct netif *netif)`  
>> ```C
>> **** ethernetif_input是数据包递交函数，通常由硬件层调用,负责将数据包递交至api层处理。****  
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
>> ****与raw/callback API类似，主要是有关操作系统，API使能的编译选项。****
>>......
>>#define NO_SYS                        1           //无操作系统
>>#define LWIP_SOCKET                   0           //不编译socket API
>>#define LWIP_NETCONN                  0           //不编译sequential API
>>#define MEM_ALLIGNMENT                4           //系统对齐字节
>>#define MEM_SIZE                      1024*32     //内存堆大小
>>......
>>```
>>`sys_arch.h的创建：主要是关于操作系统模拟相关的一些宏、结构体、数据类型的定义。`

>***操作系统模拟层的实现***
>>*全局变量与初始化*
>>```C
>> sys_arch.c
>> static OS_MEM *MboxMem   // 邮箱内存管理结构 
>> static char MboxMemory Area[最大邮箱数目*邮箱结构体字节大小]  //邮箱内存区域 
>> /*任务堆栈的定义*/
>> ......  
>> ```
  
>>*信号量函数的定义*
>>```C
>> typedef beken_semaphore sys_sem_t                        //定义信号量的数据类型
>> err_t sys_sem_new(sys_sem_t *sem, u8_t count)            //创建一个信号量
>> void sys_sem_free(sys_sem_t *sem)                        //删除一个信号量
>> void sys_sem_signal(sys_sem_t *sem)                      //释放一个信号量
>> 
>> /*根据timeout的值等信号量，若为0则阻塞直到信号量到来*/
>> /*若不为0，则最多阻塞timeout的毫秒数*，并返回等待的毫秒数/
>> u32_t sys_arch_sem_wait(sys_sem_t *sem, u32_t timeout)
>>```

>>*邮箱函数的定义*
>>```C
>> typedef beken_queue_t sys_mbox_t;                                                //定义邮箱的数据类型
>> err_t sys_mbox_new(sys_mbox_t *mbox, int size)                                   //新建一个邮箱
>> void sys_mbox_free(sys_mbox_t *mbox)                                             //删除一个邮箱
>> void sys_mbox_post(sys_mbox_t *mbox, void *data)                                 //向邮箱投递消息，阻塞
>> err_t sys_mbox_trypost(sys_mbox_t *mbox, void *msg)                              //向邮箱投递消息，不阻塞
>> u32_t sys_arch_mbox_fetch(sys_mbox_t *mbox, void **msg, u32_t timeout)           //从邮箱中取消息，阻塞  
>> u32_t sys_arch_mbox_tryfetch(sys_mbox_t *mbox, void **msg)                       //从邮箱中取消息，不阻塞
>> int sys_mbox_valid(sys_mbox_t *mbox)                                             //查看邮箱是否有效
>> void sys_mbox_set_invalid(sys_mbox_t *mbox)                                      //使邮箱无效（置为空指针）
>> ```

>***sequential API底层工作流程***  
>以下将以流程图形式讲述API在网卡处接发数据包的工作流程
> ![流程图](https://user-images.githubusercontent.com/58408817/180150969-e99b45cc-b025-4da5-b0fe-9373281dba8b.png)  
> 以上工作流程同样也基本适用于SDK3.0.X
> 关键函数：
>> tcpip_input函数通过调用tcpip_inpkt将pbuf和input函数指针封装进msg结构体中，并且送入mbox。
>> tcpip_thread在循环中不断抓取mbox中的msg，并且根据其中封装的类型、input函数推入不同的模块中。
>> tcpip_thread将由tcpip_init中的sys_thread_new函数来启动。

>***sequential API中的数据结构和部分函数***  
>> *API内数据包描述结构netbuf*
>> ```C
>> struct netbuf{
>>    struct pbuf *p,*ptr;          //指向保存数据的pbuf，p永远指向pbuf链表中的第一个pbuf，ptr可以指向其他位置
>>    struct ip_addr *addr;         //发出netbuf数据端的IP地址
>>    u16_t port;                   //发出netbuf数据端的端口号
>>  };
>> ******有关netbuf的简单操作******
>>  struct netbuf *buf;             //声明指针
>>  buf=netbuf_new();               //申请新的netbuf
>>  netbuf_alloc(buf,200);          //为buf分配200字节的空间
>>  netbuf_delete(buf);             //删除buf        
>>```
>> *连接描述结构体netconn*
>>```C
>>struct netconn {
>>    enum netconn_type type;                        //  连接的类型，包括 TCP, UDP 等
>>    enum netconn_state state;                      //  连接的状态
>>    union {                                        //  共用体，内核用来描述某个连接
>>       struct ip_pcb    *ip; struct tcp_pcb *tcp; struct udp_pcb *udp; struct raw_pcb *raw;
>>    } pcb;
>>    err_t err;                                     //  该连接最近一次发生错误的编码
>>    sys_sem_t op_completed;                        //  用于两部分 API 间同步的信号量
>>    sys_mbox_t recvmbox;                          //  接收数据的邮箱
>>    sys_mbox_t acceptmbox;                        //  服务器用来接受外部连接的邮箱
>>    int socket;                                   //  该字段只在 socket 实现中使用
>>    u16_t recv_avail;                             //
>>    struct api_msg_msg *write_msg;                //  对数据不能正常处理时，保存信息
>>    int write_offset;                             //  同上，表示已经处理数据的多少
>>    netconn_callback callback;                    //  回调函数，在发生与该 netconn 相关的事件时可以调用
>>};
>>```
>>*进程间通信结构体*
>>```C
>>struct api_msg {
>>   void (* function)(struct api_msg_msg *msg);      //  函数指针
>>   struct api_msg_msg msg;                          //  函数执行时需要的参数
>>}
>>********函数参数详解********
>>struct api_msg_msg {
>>    struct netconn *conn;                           //  与消息相关的某个连接
>>    union {
>>       struct netbuf *b;                            //  函数do_send 的参数
>>       struct {                                     //  函数do_newconn 的参数
>>       u8_t proto;
>>       } n;
>>       struct {                                     //  函数 do_bind 和do_connect 的参数
>>       struct ip_addr *ipaddr;
>>       u16_t port;
>>       } bc;
>>       struct {                                     //  函数 do_getaddr 的参数
>>       struct ip_addr *ipaddr;
>>       u16_t *port; u8_t local;
>>       } ad;
>>       struct {                                     //  函数do_write 的参数
>>       const void *dataptr;
>>       int len;
>>       u8_t apiflags;
>>       } w;
>>       struct {                                     //  函数 do_recv 的参数
>>       u16_t len;
>>       } r;
>>    } msg;
>>};
>>```
>>*网络连接函数*  
>>`struct netconn *netconn_new(enum netconn_type type)`<br/>
>>建立一个新的连接数据结构，根据是要建立TCP还是UDP连接来选择参数值是NETCONN_TCP还<br/>
>>是NETCONN_UCP。调用这个函数并不会建立连接并且没有数据被发送到网络中。

>>`void netconn_delete(struct netconn *conn)`<br/>
>>删除连接数据结构conn，如果连接已经打开，调用这个函数将会关闭这个连接。
>>
>>`int netconn_addr(struct netconn *conn, struct ip_addr **addr, unsigned short *port)`<br/>
>>这个函数用于获取由conn指定的连接的本地IP地址和端口号。
>>
>>`int netconn_bind(struct netconn *conn, struct ip_addr *addr, unsigned short port)`<br/>
>>为参数conn指定的连接绑定本地IP地址和TCP或UDP端口号。如果addr参数为NULL则本地IP<br/>
>>地址由网络系统确定。
>>
>>`int netconn_connect(struct netconn *conn, struct ip_addr *addr, unsigned short port)`<br/>
>>对UDP连接，该函数通过addr和port参数设定发送的UDP消息要到达的远程主机的IP地址和端<br/>
>>口号。对TCP，netconn_connect()函数打开与指定远程主机的连接。
>>
>>`int netconn_listen(struct netconn *conn)`<br/>
>>使参数conn指定的连接进入TCP监听（TCP LISTEN）状态。
>>
>>`struct netconn *netconn_accept(struct netconn *conn)`<br/>
>>阻塞进程直至从远程主机发出的连接请求到达参数conn指定的连接。这个连接必须处于监听<br/>
>>（LISTEN）状态，因此在调用netconn_accept()函数之前必须调用netconn_listen()函数。
>>
>>`struct netbuf *netconn_recv(struct netconn *conn)`<br/>
>>阻塞进程，等待数据到达参数conn指定的连接。如果连接已经被远程主机关闭，则返回NULL，<br/>
>>其它情况，函数返回一个包含着接收到的数据的netbuf。  <br/>
>>`int netconn_write(struct netconn *conn,void *data,int len,unsigned flags)`<br/>
>>用于tcp连接，把data指向的数据放入conn的输出队列。  
>>
>>`int netconn_send(struct netconn *conn, struct netbuf *buf)`<br/>
>>使用参数conn指定的UDP连接发送参数buf中的数据。函数对要发送的数据大小没有进行校验，无<br/>
>>论是非常小还是非常大，因而函数的执行结果是不确定的。   
>>
>>`int netconn_close(struct netconn *conn)`<br/>
>>关闭参数conn指定的连接。  
>***demo***  
>```C
>static void process_connection(struct netconn *conn){
>     struct netbuf *inbuf;
>     char *rq;
>     int len;
>     inbuf=netconn_recv(conn);              //用inbuf接受数据包
>     netbuf_data(inbuf,&rq,&len):           //用数组rq接受其中的数据
>     /*s*/
>     ......
>     netconn_write(...)                     //发送数据
>     ......
>     netconn_close(conn)                    //关闭连接
> }
>********主函数********
>int main(){
>     struct netconn *conn,*newconn;         //声明连接
>     conn=netconn_new(NETCONN_TCP);         //建立连接句柄
>     netconn_bind(conn,NULL,80);            //绑定端口
>     netconn_listen(conn);                  //进入监听状态
>     while(1){
>        newconn=netconn_accept(conn);       //接受新的连接请求
>        process_connection(newconn);        //处理连接
>        netconn_delete(newconn);            //删除连接句柄
>      }
> }
>```
### Socket API
Socket API是基于Sequential API实现的，实现方法大多为对下层API的简单封装，它进一步简化了代码。
>Socket API中的部分函数
>```C
>//建立一个TCP或者UDP连接
>int socket(int domain, int type, int protocol)
>//绑定端口号
>int bind(int s, struct sockaddr *name, int namelen)
>//监听连接
>int listen(int s, int backlog)
>//接受请求并获取远程主机的IP地址和端口号
>int accept(int s, struct sockaddr *addr, int *addrlen)
>//对于TCP和UDP内部分别调用不同的API函数发送数据
>int send(int s, void *data, int size, unsigned int flags)
>//用UDP发送数据，可以指定数据接受器
>int sendto(int s, void *data, int size, unsigned int flags, struct sockaddr *to, int tolen)
>//将接收到的netbuf复制到mem指针指向存储区
>int recv(int s, void *mem, int len, unsigned int flags)
>//与recv相同，并返回数据长度
>int read(int s, void *mem, int len)
>//与recv相同，并获取远程主机的IP地址和端口号
>int recvfrom(int s, void *mem, int len, unsigned int flags, struct sockaddr *from, int *fromlen)
>```

## 三种API的对比
|API|代码复杂度|优点|缺点|
|----|----|----|----|
|raw/callback|高|1.执行效率高<br/>2.节省内存开销|1.代码逻辑复杂，可读性差<br/>2.应用与内核处于统一进程，影响数据包的接收。|
|sequential|中|拆分连接和应用数据处理，应用IPC通信，提高数据包处理效率。|1.代码逻辑仍然较为复杂<br/>2.IPC数据交流会耗费更多的时间和内存。|
|socket|低|编程接口通用度高，简单易用。|它是基于sequential API的，所以效率相比而言还会更低一些。|

