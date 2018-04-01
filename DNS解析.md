在上个项目提交测试人员测试时，被告知偶尔有无故白屏的现象，在确认本地代码没有问题的情况下，使用他的手机结合Wireshark抓包了数次，终于发现问题所在：偶尔存在DNS解析失败的情况。

###1. Wireshark

至于Wireshark的使用，可以阅读参考[Wireshark抓包iOS入门教程](https://www.jianshu.com/p/c67baf5fce6d)，大致的使用流程是Mac连上iPhone，然后再terminal中输入`rvictl -s [设备udid]`，其意思是建立了一个映射到iPhone的虚拟网卡，这个新增的网卡叫做rvi0，选中双击即可监听设备上的流量。

数据太多？我们可以添加过滤条件，比如对于某一个端口发送UDP请求，我们要查看数据：`udp.port == 端口号`；或者说我们要查看这个ip下所有的流量：`ip.addr==IP号`。

<br>

### 2. DNS解析

####2.1 原理

DNS是**计算机域名系统 (Domain Name System 或Domain Name Service) **的缩写，它是由域名解析器和域名服务器组成的。域名服务器是指保存有该网络中所有主机的域名和对应IP地址，并具有将域名转换为IP地址功能的服务器。其中**域名必须对应一个IP地址，一个域名只能对应一个IP地址（比如访问一个域名不可能向两个ip地址请求），而IP地址不一定有域名且可以对应多个域名**。域名系统采用类似目录树的等级结构。域名服务器为客户机/服务器模式中的服务器方，它主要有两种形式：**主服务器和转发服务器**。将域名映射为IP地址的过程就称为“**域名解析**”。

①DNS是应用层协议，client端（一般指浏览器）构建DNS查询请求，依次被传输层，网络层，数据链路层等封装传送到达DNS服务器端，最终client端接收到DNS响应消息。

②DNS主要基于UDP运输层协议，这里解释下为什么使用UDP（User Datagram Protocol）这样的无连接的，尽最大能力交付的不可靠数据连接，而不是使用TCP(Transmission Control Protocol 传输控制协议)这样的面向连接的可靠数据连接。

一次UDP名字服务器交换可以短到两个包：一个查询包、一个响应包。一次TCP交换则至少包含9个包：三次握手初始化TCP会话、一个查询包、一个响应包以及四次挥手的包交换。

​    考虑到效率原因，TCP连接的开销大得，故采用UDP作为DNS的运输层协议，这也将导致只有13个根域名服务器的结果。我们知道UDP数据包最大为512个字节，这也导致只有13个根域名服务器的结果。

<br>

#### 2.2 解析过程

第一步：浏览器将会检查缓存中有没有这个域名对应的解析过的IP地址，如果有该解析过程将会结束。浏览器缓存域名也是有限制的，包括缓存的时间、大小，可以通过TTL属性来设置。

第二步：如果用户的浏览器中缓存中没有，浏览器将会查找操作系统缓存中是否有这个域名对应 的DNS解析结果。其实操作系统也会有一个域名解析的过程，在Windows中可以通过C:WindowsSystem32driversetchosts文件来设置，你可以将任何域名解析到任何能够访问的IP地址。那么浏览器会首先使用这个IP地址。也正是因为有这种本地DNS解析的规程，所以黑客就有可能通过修改你的域名解析来把特定的域名解析到它指定的IP地址上，导致这些域名被劫持。

第三步：读取网络设置中的“DNS服务器地址”，操作系统就会将这个域名发送给这里设置的LDNS，也就是本地的域名服务器。这个DNS通常都提供给你本地互联网接入的一个DNS服务。如果你是在小区接入的互联网，那这个DNS就是提供给你接入互联网的应用提供商，即电信或者是联通，也就是通常所说的SPA，可以通过ipconfig查询这个地址。

第四步：如果LDNS仍然没有命中，就直接到Root Server域名服务器中请求解析。

第五步：根域名服务器返回给本地服务器一个所查询域的主域名服务器地址（gTLD Server）。gTLD是国际顶级域名服务器，如.com、.cn、.org等，全球只有13台左右。

第六步：本地域名服务器（Local DNS Server）再向上一步返回的gTLD服务器发送请求。

第七步：接受请求的gTLD服务器查找并返回此域名对应的Name Server域名服务器的地址，这个Name Server通常就是你注册的域名服务器的域名服务商，例如你的域名服务商A那里申请一个域名，那个gTLD将会把这个域名解析任务交由这个域名服务商的服务器来解析。

第八步：Name Server域名服务器会查询存储的域名和IP的映射关系表，正常情况下都根据域名得到目标IP表，连同一个TTL值返回给DNS Server域名服务器。

第九步：返回该域名对应的IP以及TTL值，Local DNS Server会缓存这个域名和IP的对应关系，缓存时间由TTL值限制。

第十步：把解析的结果返回给用户，用户根TTL值缓存在本地系统缓存中，域名解析过程结束。

我们可以使用`nslookup`来查看解析步骤（也可以用它来判断DNS解析是否有效，如果DNS解析正常的话，会反馈回正确的IP地址。）：

![4F917DFD-6B45-49E8-BAA4-BA833AB4448E.png](https://upload-images.jianshu.io/upload_images/3261360-d6de9f2d7083065b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Server：DNS服务器的主机名

Address：DNS服务器的IP地址

Name：解析的URL

Address：解析的IP

<br>

### 3. 问题解决

由于我们使用的是联通的服务器线路，在移动等其他设备上可能偶尔会存在解析失败或者运营商DNS劫持的情况。事实上，对应上线项目是需要服务器多运营商多地部署才是最合理的。

对于解析失败的情况，我的处理是在APP内部内置一个默认IP，当然最好是后续每隔一段时间从服务器获取最新的映射，并覆盖本地的IP；在启动时，判断域名是否有效，有效则走正常流程，无效则使用默认IP去进行访问。
判断代码如下：

```objective-c
+ (NSString *)getIPAddress:(NSString*) hostname{
        Boolean result;
        CFHostRef hostRef;
        CFArrayRef addresses;
        NSString *ipAddress = @"";
          hostRef = CFHostCreateWithName(kCFAllocatorDefault, (__bridge CFStringRef)hostname);
          if (hostRef) {
              result = CFHostStartInfoResolution(hostRef, kCFHostAddresses, NULL); // pass an error instead of NULL here to find out why it failed
              if (result == TRUE) {
                  addresses = CFHostGetAddressing(hostRef, &result);
              }
          }
          if (result == TRUE) {
              CFIndex index = 0;
              CFDataRef ref = (CFDataRef) CFArrayGetValueAtIndex(addresses, index);
              struct sockaddr_in* remoteAddr;
              char *ip_address;
              remoteAddr = (struct sockaddr_in*) CFDataGetBytePtr(ref);
              if (remoteAddr != NULL) {
                  ip_address = inet_ntoa(remoteAddr->sin_addr);
              }
              ipAddress = [NSString stringWithCString:ip_address encoding:NSUTF8StringEncoding];
          }
          return ipAddress;
      }
```


