# Android VpnServices使用总结

Android系统自带的VpnServices是从4.0开始（API LEVEL 15），自己带了一个帮助在设备上建立VPN连接的解决方案，且不需要root权限，本文将对其做一个简单的介绍，并总结其在实际使用的经验。

### 一、基本原理

在介绍如何使用这些的API之前，先来说说其基本的原理。

android设备上，如果已经使用了VpnService框架，建立起了一条从设备到远端的VPN链接，那么数据包在设备上大致经历了如下四个过程的转换：
![vpnservices-trace.png-47.3kB][1]

1. 应用程序使用socket，将相应的数据包发送到真实的网络设备上。一般移动设备只有无线网卡，因此是发送到真实的WiFi设备上；

2. Android系统通过iptables，使用NAT，将所有的数据包转发到TUN虚拟网络设备上去，端口是tun0；

3. VPN程序通过打开/dev/tun设备，并读取该设备上的数据，可以获得所有转发到TUN虚拟网络设备上的IP包。因为设备上的所有IP包都会被NAT转成原地址是tun0端口发送的，所以也就是说你的VPN程序可以获得进出该设备的几乎所有的数据（也有例外，不是全部，比如回环数据就无法获得）;

4. VPN数据可以做一些处理，然后将处理过后的数据包，通过真实的网络设备发送出去。为了防止发送的数据包再被转到TUN虚拟网络设备上，VPN程序所使用的socket必须先被明确绑定到真实的网络设备上去。(也就是防止上图中第4步再转到第2步，接下来会介绍几种绕过方式)

### 二、代码实现

要实现Android设备上的VPN程序，一般需要分别实现一个继承自Activity类的带UI的客户程序和一个继承自VpnService类的服务程序。

#### 申明权限

要想让你的VPN程序正常运行，首先必须要在AndroidManifest.xml中定义服务的节点显式申明使用**“android.permission.BIND_VPN_SERVICE”**权限。

```
<service
    android:name="com.shadowsocks.ShadowsocksVpnService"
    android:permission="android.permission.BIND_VPN_SERVICE"
    android:process=":shadowsocks">
        <intent-filter>
            <action android:name="android.net.VpnService" />
        </intent-filter>
</service>
```

#### 客户程序实现

客户程序一般要首先调用VpnService.prepare函数：

```
Intent intent = VpnService.prepare(this);
if (intent != null) {
  startActivityForResult(intent, 0);
} else {
  onActivityResult(0, RESULT_OK, null);
}
```

VpnService.prepare函数的目的，主要是用来检查当前程序是否已经被用户授权过Vpn 权限，如果当前程序还没有被用户授权过vpn权限，则VpnService.prepare函数会返回一个intent。这个intent就是用来触发确认对话框的，程序会接着调用startActivityForResult将对话框弹出来等用户确认。如果用户确认了，会调用onActivityResult函数，并告之用户的选择。

![vpn-dialog.jpg-209.1kB][2]

如果之前用户已经授权过本程序的vpn权限，就不需要用户再确认了。因为用户在本程序第一次建立VPN连接的时候已经确认过了，就不要再重复确认了，直接手动调用onActivityResult函数就行了。

在onActivityResult函数中，处理相对简单：

```
protected void onActivityResult(int request, int result, Intent data) {
  if (result == RESULT_OK) {
    Intent intent = new Intent(this, MyVpnService.class);
    ...
    startService(intent);
  }
}
```
如果返回结果是OK的，也就是用户同意建立VPN连接，则将你写的，继承自VpnService类的服务启动起来就行了。

目前Android只支持一条VPN连接，如果新的程序想建立一条VPN连接，会先中断系统中当前存在的那个VPN连接。

#### 服务程序实现

服务程序必须要继承自android.net.VpnService类：

```public class MyVpnService extends VpnService``` 
VpnService类封装了建立VPN连接所必须的所有函数，后面会逐步用到

建立链接的第一步是要用合适的参数，创建并初始化好tun0虚拟网络端口，这可以通过在VpnService类中的一个内部类Builder来做到：
```
Builder builder = new Builder();
builder.setMtu(...);
builder.addAddress(...);
builder.addRoute(...);
builder.addDnsServer(...);
builder.addSearchDomain(...);
builder.setSession(...);
builder.setConfigureIntent(...);
...  
ParcelFileDescriptor interface = builder.establish();
```
可以看到，这里使用了标准的Builder设计模式。在正式建立（establish）虚拟网络接口之前，需要设置好几个参数，关于这些参数和方法的具体含义在最后会专门一个个介绍；

最后调用Builder.establish函数，如果一切正常的话，tun0虚拟网络接口就建立完成了。系统同时还会通过iptables命令，修改NAT表，将所有数据转发到tun0接口上。

这之后，就可以通过读写VpnService.Builder返回的ParcelFileDescriptor实例来获得设备上所有向外发送的IP数据包和返回处理过后的IP数据包到TCP/IP协议栈：
```
// 从tun0虚拟网卡读取到ip的数据流，传给加密程序进行加密(或者其他处理)
FileInputStream in = new FileInputStream(interface.getFileDescriptor());
  
// 网络返回ip数据包，经过解密程序解密后，可以通过这个输出流写回给tun0虚拟网卡
FileOutputStream out = new FileOutputStream(interface.getFileDescriptor());
  
// Allocate the buffer for a single packet.
ByteBuffer packet = ByteBuffer.allocate(32767);
...
// Read packets sending to this interface
int length = in.read(packet.array());
...
// Write response packets back
out.write(packet.array(), 0, length);
```
ParcelFileDescriptor类有一个getFileDescriptor函数，其会返回一个文件描述符，这样就可以将对接口的读写操作转换成对文件的读写操作。

每次调用FileInputStream.read函数会读取一个IP数据包，而调用FileOutputStream.write函数会写入一个IP数据包到TCP/IP协议栈。

这其实基本上就是这个所谓的VpnService的全部了，是不是觉得有点奇怪，半点没涉及到建立VPN链接的事情。这个框架其实只是可以让某个应用程序可以方便的截获设备上所有发送出去和接收到的数据包，仅此而已。能获得这些数据包，当然可以非常方便的将它们封装起来，和远端VPN服务器建立VPN链接，但是这一块VpnService框架并没有涉及，留给你的应用程序自己解决。

这里还有一个问题没有解决，就是文章最开始那张数据流程图中，vpn程序的数据为什么可以直接到正式网卡？一般的应用程序，在获得这些IP数据包后，会将它们再通过socket发送出去。但是，这样做会有问题，你的程序建立的socket和别的程序建立的socket其实没有区别，发送出去后，还是会被转发到tun0接口，再回到你的程序，这样就是一个死循环了。为了解决这个问题，VpnService类提供了一个叫protect的函数，在VPN程序自己建立socket之后，必须要对其进行保护：

``` protect(my_socket); ```

其背后的原理是将这个socket和真实的网络接口进行绑定，保证通过这个socket发送出去的数据包一定是通过真实的网络接口发送出去的，不会被转发到虚拟的tun0接口上去。当然vpnservices也提供了其他方式可以实现让自己应用的数据不再经过tun0，那就是addRoute和addDisallowApplication/addAllowApplication这两个方式，在接下来详细介绍VpnService的具体配置方法时再解释一下。

好了，Android系统默认提供的这个VPN框架就只有这么点东西。

### 最后，简单总结一下：

1. VPN连接对于应用程序来说是完全透明的，应用程序完全感知不到VPN的存在，也不需要为支持VPN做任何更改；

2. 并不需要获得Android设备的root权限就可以建立VPN连接。你所需要的只是在你应用程序内的AndroidManifest.xml文件中申明需要一个叫做“android.permission.BIND_VPN_SERVICE”的特殊权限；

3. 在首次正式建立VPN链接之前，Android系统会弹出一个对话框，需要用户明确的同意；

4. 一旦建立起了VPN连接，Android设备上所有发送出去的IP包，都会被转发到虚拟网卡的网络接口上去（系统是通过给不同的套接字打fwmark标签和iproute2策略路由来实现的）

5. VPN程序可以通过读取这个接口上的数据，来获得所有设备上发送出去的IP包；同时，可以通过写入数据到这个接口上，将任何IP数据包插入系统的TCP/IP协议栈，最终送给接收的应用程序；

6. Android系统中同一时间只允许建立一条VPN链接。如果有程序想建立新的VPN链接，在获得用户同意后，前面已有的VPN链接会被中断；

7. 这个框架虽然叫做VpnService，但其实只是让程序可以获得设备上的所有IP数据包。通过前面的简单分析，大家应该已经感觉到了，这个所谓的VPN服务，的确可以方便的用来在Android设备上建立和远端服务器之间的VPN连接，但其实它也可以被用来干很多有趣的事情，比如可以用来做防火墙，可以用来抓设备上的所有IP包，也可以使用sock5协议封装ip数据让发到中转服务器来做数据中转；

# [VpnService.Builder](https://developer.android.com/reference/android/net/VpnService.Builder)的接口使用详解


## addAddress(InetAddress/String address, int prefixLength)

设置虚拟网络端口tun0的IP地址，例如这个设置为：26.26.26.1，到时从
tun0读取出来的ip包，源ip就是26.26.26.1。这个ip设置最好选择这样一个不常见的IP，防止和目前已经有的IP地址冲突

## addAllowedApplication（String packageName）
可以理解设置代理应用白名单，只有白名单里的程序走代理，参数是应用的packageName，默认情况下如果不设置此方法及addDisallowedApplication，所有手机上的应用程序(包括系统应用)的流量都会走代理。
但是如果一旦使用此方法，就只有被设置进去的应用程序的流量会走代理，想要增加多个应用程序走代理，只能多次调用；

**注意** ：allow和disallow这两个方法只能选择其中一个，要么只设置白名单，要么只设置黑名单，如果两个方法都调用，后调用的方法会抛UnsupportedOperationException异常；

文档里还提到packageName必须是已经安装在手机里的包名，如果设置的包名手机没有安装，会抛PackageManager.NameNotFoundException异常，但实际使用下来，如果设置没有安装的程序包名，并不会抛异常；

## addDisallowedApplication(String packageName)

设置代理黑名单包名，只有黑名单里的程序不走代理，其他都走代理；
其他逻辑跟addAllowedApplication一样；

## addDnsServer(String/InetAddress address)

设置Vpn Dns服务器，这里可以设置ipv4和ipv6，只设置ipv4地址的服务器，那么就没有必要再显示的设置allowFamily(ipv4)，vpn就会代理ipv4的ip包；

这个API比较使用频率比较高，但是并不难理解：如果你设置addDnsService("8.8.8.8")，那么接下来那些使用系统的api(如：InetAddress.getByName(String host))去查询dns的请求，默认的dns查询地址就不会走内网网关，而是走我们这里设置的地址8.8.8.8；

**注意**，我们这里设置的Dns地址8.8.8.8，如果这个ip被添加到路由表(接下来介绍的方法addRoute)里面，那么解析流程就变成 App ->dns request -> proxy server -> dns server;
如果这个ip没有添加到路由表里面，那么就是直接发起发往指定的Dns服务器: App -> dns request -> dns server

## addRoute(InetAddress/String address, int prefixLength)
设置路由表，支持ipv4及ipv6; 关于路由表的细节这里不做介绍。你只要理解这里可以设置哪些ip段走代理，例如这里传入198.18.0.0/16网段，addRoute("198.18.0.0", 16), 那意思就是只有目标地址是198.18...开头的ip段会走代理，其他ip端还是直连；那么如果想所有ip都走代理呢，那就设置"0.0.0.0/0"就行(ipv6使用::/0)，意思是所有ip都走代理;

它的作用就是分流，例如一个常见的场景是，如果你想国内的ip直连，国外的ip走代理，那么就可以使用这个方法实现，只要你将所有国内的ip段都添加进去就行；

## addSearchDomain(String domain)
就是添加DNS域名的自动补齐。DNS服务器必须通过全域名进行搜索，但每次查找都输入全域名太麻烦了，可以通过配置域名的自动补齐规则予以简化；
(这个方法没有用过，也没有看到别人使用)

## allowBypass（）
这个方法比较难理解，就是vpn服务提供了一种方式，允许其他app可以选择流量不走代理；
默认情况下，所有app流量都会走代理，但是如果vpnservices主动调用了此方法，那么其他app就可以使用[ConnectivityManager#bindProcessToNetwork](https://developer.android.com/reference/android/net/ConnectivityManager#bindProcessToNetwork(android.net.Network))这个方式绕过vpn，而让网络直连；

关于bindProcessToNetwork这方法多说几句，默认情况下，即使不调用allowBypass，vpn自身应用也可以通过这个方法让自己app的流程绕开代理；而其他应用如果vpn服务没有调用allowBypass允许其他应用不走代理，那么使用于bindProcessToNetwork会绑定失败；

## allowFamily(int family)

这个方法只接收这两个入参：AF_INET (IPv4地址) or AF_INET6 (IPv6地址).
默认情况下，如果不设置，只要设置了addDnsServer/addRoute这两个参数，那么实际上就是允许特定ipv4/ipv6的流量走代理了。
例如 addRoute传入0.0.0.0/0,那么就相当于允许ipv4地址走代理，如果传入::/0，那么就相当于ipv6地址的ip能走代理，如果两种ip类型的都传，那么ipv4/ipv6的地址都能走代理；

## setBlocking(boolean blocking) 

这个方法是设置vpn的filedescriptor的block模式， 一般使用不用设置，默认是非阻塞模式；

## setConfigureIntent(PendingIntent intent)
这个方法是为了在点击默认vpn系统通知栏弹出系统框是，是否展示配置选项，如果设置了intent，那么在系统弹框里就多了一个配置选项，点击配置按钮就会执行你传人的PendingIntent，跳转到相应界面；如下图：
![vpn-configureIntent.jpg-52.9kB][3]

## setHttpProxy(ProxyInfo proxyInfo)
这个方法是安卓api 29以上才有，http代理设置我理解跟在wifi设置里，手动设置http代理的方式一样，一个常见场景是同个网络环境手机抓包http流量，需要设置代理到pc的ip和监听端口；

然而这个http设置之后跟vpn服务设置怎么配合，我并没有实际使用经验，使用过的人可以分享一下经验；

## setMtu (int mtu)
Mtu全程是Maximun Transmission Unit，即表示虚拟网络端口的最大传输单元，如果发送的包长度超过这个数字，则会被分包；这里是设置从虚拟网卡tun0 -> 读取出来的数据包最大字节，超过设置的mtu就会分成多个ip包；

## setSession (String session)
Session，就是你要建立的VPN连接的名字，它将会在系统管理的与VPN连接相关的通知栏和对话框中显示出来

## setMetered(boolean isMetered)
就是将vpn这个Network类型是否为计量网络, 一些应用会使用ConnectivityManager#isActiveNetworkMetered()来获取当前网络是否会计量网络，如果是计量网络就会优化请求数据量，例如一般国内应用会在Wifi网络环境下后台自动下载升级apk，如果是使用isActiveNetworkMetered的话来判断是否可以使用大数据量下载，vpn这个设置就会影响isActiveNetworkMetered的结果。

Android Q及以上机器这个默认设置为true;

## setUnderlyingNetworks(Network[] networks)
这个方法主要是用来允许VPN应用绑定到特定的网络上（例如VPN流量只走WiFi或只走移动数据）。应用场景例如企业VPN只绑定到移动数据网络，这样手机连上企业内网WiFi时就不用再手动断开VPN。
另外：VpnService也有一个这样的方法，区别是在builder里设置，是在一开始就让vpn应用绑定在特定网络，如果vpn连接已经建立，中途想切换特定网络，可以直接调用VpnService.setUnderlyingNetworks来重新设置；

## establish ()
最终上面的配置方法都配置好了之后，调用establish()来真正建立vpn连接，这个方法会返回 ParcelFileDescriptor文件描述符，然后从这个文件描述符里读取和写入数据；具体使用可以参考上面的例子；

## 最后。一些Vpn应用场景的实现方式

### 数据分流

从上面的api里可以看到，为什么有那么多配置，都是为了分流（特定数据直连/特定数据走代理）；

1. 不同网络类型的分流(wifi/移动网络)，使用setUnderlyingNetworks
2. 不同ip类型的分流，使用置addDnsServer/addRoute/allowFamily配置ipv4/ipv6
3. 路由表分流，使用addRoute设置只让某些ip段走代理
4. 不同应用的分流，使用addAllowedApplication/addDisallowedApplication方法，这个api应该是最容易理解和使用最方便的
5. vpn sdk自己实现分流，例如使用shadowsocks,sslocal里就自己实现acl(Access Control List访问控制列表)规则

介绍了上面这么多分流技术之后，最开始提出的问题说是否有另外的方式让vpn自己的流量走入死循环？答案也显而易见：
1. 使用addDisallowedApplication方法，将自己应用排除在外，这种方式也很简单
2. 使用addRoute来做分流，不过这个比较麻烦，需要将代理服务器ip排除在路由表里，一般使用上面那种方式，或者protect;

  [1]: https://github.com/asdzheng/vpnservices/blob/main/VpnServices/vpnservices-trace.png
  [2]: https://github.com/asdzheng/vpnservices/blob/main/VpnServices/vpn-dialog.jpg
  [3]: https://github.com/asdzheng/vpnservices/blob/main/VpnServices/vpn-configureIntent.jpg
