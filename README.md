
**大纲**


**1\.Redis服务器的Socket网络连接建立**


**2\.Redis多路复用监听与文件事件模型**


**3\.基于队列串行化的文件事件处理机制**


**4\.完整的Redis Server网络通信流程**


**5\.Redis串行化单线程模型为什么能高并发**


**6\.Redis内核级请求处理流程与原理**


**7\.Redis通信协议与内核级请求数据结构**


**8\.Redis Server的初始化与持久化**


**9\.Redis分布式集群**


**10\.Redis集群模式的数据结构分析**


**11\.Redis节点之间的三次握手原理分析**


**12\.基于slots槽位机制的数据分片原理分析**


**13\.Redis集群slots分配与内核数据结构**


**14\.基于slots槽位的命令执行流程分析**


**15\.基于跳跃表的slots和key关联关系**


**16\.集群扩容时的slots转移过程与ASK分析**


**17\.Redis主从架构原理**


**18\.Redis老版本的sync主从复制原理以及缺陷**


**19\.Redis新版本psync的偏移量和复制积压缓冲区**


**20\.Redis集群的故障探测**


 


**1\.Redis服务器的Socket网络连接建立**


Redis是一种基于文件事件的网络通信模型，它会将网络事件抽象为文件事件File Event。也就是说，在Redis Server的内部，各种网络通信事件其实都是一些文件事件。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-axegupay5k/96c49c43dfee4b4095451539908bee28~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=6XcSEdYU3LIOrzfgmgXqrdcpckQ%3D)
 



**2\.Redis多路复用监听与文件事件模型**


**(1\)阻塞IO模型(BIO)**


**(2\)非阻塞IO模型**


**(3\)IO复用模型(NIO)**(两次调用两次返回)


**(4\)文件事件模型**


 


**(1\)阻塞IO模型(BIO)**


应用程序调用IO函数时，如果数据没有准备好，那么只能一直等待。如果数据准备好了，则数据会从内核空间拷贝到用户空间，然后IO函数返回成功指示。


 


阻塞IO(BIO)的优点：程序简单，在阻塞等待数据期间，用户线程挂起，用户线程不会占用CPU资源。


 


阻塞IO(BIO)的缺点：为每个连接配套一条独立的线程，或者一条线程维护一个连接成功的IO流的读写。在并发量小的情况下，使用BIO模型没什么问题。但在高并发场景下，则需要大量线程来维护大量网络连接。此时内存、线程切换开销会非常巨大，因此BIO模型在高并发场景下不可用。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/9b197bbad17d4314a3d7aa74327400f7~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=E8gM265QoRBfh6bbtYZyrUKTMGg%3D)
 



**(2\)非阻塞IO模型**


把一个Socket接口设置为非阻塞就是告诉内核：当请求的IO操作无法完成时，不要让进程睡眠，而是返回一个错误。这样IO操作函数将不断测试数据是否已经准备好。如果没有准备好，则继续测试，直到数据准备好为止。在这个不断测试的过程中，会大量占用CPU的时间，所以该模型不被推荐。


 


非阻塞IO的特点：应用程序线程需不断进行IO系统调用，轮询数据是否已准备好。如果没准备好，则继续轮询，直到完成系统调用为止。


 


非阻塞IO的优点：每次发起IO系统调用，在内核等待数据过程中可以立即返回，用户线程不会被阻塞，实时性比较好。


 


非阻塞IO的缺点：需要不断重复发起IO系统调用。这种不断轮询，将不断地询问内核，将占用大量的CPU时间，系统资源利用率较低。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/dc06c2311db843b4aff9256a4117005d~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=gKsetF4mMv0E3xJqbqW6QjsvmQI%3D)
 



**(3\)IO复用模型(NIO)**(两次调用两次返回)


IO多路复用模型，就是内核后来发展了，产生了一种新的系统调用select/epoll。通过select/epoll系统调用，一个线程可以监视多个文件描述符。一旦某个描述符就绪(一般是内核缓冲区可读/可写)，内核能够通知程序进行相应的IO系统调用。非阻塞的BIO就是一个线程只能轮询自己的一个文件描述符。


 


IO多路复用模型的基本原理就是select/epoll系统调用。单个线程不断轮询select/epoll系统调用所负责的成百上千的Socket连接。当某个或者某些socket网络连接有数据到达了，就返回这些可以读写的连接。好处就是通过一次select/epoll调用，就能查询到可以读写的成百上千个网络连接。


 


**select版的多路复用：**


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/c0204ec13a724e15a879b9b21221d3ca~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=bJWJm4SLcxpmcB7upZawC31TQ2Y%3D)
 



**epoll版的多路复用：**


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/78f0323b13ff4300bd2412ba0280300c~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=FjiF0jEPT2JuQ43YMDHTk07oQPw%3D)
 



在这种模式中，首先进行的不是read系统调动，而是select/epoll系统调用。这里有一个前提，需要将目标网络连接，提前注册到select/epoll的可查询Socket列表中，然后才能开启整个IO多路复用模型的读流程。


 


**多路复用的流程如下：**


**步骤一：**进行select/epoll系统调用，查询可读的连接，内核会查询select/epoll系统调用的可查询Socket列表。当任何一个Socket中的数据准备好了，select/epoll系统调用就会返回。当用户线程调用了select/epoll系统调用，则整个线程会被阻塞。


**步骤二：**用户线程获得了目标连接后会发起read系统调用，阻塞用户线程。内核开始复制数据，将数据从内核缓冲区拷贝到用户缓冲区，并返回结果。


**步骤三：**用户线程才解除block状态，用户线程终于真正读取到数据，继续执行。


 


多路复用IO(NIO)的特点：


IO多路复用模型，建立在操作系统内核能够提供系统调用select/epoll的基础上。多路复用IO需要用到两个系统调用：一个是select/epoll的查询调用，一个是read的读取调用。负责select/epoll查询调用的线程，需要不断进行select/epoll轮询，找出可进行IO读取的Socket。另外，多路复用IO模型与前面的非阻塞IO模型是有关系的。对于每一个可以查询的Socket，一般都设置成为非阻塞模型。只是这一点，对于用户程序是透明的(不感知)。


 


多路复用IO(NIO)的优点：


使用select/epoll的优势在于，它可以同时处理成千上万个连接。与一条线程维护一个连接相比，IO多路复用技术的最大优势是：系统不必创建线程，也不必维护这些线程，从而大大减小了系统的开销。Java的NIO技术，使用的就是IO多路复用模型，在Linux系统上使用了epoll系统调用。


 


多路复用IO(NIO)的缺点：


本质上，select/epoll系统调用，属于同步IO，也是阻塞IO。都需要在读写事件就绪后，自己负责进行读写，也就是说这个读写过程是阻塞的。


 


Redis使用多路复用监听Socket：


针对大量的Socket，不太可能每个Socket都用一个线程来监听网络请求和发送响应。面对并发的成千上万个Socket，不可能准备成千上万个线程来处理。所以Redis通过多路复用，实现了一个线程监听多个Socket，这就是IO多路复用模式。


 


**(4\)文件事件模型**


当有大量的客户端来并发访问Redis时，Redis Server端就会有大量的Socket，这些Socket里可能会产生一些网络事件：


一.Accept(连接应答事件)


二.Read(有数据可以读取的事件)


三.Write(有数据可以写出的事件)


四.Close(连接被关闭)


 


这些网络事件都会被抽象成文件事件File Event，即Redis里的网络事件(Accept、Read、Write、Close)都会抽象为文件事件。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/15bb0004012645a1ac60b99ba7e4b023~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=HXAxaRNRrwrfc2JgXN3ezjywa9Q%3D)
 



**3\.基于队列串行化的文件事件处理机制**


当同一时间有大量的Redis Client发送请求，那么短时间里就会有大量的请求到达Redis Server。然后Redis Server内部的大量Socket就会短时间内产生大量事件，比如Accept事件、Read事件等。此时有两种处理方案：


 


**方案一：**把这产生事件的Socket一个个放入Queue里进行串行化排队，不管同一时间有多少请求过来，要返回多少响应，都让所有产生事件的Socket进入队列里排队，串行化等待处理。


 


**方案二：**把产生事件的Socket分发给不同的线程来进行并发处理，通过开启大量的线程，让多个线程并发去处理不同Socket产生的事件。


 


Redis选择的是方案一：


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/1b426118f45b4ab88dc0f04e0ec3bed0~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=AS3rXu4H3SrxUzsrT%2FEAJjFU8GI%3D)
 



**4\.完整的Redis Server网络通信流程**


**步骤一：**在Redis服务器初始化时，Redis会将连接应答处理器和服务器监听套接字的AE\_READABLE事件关联起来。接着如果一个客户端跟Redis发起连接请求，那么服务器监听套接字就会产生AE\_READABLE事件，然后触发连接应答处理器来处理客户端的连接请求，接着创建客户端套接字，并将这个新创建的客户端套接字的AE\_READABLE事件与命令请求处理器关联起来。


 


**步骤二：**当客户端向Redis发起命令请求时，不管是读请求还是写请求都一样。首先会在客户端套接字产生一个AE\_READABLE事件，然后由命令请求处理器来处理。命令请求处理器就会从客户端套接字中读取请求相关数据，传给相关程序去执行。


 


**步骤三：**多个Socket可能会并发产生不同的操作，每个操作对应不同的文件事件。但IO多路复用程序会监听多个Socket，会将Socket放入一个队列中排列。每次从队列中取出一个Socket给文件事件分派器，然后文件事件分派器会把Socket分给对应的事件处理器进行处理。


 


**步骤四：**接着Redis准备好给客户端的响应数据后，Redis会将客户端套接字的AE\_WRITABLE事件与命令响应处理器关联起来。


 


**步骤五：**当客户端准备好读取响应数据时，会在客户端套接字上产生一个AE\_WRITABLE事件。触发命令响应处理器来处理，将准备好的响应数据写入客户端套接字，供客户端来读取。


 


**步骤六：**命令响应处理器写完后，就会删除这个客户端套接字的AE\_WRITABLE事件和命令响应处理器的关联关系。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/df4db35acc8347769be122a3021ff674~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=xsatBpZVB8%2FZ3x2DzZexYM7UGfM%3D)
 



**5\.Redis串行化单线程模型为什么能高并发**


**(1\)大量Redis Client短时间发起连接请求**


**(2\)与大量Redis Client短时间建立连接**


**(3\)建立连接后短时间内发起大量命令请求**


**(4\)为什么通过队列 \+ 单线程进行串行化处理**


**(5\)Redis可以抗下高并发靠的是什么**


**(6\)Redis单线程模型的高并发总结**


 


Redis可以抗高并发，它的并发能力很强。普通机器单机就可以轻松抗每秒几千并发，高配机器可以抗几万并发。接下来结合Redis Server端的网络通信模型来分析一下：为什么Redis能抗高并发。


 


**(1\)大量Redis Client短时间发起连接请求**


大量Redis Client在短时间对单个Redis Server发起请求时会出现什么？


 


首先会出现大量的Redis Client同时要和Redis Server建立网络连接，Redis Server负责连接的Socket在短时间内需要处理大量要建立连接的请求和事件。


 


**(2\)与大量Redis Client短时间建立连接**


为什么短时间内Redis Server可以跟大量的Redis Client建立连接？


 


原因一：连接建立的性能开销是比较低的，短时间内完成大量的连接建立是没问题的。


 


原因二：要看和Redis Client建立的连接是短连接还是长连接。如果是短连接模式，那么频繁的建立和断开连接就耗性能，所以建立的都是长连接。虽然第一次建立连接需要花费一些时间，但是后续这个连接就不用重复建立和断开了，可以复用。所以，短时间内大量的客户端请求和Redis Server建立连接，是没有问题的。


 


**(3\)建立连接后短时间内发起大量命令请求**


大量Redis Client和Redis Server建立连接后，会基于Socket在短时间内高并发地发送命令请求。于是Redis Server的各个Socket短时间内就会出现大量的网络事件，这些网络事件会全部通过队列 \+ 单线程来进行串行化处理。


 


**(4\)为什么通过队列 \+ 单线程进行串行化处理**


针对内存里的共享数据结构，如果允许多线程并发访问，那么就会导致频繁的加锁和互斥。而且多线程对CPU负载消耗是很大的，如果CPU负载太高，多线程运转的效率也会急转直下。所以多线程访问一块共享内存数据结构，大量的加锁和互斥竞争，会导致性能不高。因此单线程可避免多线程切换对CPU负载消耗，避免对内存数据结构进行大量的加锁和互斥。


 


**(5\)Redis可以抗下高并发靠的是什么**


Redis可以抗下高并发靠的是每个请求执行和处理的速度和效率。如果一个请求要耗费10ms才能处理完毕，此时每秒只能处理100个请求。如果一个请求只要1ms就可以处理完毕，那么每秒就能处理1000个请求。而当Redis的一个请求是基于纯内存来操作，而且避免了加锁和互斥，那么处理速度是很快的。


 


**(6\)Redis单线程模型的高并发总结**


首先，依靠IO多路复用可以同时监听大量连接请求，大量请求过来后先进行串行化排队。然后，由于基于纯内存来操作数据 \+ 单线程没有线程切换及数据竞争，所以大量请求都能很快地处理掉。


 


**6\.Redis内核级请求处理流程与原理**


Redis Server会为每个建立好连接的客户端，创建一个RedicsClient内存数据结构。RedicsClient之间会以链表的方式进行组织，RedicsClient中包含关键的两部分是：输入缓冲区 \+ 输出缓冲区。


 


Redis Server为客户端建立好连接后，当客户端发送请求命令给Redis Server时，Redis Server的命令请求处理器就会往RedisClient数据结构的输入缓冲区中写入请求命令。


 


Redis Server中会有一个命令查找表，由一系列的RedisCommand组成。根据RedisClient的输入缓冲区的请求命令 \+ 命令查找表，可以找到对应的RedisCommand。根据具体的RedisCommand，就可以操作Redis Server中的内存数据结构，如字符串、哈希等。根据RedisCommand命令操作完Redis Server中的内存数据结构后，就会将操作结果放入RedisClient的输出缓冲区。


 


后续客户端想要获取请求响应时：Redis Server就可以找到该客户端对应的RedisClient数据结构，然后就可以从RedisClient的输出缓冲区找到命令请求的操作结果，进行返回。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/a453f26e789c4008948a1a6c620736a0~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=aAYPOk7MfEix4qoF71QUe14DxSE%3D)
 



**7\.Redis通信协议与内核级请求数据结构**


Redis Client和Redis Server端之间的数据需要按照约定的格式来进行组织。比如客户端请求：set key value \-\> jedis.set(key, value)，会通过某种协议组织成某种数据。


 


在网络通信的过程中，代码里的对象和数据结构，往往需要进行按照协议进讲行封装和组织。按照协议进行组织之后，会得到一些协议数据，比如：



```
*3\rn$3\rnSETr\n$3r\nkeyr\n$5Vr\nvalue\r\n
```

按照协议组织的请求数据，还必须进行序列化。


 


请求数据从Client端发送到Server端的过程中：首先会按照协议组织成协议数据，然后协议数据会被序列化成字节数据流，即byte\[]字节数组，接着字节数据流会被Socket通过网络传输到Server端。Server端通过Socket读取出来的首先是字节流，然后把字节流进行反序列化拿到一个协议数据，接着协议数据才会按照协议还原回请求数据，最后请求数据会被放入到RedisClient的输入缓冲区里。


 


**请求数据 \-\> 协议数据 \-\> 序列化字节流数据 \-\> Socket网络传输 \-\> 反序列化字节流数据 \-\> 协议数据 \-\> 请求数据**


 


RedicsClient之间会以链表的方式进行组织。


 


RedicsClient中会有一个querybuf字段表示输入缓冲区，还原出来的请求数据会放入querybuf中。


 


RedicsClient中会有一个buf字段表示输出缓冲区，调用命令函数的执行结果就会写入buf中。


 


RedicsClient中会有一个arg字段存放命令字符串。


 


RedicsClient中会有一个cmd字段指向在命令查找表找到的RedisCommand。


 


**8\.Redis Server的初始化与持久化**


**(1\)Redis启动时初始化的主要工作**


**(2\)Redis的定位**


**(3\)Redis的持久化**


 


**(1\)Redis启动时初始化的主要工作**


初始化命令查找表、RedisCommand、IO多路复用程序、套接字Queue、事件处理器等，读取并加载redis.conf里的配置到内存。


 


**(2\)Redis的定位**


Redis是基于内存进行NoSQL数据存储，可以认为它是一个分布式缓存系统。因为Redis是基于内存的，所以才可以用来进行缓存。


 


由于基于内存必然会面临一个问题：如果Redis进程重启，那么内存里的数据就全部丢失了。所以一般也会给Redis开启数据持久化：用AOF来进行数据持久化，用RDB去进行周期性的冷备份。


 


**(3\)Redis的持久化**


AOF就是每次对内存里的数据进行更新后，都会有对应的记录写入到内存中，可以设定AOF记录刷新到磁盘的策略。


 


一.比如每条数据写入内存后就把它的AOF记录刷到磁盘，确保每条数据都不会丢失，但这样会导致Redis的性能退化到基于磁盘的数据存储级别。


 


二.当然也可以设定每秒把内存里的AOF记录刷到磁盘文件里。如果突然对Redis Server进程进行重启，可能会丢失1秒内写入到内存的数据所对应的AOF记录。但只要Redis Server重启后，就会把磁盘文件里之前的AOF记录都读取出来，还原内存数据。


 


RDB就是周期性把内存的数据写一份快照放到磁盘文件里去。RDB比较适合做数据冷备份，它会将数据备份成一个文件，可以将RDB文件放到其他服务器上。如果服务器磁盘坏了导致AOF没了，此时就可以基于1个小时前的RDB去进行数据恢复。


 


**9\.Redis分布式集群**


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/9d4814c246bf4423902b4d2da31af15d~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=xbuIOvr7eJJzWdPoRU1N%2Fy5oyto%3D)
 



**10\.Redis集群模式的数据结构分析**


在Redis集群模式下，每个节点都会给其他的节点创建对应的内存数据结构，来保存该节点信息。


 


每个Redis Server都会有一个ClusterState内存数据结构，用来保存所有节点的信息。


 


ClusterState结构中会有一个myself字段，指向ClusterState它自己代表的ClusterNode节点。


 


ClusterState结构中会有一个state字段，表示整个Redis集群当前所处的状态。


 


ClusterState结构中会有一个nodes字段，表示一个包含了各个节点的ClusterNode结构的字典。


 


ClusterNode结构里会有一个name字段，代表着该Redis节点在集群中自动生成的名字。


 


ClusterNode结构里会有一个flags字段，代表着该Redis节点在集群中的角色是主还是从。


 


ClusterNode结构里会有一个ip字段，代表着该Redis节点的IP地址。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/06267b946a5440b7ab4e8a66f49e6e8a~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=i4n%2FVzyf81BC%2FIsmyWujJU2%2BWW8%3D)
 



**11\.Redis节点之间的三次握手原理分析**


Redis集群里的节点在互相通信前，需要进行meet协议下的三次握手。meet协议下的三次握手，其实就是通过一系列命令让多个Redis节点组成一个集群。


 


具体来说就是会以一台Redis节点为基础，告诉它有另外一个Redis节点。首先Redis Server会给要连接的其他Redis节点，在自己内存里创建一个ClusterNode数据结构。然后发送一条meet消息到要连接的其他Redis节点上，该Redis节点收到一条meet消息后会也在内存里创建一个ClusterNode，并返回一条pong消息。当这个Redis Server收到pong消息后，又会发出一条ping消息给这个要连接的其他Redis节点。


 


meet \-\> pong \-\> ping


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/bfe059d986c045e4a4acebee89231c45~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=oTfAv2PDsL1XmCr16V4QPpjLp5I%3D)
 



meet三次握手协议是Redis自己的协议，与TCP网络连接的三次握手是不同的。TCP三次握手是用来建立基础的底层网络连接，属于网络层的协议。Redis三次握手属于应用层的协议，用于和集群里的各个节点建立相互连接。


 


Gossip协议的核心就是：发送meet、pong、ping消息时，会顺便随机选节点记录的2个节点信息一起发送出去。


 


三次握手 \+ Gossip协议传播 \-\> 集群建立


 


**12\.基于slots槽位机制的数据分片原理分析**


假设需要往Redis集群里写入100G的数据，Redis集群里有5个节点，那么每个节点就需要在内存里存储20G的数据，这时就要面临两个问题。


 


**问题一：**往Redis集群写入数据时数据应该写入到哪个数据分片，也就是往Redis集群写入一条数据时，这条数据应该写入到集群的哪个节点里。


 


**问题二：**数据应该如何在Redis集群各个节点间进行迁移和Rebalance。


 


**一.数据节点的扩容**


Redis集群刚开始是5个节点，现在往集群里加入1个节点。那么由于刚刚加入集群的这个节点是空的，没有数据的。所以Rebalance就是把已有的5个节点上的数据，迁移到新的空节点上去。让已有的节点上的数据变少，让新的空节点上的数据变多，这是数据扩容要实现的效果。


 


**二.数据节点的缩容**


Redis集群目前是5个节点，为节约成本需要缩容为3个节点，那么减少的2个节点的数据也需要迁移到另外的3个节点上。


 


**三.为解决数据扩容和缩容引入数据分片**


Redis集群的每个节点会包含n个数据分片，在往Redis集群写入数据时，会通过路由算法把每条数据写入到某节点的某数据分片里，其中的路由算法可能是随机分配或者是轮询分配等。


 


当Redis集群需要进行扩容时：首先计算出已有的每个节点要迁移哪些数据分片给新节点，然后再对这些指定的数据分片进行迁移。


 


当Redis集群需要进行缩容时：只需要把减少的节点机器上的数据分片迁移给剩余的机器即可。


 


Redis集群的数据分片是通过slots槽位来实现的，Redis集群的数据分片数量是固定的，总共有16384个slots。


 


所以Redis集群启动后，需要给各个节点分配它们负责的slots槽位。每个节点会负责一部分的slots槽位，每个slot槽位就是一个数据分片。


 


**13\.Redis集群slots分配与内核数据结构**


可以通过一些命令手动给节点分配槽位范围。比如指定给节点1分配的槽位范围是1\-5000，指定给节点2分配的槽位范围是5001\-10000等。当一个节点分配好槽位后，就会把自己负责的槽位信息同步给其他节点。


 


每个ClusterNode里会有一个slots字段，用于存放该ClusterNode对应节点被分配的槽位集合。


 


每个ClusterNode里会有一个numslots字段，用于存放该ClusterNode对应节点分配的槽位数量。


 


每个Redis节点里的ClusterState中也会有一个slots字段，是一个大小为16384的数组，数组中的每一个元素会通过指针指向一个ClusterNode，所以ClusterState.slots数组里存放了16384个槽位中每个槽位在哪个节点中。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/75649e6b24c2466c96025734903abc39~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=KG7ULoMjGrWGVNHqWXem%2Br%2BVU7g%3D)
 



**14\.基于slots槽位的命令执行流程分析**


**(1\)多个Redis节点如何一起组成一个Redis集群**


**(2\)组成集群后各节点如何构建内存里的数据结构**


**(3\)Redis集群的去中心化**


**(4\)如何决定一个key应交给集群中哪个节点来处理**


 


**(1\)多个Redis节点如何一起组成一个Redis集群**


各个Redis节点互相之间进行三次握手，基于Gossip协议将连接到的节点扩散给其他节点，这样就可以让一群节点互相之间建立连接了。


 


**(2\)组成集群后各节点如何构建内存里的数据结构**


首先会进行slots分配，即基于命令来分配各个节点的slots。然后每个节点会在内存里记录各个ClusterNode都有哪些slots。每个节点的ClusterState的slots数组16384个元素会指向各个ClusterNode。每个节点会把自己的slots同步给其他节点，从而实现集群里的槽位分配。


 


**(3\)Redis集群的去中心化**


Redis集群架构有一个很关键的特征，就是去中心化。即Redis节点之间的关系是对等的，每个节点处理的事情都一样。这样就避免了需要去选举一个Leader来管控整个集群的情况。


 


**(4\)如何决定一个key应交给集群中哪个节点来处理**


客户端想要对某个key进行请求操作时，由于不知道究竟找哪个节点去处理，所以会随机找一个节点来发送关于这个key的命令请求。


 


该随机节点接收到请求后，会在内部先计算一下这个key究竟属于哪个slot。如果发现该slot就在当前节点中就直接处理，否则就响应MOVED(slot \+ 目标地址)错误让客户端进行请求重定向。


 


计算key属于哪个slot的方法：通过CRC算法(key)得到一个值，然后和16383做位运算得到一个0到16383范围内的数字，这个数字就代表了这个key应该属于哪个slot了。


![](https://p26-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/9e08a6ffc3aa4fcdaad12a799485bc62~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=0QFey9920RIyfZH33RV%2FRRIuFo0%3D)
 



**15\.基于跳跃表的slots和key关联关系**


在集群模式下，所有数据都会被划分到16384个数据槽位里。16384个数据槽位就等于16384个数据分片，每个槽位里都有一部分数据。客户端发送命令后，等待Server端计算key所属的槽位，之后就可以把数据放在对应的槽位里。


 


每次操作完key\-value数据后，会通过skipList跳跃表来对这个key与slot进行关联绑定。这个跳表中的每个节点的分值是槽号，成员是键。在跳表中，节点按各自所保存的分值从小到大排列。通过这个跳表就能快速获取某个slot的所有数据库键。


 


**16\.集群扩容时的slots转移过程与ASK分析**


**(1\)集群扩容时的slots是如何进行转移的**


**(2\)进行slots转移过程中可能会出现的ASK错误**


 


**(1\)集群扩容时的slots是如何进行转移的**


集群里加入一个新节点，可以通过命令把某节点上的一部分slots转移到该新节点中。进行slots转移需要依靠ClusterState里的两个数据结构：migration\_slots\_to和importing\_slots\_from。进行转移时，首先会根据跳表查出要转移的slots包含的所有key，然后再将这些key发送到新节点中进行存储。


 


**(2\)进行slots转移过程中可能会出现的ASK错误**


ASK错误指的是：节点的槽位正在迁移，但却收到了一个key请求，此时该节点没能在自己的数据库里找到该key。于是该节点就会看一下ClusterState.migration\_slots\_to，看看key所属的槽位是否发生迁移。如果是，则响应客户端ASK错误并引导客户端到正在导入槽位的节点去查找key。


 


**17\.Redis主从架构原理**


在从节点的ClusterNode中，其slaveof字段会指向主节点的ClusterNode。


 


在主节点的ClusterNode中，其numslaves字段表示拥有多少个从节点。


 


在主节点的ClusterNode中，其slaves字段中会指向具体的从节点的ClusterNode。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/06f08a6c2ebd467c9d00792ee00b4791~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=n2ArCu6h61gowzxabgKUf7yj0j4%3D)
 



**18\.Redis老版本的sync主从复制原理以及缺陷**


Redis 2\.x以前的老版本里使用的sync主从复制有很多缺陷，只有了解老版本的sync主从复制原理，才能理解Redis主从复制原理的演进过程。


 


某节点通过slaveof操作成为主节点的从节点后，主节点会做如下事情。


 


**步骤一：**首先主节点会执行bgsave命令进行后台保存，生成数据快照文件——RDB文件


 


**步骤二：**然后主节点会将生成RDB文件的时间点之后的所有命令，写入复制缓冲区中


 


**步骤三：**接着主节点会将RDB文件传输给从节点，从节点收到RDB文件后会将数据加载到内存中


 


**步骤四：**之后主节点便会把复制缓冲区里的命令发送给从节点执行，这样主从节点的数据几乎一样了


 


**步骤五：**最后主节点通过命令传播机制，把主节点最新的命令操作也同步给从节点


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/2a23e3104be84457b5489e6cfe7d3697~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=MJJVWEGMhvUbCeKj878mKC%2FCR6w%3D)
 



这种sync模式的问题是：每次从节点重启都要发送sync命令给主节点，让主节点按照上述步骤重新做一遍数据同步操作。由于每次从节点重启都要执行sync，而sync开销最大的地方是执行bgsave操作(特别耗时)。


 


bgsave是一个极为重量级且耗时的操作。如果Redis内存里有大量数据(如十个G)，那么执行bgsave会产生大量磁盘IO，会非常耗时。主节点把这么大的RDB文件传输给从节点，会非常耗费网络资源和带宽，甚至可能被打满。从节点收到RDB文件后，几乎会阻塞对外服务，通过大量磁盘IO加载RDB文件到内存。


 


以上便是sync模式进行主从同步的缺陷。


 


**19\.Redis新版本psync的偏移量和复制积压缓冲区**


**(1\)Redis老版本的主从复制只支持sync模式**


**(2\)Redis新版本使用psync模式优化断线重连**


 


**(1\)Redis老版本的主从复制只支持sync模式**


每次从节点断线重连，主节点都要执行一遍bgsave生成RDB文件并传输数据快照给从节点。在很多情况下，从节点可能仅仅是做一些运维工作而需要进行重启。这时候从节点的断线重连时间间隔其实不长，所以没有必要每次断线重连都传输RDB快照。


 


**(2\)Redis新版本使用psync模式优化断线重连**


如果断开的时间间隔还比较短，那就不需要传输RDB也不需要执行bgsave。其实只需要想办法把断线这段时间里做出的那些命令变更，传输给断线重连的节点，然后断线重连的节点再重新执行这些命令即可。


 


如果断开的时间间隔太长，比如几小时甚至几天。那么在这段时间里做出的数据变更太多，此时就只能通过bgsave生成RDB文件进行传输同步。


 


**(3\)psync模式的核心**


psync主要是基于复制偏移量 \+ 复制积压缓冲区来实现优化的。


 


主从节点都会记录各自的复制偏移量。断线重连后主节点只需对比相互间的复制偏移量差距，然后决定是否执行bgsave生成RDB文件。


 


如果产生偏移量差距的命令都在复制积压缓冲区里，此时就可以把这些命令直接传输给从节点。从节点只需把这些命令执行一遍，就能让数据完成同步。


 


如果节点落后的数据太多，导致重启时主从节点间的数据偏移量差距太大，那么这些偏移量对应的命令在复制积压缓冲区里就找不到了，此时就只能进行全量同步。也就是把生成的RDB快照文件传输给从节点，来实现主从数据同步。


![](https://p3-sign.toutiaoimg.com/tos-cn-i-6w9my0ksvp/7b4441d80b9e4e0cad4bb26e7b871fc6~tplv-tt-origin-web:gif.jpeg?_iz=58558&from=article.pc_detail&lk3s=953192f4&x-expires=1734527113&x-signature=u708ae8mDHGzmNsLAt0VvgYaAAU%3D)
 



**20\.Redis集群的故障探测**


**(1\)Redis定时ping与疑似下线分析**


**(2\)Redis故障报告与下线标记**


 


**(1\)Redis定时ping与疑似下线分析**


每个Redis节点都会定时发送ping消息给所有的其他节点。整个Redis集群里是没有Controller或者Leader角色的，每个节点都是对等的，并没有中心化的总控节点。通过各个节点互相间进行探测和通信来实现集群的功能。


 


每个Redis节点定时发送的ping消息会尝试探测其他节点是否还存活，如果其他节点是存活的就会返回pong消息。


 


如果两个从节点发送出去的ping消息，并没有在指定时间范围内收到pong消息。此时每个从节点就会把主节点的flags状态由master标记为pfail，也就是标记为疑似下线状态。


 


**(2\)Redis故障报告与下线标记**


每个节点都会把疑似下线的信息发送给其他节点，从而实现相互间疑似下线信息的交换。每个节点在汇总针对某个节点的疑似下线报告fail\_reports时，都会判断集群里是否过半节点都认为某个节点下线了，如果是则标记某个节点为正式下线状态fail，最后还要把某个节点被标记为正式下线状态的信息同步给其他节点。


 


flags状态变化：master/slave \-\> pfail \-\> fail。


 


对于Redis集群里的任何一个节点(无论是主节点还是从节点)，都遵循如下的故障探测机制：


一.每个节点都定时发送ping消息给其他节点


二.如果一段时间没收到某节点的pong消息，就标记该节点状态为pfail


三.将pfail信息交换给其他所有节点


四.汇总fail\_report，如果过半节点数量都认为pfail，则标记节点状态为fail，以及同步该节点的fail状态给其他所有节点，这时该节点便正式死亡


 本博客参考[wgetCloud机场](https://longdu.org)。转载请注明出处！
