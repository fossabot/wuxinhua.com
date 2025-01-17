---
title: '从面试题“输入URL...发生了什么”学到的(下)'
excerpt: '较详细地描述了输入URL到页面展现的过程，而下篇将扩展补充这一过程涉及的另一些知识点'
date: '2017-10-24 00:15:09'
tags:
---
在[上篇](http://wuxinhua.com/2017/10/13/What-happen-from-input-the-URL-in-the-browser-to-the-page-bring-out/)里，较详细地描述了输入URL到页面展现的过程，而下篇将扩展补充这一过程涉及的另一些知识点。  

- **这个过程涉及到的其他一些知识点;**  
    1. 关于缓存，浏览器的缓存策略；  
    2. TCP协议的三次握手；
    3. SYN Flood攻击;
    4. 关于Nagle算法；  
    5. 浏览器的工作原理；
    6. 初识v8引擎；

#### #关于缓存，浏览器的缓存策略  

##### 浏览器端如何使用缓存？  

我理解的缓存即当用户浏览某一个网站时，浏览器将网页数据暂时存放在客户端,下一次访问时不再使用服务器端资源，直接使用本地缓存数据；通过使用客户端缓存技术，达到减轻服务器负担，提升网站性能的目的,也是网页性能优化的一大利器，使用缓存的优点：  

1. 减少了冗余的网络数据传输;(同样的东西不用再反复请求、回复)  
2. 缓解了网络瓶颈问题；  
3. 降低了对原始服务器的要求;(减缓服务器拥堵情况)  
4. 降低了距离时延;(距离不再是问题)  

那么浏览器端是如何使用缓存的呢？缓存分为强缓存和协商缓存，并且通过不同的缓存命中策略来判断是否使用缓存；

- 浏览器第一次发送请求，服务器端成功返回资源后，如果响应头中没有对缓存进行限制，那么浏览器端会将资源和header头一并缓存；
- 浏览器在加载资源时，先根据这个资源的一些header头部值判断它是否命中强缓存，如果命中，浏览器直接从自己的缓存中读取资源，不会发请求到服务器，如果没有则进一步发送请求到服务器判断是否命中协商缓存，如果命中返回304，而且不会再返回该数据；  
- Cache-Control是强缓存使用的策略之一，在客户端下一次加载资源时，先比较与上一次返回200的时间差，如果没有超过cache-control设置的max-age,则命中强缓存，不会发送请求,直接从缓存中读取文件；如果已过期，浏览器端会将ETag和If-None-Match字段附在header中发送给服务端，Etag是上一次加载资源时，服务器返回附在header头部的一个字符串，是对该资源的一种唯一标识，只要资源有变化，Etag就会重新生成，服务器拿到Etag值后会比较该资源文件，如果未做更新，则命中协商缓存，直接返回304；  
- 当使用Ctrl + F5键强制刷新网页时，浏览器端会直接从服务器加载资源，跳过强缓存和协商缓存；  
- 当使用F5刷新网页时，浏览器会跳过强缓存，但是会检查协商缓存；  

##### 为什么有的缓存是200 OK(from cahe),有的是304 Not Modified,有什么区别？  

二者都是设置了缓存的情况下触发的，其实是强制缓存和协商缓存产生的不同结果：  
如果response中的header设置了Expires或者Cache-Control字段，返回资源没有过期，并且用户没有主动触发的刷新行为，那么客户端不会再向服务端发送请求，直接拿上次返回的结果，返回200 OK(from cahe)。而304意味着浏览器向服务端发送“If-Modified-Since”请求来判断这次请求的内容是否有更新，如果更新了则返回最新的内容，如果没有更新，返回304，它虽然没有200那么快，因为它仍然需要一次完整的HTTP请求，但是服务器并不需要再向客户端发送实体内容。  

##### 关于Cache-Control的max-age=0 和 no-cache有什么区别？  

二者都是禁止客户端使用缓存，区别在于max-age=0表示每次请求都会去访问服务器，并确认文件是否有更新，有则返回新文件，没有则返回304读取缓存文件；但是no-cache总是请求服务器最新文件。  

##### 关于ETag和Last-Modified的作用和区别

当浏览器某个资源的请求没有命中强缓存，就会向服务器发送请求，验证协商缓存是否命中，如果协商缓存命中，返回304状态码，并且显示Not Modified，服务器判断是否命中协商缓存，靠的是Last-Modified，If-Modified-Since和ETag、If-None-Match这两对Header头部中的值来管理的，具体如下：

Last-Modified 与 If-Modified-Since:

1. 浏览器第一次响应请求，返回资源的时会在响应header上加上Last-Modified值，标识该资源在服务器上的最后修改时间；  
2. 浏览器再次请求这个资源时，会在请求头上header加上If-Modified-Since,即前一次返回的Last-Modified时间戳;  
3. 服务器会通过比较请求头的If-Modified-Since和服务器中该资源的最后修改时间，没有变化则返回304 Not Modified；  

ETag 和 If-None-Match:

1. 浏览器第一次向服务器请求一个资源，服务器在返回这个资源的同时，在response的header加上ETag值，这个header是服务器根据当前请求的资源生成的一个唯一标识，这个唯一标识是一个字符串，只要资源有变化字符串会更新，但是跟最后修改时间没有关系；  
2. 客户端再次请求的时候在header头加上If-None-Match，即上一次返回的ETag值，服务器会比较Etag值来判断资源是否有变化，没有更新则返回304 Not Modified；  
可以比较下下面的两个值，分别是Etag和Last-Modified:

```sh
etag:"13545FC6301ECAFB470B0F52DE05926C"
last-modified:Thu, 23 Mar 2017 07:35:45 GMT
```

Etag是文件每次改动的唯一标识，而Last-Modified是文件最后一次改动的时间值，二者的作用一样，但略有不同：  

- 精确度上，Etag大于Last-Modified；资源一旦变化，Etag就会更新，但是Last-Modified的时间单位是秒，如果某个文件在1秒内改变了多次，Last-Modified并没有办法体现出来;  
- 在[RFC 2616](http://www.ietf.org/rfc/rfc2616.txt)版HTTP协议中规定1.1版本的客户端请求中ETag是必不可少的，如果ETag和Last-Modified都存在，两者都应该附在Header中发送到服务器端。  

#### # TCP相关

##### HTTP事务延迟  

碰到网络不好的情况下，会发现请求之后的响应往往不是及时的，HTTP的性能很多情况下取决于底层TCP通道的性能，来看下HTTP请求过程中可能会出现哪些网络延迟？  

1. 客户端需要根据URI来确定资源对应的web服务器ip和端口，需要DNS系统对域名进行解析，这里可能要耗费数十秒的时间；  
2. 发送报文前需建立TCP连接，这里同样需要时间;  
3. 建立连接之后，web服务器从TCP连接中读取数据，并处理请求，传输和处理报文都需要时间；  
4. 最后一步服务端发送响应报文同样耗费时间；  

##### TCP的三次握手协议  

在知乎上有个问题[TCP 为什么是三次握手，为什么不是两次或四次？](https://www.zhihu.com/question/24853633)很有意思，查了一些资料，再来好好理解一下三次握手。  
我们都知道网络OSI模型分成七层，最上层应用层有我们熟悉的HTTP协议、FTP协议等，而TCP位于七层模型的第四层-Transport层，即传输层，这里有两个大名鼎鼎的协议:TCP和UDP协议，
TCP和UDP处在同一层，但是TCP和UDP最不同的地方是TCP提供可靠的数据服务，是一个面向连接的协议，在双方发送数据之前，需要先建立一个连接，类似于两个人用两台电话通话，先确认是否正常拨通，通了才开始聊天这样的一个过程，所以它比UDP协议可靠得多，UDP只是发送数据而已，不提供超时重传、出错重传等功能，也就是不关心发送的数据是否送达，是不可靠协议。而TCP连接建立的过程可以简单得称为“三次握手”，如图所示：

![](https://assets.wuxinhua.com/blog/assets/tcpopen3way.png)  

1. 客户端首先向服务器申请打开某一个端口(用SYN段等于1的TCP报文)，主要是要初始化Sequence Number 的初始值，通信的双方要互相通知对方自己的初始化的Sequence Number（缩写为ISN：Inital Sequence Number）——所以叫SYN，全称Synchronize Sequence Numbers；  
2. 服务器端发回一个SYN-ACK报文通知客户端请求报文已收到；  
3. 客户端收到报文以后再次发出确认报文确认刚才服务器端发出的确认报文；(有点绕，即告诉服务端你刚刚发的确认报文我也收到了)  

至此，三次握手过程完成，连接已经建立，可以传输数据了，所以说要确认双方建立连接，一定得发送送这三次报文，而且三次是最划算的，少了不行，多了浪费资源。TCP和UDP不同的地方就在于TCP得先建立这个连接，假如出现下面这种情况：当一个客户端向服务器发送SYN报文后突然断开连接了，那么将无法接到和响应服务器端发送的SYN+ACK确认报文，这样第二次握手无法完成，这时服务端将等待一段时间后重试（再次发送SYN+ACK到客户端），并且将重试5次，间隔时间分别是1s、2s、4s、8s、16s，如果依然没有响应，则断开这个连接，单个用户倒还好，一旦出现恶意攻击者大规模模拟这个现象，将耗费服务器中大量的资源，这也是下面要讲到的SYN 洪水攻击。

##### SYN Flood攻击  

SYN洪水攻击是Dos(拒绝服务攻击)中的一种，利用TCP协议连接的缺陷，发送大量的TCP连接请求，从而使得被攻击的服务器资源耗尽，即服务器的SYN连接队列耗尽，让正常的tcp连接请求无法处理，目标系统运行缓慢严重者引起网络堵塞甚至系统 瘫痪,服务器随后就不再接受新的网络连接,从而造成正常的客户端无法访问服务器的情况发生。  
比较常见的有增大队列SYN最大半连接数,减小超时值,利用SYN cookie技术,过滤可疑的IP地址等常用应对方法：  

1. 当服务器接收到SYN包,在TCP协议栈中会将相应的半连接记录添加到队列中,之后等待接受下面准备握手的数据包,如果握手成功,那么这个半连接记录将从队列中删除,但如果超时，只能在重发确定连接失效的情况下才会将该记录删除，服务器端的最大半连接容量有限，在受到SYN形式的Dos攻击后，这个队列将很快被塞满，因此防御方法可以将该值设置成更大值；  

2. 除了调整SYN的连接数，还可以调整TCP参数tcp_synack_retries，可以用他来减少重试次数；  

3. 在服务器上可以进行抓包操作,这样可以对数据包中的可疑IP进行检测,然后再对这些IP进行过滤,从而将其无法正常连接服务；

4. Linux下给了一个叫tcp_syncookies的参数来应对这种攻击，当SYN队列满了后，TCP会通过源地址端口、目标地址端口和时间戳打造出一个特别的Sequence Number发回去（又叫cookie），如果是攻击者则不会有响应，如果是正常连接，则会把这个 SYN Cookie发回来，然后服务端可以通过cookie建连接（即使你不在SYN队列中）；具体可以查看陈皓的文章[TCP 的那些事儿（上）](https://coolshell.cn/articles/11564.html)中对SYN 洪水攻击的描述；  

##### 关于Nagle算法  

TCP/IP协议中，客户端无论发送多少数据，总是会在数据前面加上协议头，服务端接收到数据，也需要发送ACK表示确认，为了尽可能的利用网络带宽，TCP总是希望尽可能的发送足够大的数据，Nagle算法主要为了避免客户端发送过多的小的数据包，通过减少需要通过网络发送包的数量来提高TCP/IP传输的效率。  

tcp协议中服务端对接受到每个数据包都会发送一个确认ack，但这样确实有点浪费，所以tcp在发送ack前有一段延迟，如果在这段等待时间内有数据需要发送到客户端，则会捎上确认的ack一起发送，如果ack没有及时返回，之前发送的数据包可以认为是未被确认的小段，而Nagle算法的基本定义是任意时刻，最多只能有一个未被确认的小段；这个小段指的是一个小于MSS尺寸的数据块，Nagle算法只允许一个未被ACK的包存在于网络，它并不管包的大小，因此事实上就是一个扩展的停-等协议，设置Nagle算法后可以理解为客户端在攥数据包，等到一定数量大小后集中发送过去。Nagle算法默认是打开的，例如在一些交互性较强的程序中，可以在Socket设置TCP_NODELAY选项来关闭这个算法。  

##### 浏览器的工作原理  
  
再次将重点回归到浏览器本身，浏览器的主要功能有两个：1、发送请求；2、将服务器端返回的资源呈现出来，这里所说的资源一般是指 HTML 文档，也可以是 PDF、图片或其他的类型。

###### 浏览器主要构成  

浏览的内部组件包括：  

1、用户界面

包括地址栏、后退/前进按钮、标签页、刷新、书签目录、菜单等用户在主窗口界面可以操作部分。

目前使用的主流浏览器有五个：Internet Explorer、Firefox、Safari、Chrome 和 Opera，浏览器的用户界面没有正式的规范，所以不同的浏览器用户界面之间还有一定的差别，但基本都会有一些通用的元素：地址栏、状态栏、工具栏等。  

2、浏览器引擎  

用户用户界面和呈现引擎之间传送指令。  

3、渲染引擎  

负责呈现请求的内容，例如请求的是内容是HTML，渲染引擎负责解析HTML和CSS内容，将网页内容呈现在屏幕上。渲染引擎工作的主要流程如下：  

1）、将HTML文档的各标记逐个转化成“内容树”上的 DOM 节点。同时也会解析外部 CSS 文件以及样式元素中的样式数据。  

2）、组合HTML的DOM节点和CSS样式创建另一个树形结构：呈现树，呈现树包含多个带有视觉属性（如颜色和尺寸）的矩形。这些矩形的排列顺序就是它们将在屏幕上显示的顺序。  

3）、“布局”阶段，为每个节点分配一个应出现在屏幕上的确切坐标。  

4）、最后是绘制 - 呈现引擎会遍历呈现树，由用户界面后端层将每个节点绘制出来。  

4、网络  
用于网络调用，比如发送HTTP请求。  

5、用户界面后端  
用于绘制基本的窗口小部件，比如组合框和窗口。  

6、Javascript引擎  

用于解析和执行 JavaScript 代码。

7、数据存储  

用于持久化浏览器端存储的各种数据，例如Cookie.  
具体结构图如下：
![browser](https://blog-imgs-oss.oss-cn-beijing.aliyuncs.com/browser-structure.png?Expires=1539587722&OSSAccessKeyId=TMP.AQF34XEALXdcnxbgmCq0HGdlYJ1rESOD1tu37CyNC1xPZM_0htjrSCvdA5c9AAAwLAIUBPB4z3xNnrZbiPP3bNGV7J5_N_ICFBfsrRcotLisf5j05L1Q5NeWXsRx&Signature=vDf7qiit5k5dq%2BmABJN2RU3q1PI%3D)  

##### 初识v8引擎  

~~ 篇幅较多 未完待续

附：
[陈皓-TCP 的那些事儿](https://coolshell.cn/articles/11609.html)
[How browsers work](http://taligarsiel.com/Projects/howbrowserswork1.htm)
