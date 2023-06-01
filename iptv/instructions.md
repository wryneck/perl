北京联通宽带，起因是智能电视各种功能，外面再套一层智能机顶盒的各种功能，家里老人完全懵了，实在学不会。另外家里多个房间如果都想看电视的话，每多一个房价就得多花一份初装费和月租费，不太值得花这个钱，因此在论坛了解了各种iptv的多播转单播，经过实践后，总结出了几种实现方式，对应不同的网络结构。

# 方案一：直接使用运营商光猫上网的，这种最简单，可以直接通过一个单口linux小主机来实现，把小主机通过有线或无线与光猫连接即可，如下图：
![](https://wryneck.github.io/perl/iptv/1.png)
1. 在光猫中的DHCP功能中，把linux小主机加入到静态表中，比如分配该机器的ip为192.168.1.10
2. 小主机安装docker及docker-compose，建议使用debian，直接apt install docker-compose即可完成安装，其他请自行查阅。
3. 新建/root/docker-run/msd_lite目录，并在目录下新建docker-compose.yml文件和msd_lite.conf文件，内容如下：
- (1) docker-compose.yml
```yml
version: '3'
services:
  msd_lite:
    image: tinyserve/msd_lite
    container_name: msd_lite01
    restart: always
    network_mode: host
    volumes:
      - ./msd_lite.conf:/etc/msd_lite.conf
```
- (2) msd_lite.conf
```xml
<msd>
        <log>
                <file>/root/msd_lite/msd_lite.log</file>
        </log>

        <threadPool>
                <threadsCountMax>1</threadsCountMax> <!-- 0 = auto -->
                <fBindToCPU>yes</fBindToCPU> <!-- Bind threads to CPUs. -->
                <fCacheGetTimeSyscall>yes</fCacheGetTimeSyscall> <!-- Cache gettime() syscalls.. -->
                <timerGranularity>100</timerGranularity> <!-- 1/1000 sec -->
        </threadPool>


<!-- HTTP server -->
        <HTTP>
                <bindList>
                        <bind><address>0.0.0.0:7842</address><fAcceptFilter>y</fAcceptFilter></bind>
                        <bind><address>[::]:7842</address></bind>
                </bindList>

                <hostnameList> <!-- Host names for all bindings. -->
                        <hostname>*</hostname>
                </hostnameList>
        </HTTP>


        <hubProfileList> <!-- Stream hub profiles templates. -->
                <hubProfile>
                        <fDropSlowClients>no</fDropSlowClients> <!-- Disconnect slow clients. -->
                        <fSocketHalfClosed>no</fSocketHalfClosed> <!-- Enable shutdown(SHUT_RD) for clients. -->
                        <fSocketTCPNoDelay>yes</fSocketTCPNoDelay> <!-- Enable TCP_NODELAY for clients. -->
                        <fSocketTCPNoPush>yes</fSocketTCPNoPush> <!-- Enable TCP_NOPUSH / TCP_CORK for clients. -->
                        <precache>4096</precache> <!-- Pre cache size. Can be overwritten by arg from user request. -->
                        <ringBufSize>1024</ringBufSize> <!-- Stream receive ring buffer size. Must be multiple of sndBlockSize. -->
                        <skt>
                                <sndBuf>512</sndBuf> <!-- Max send block size, apply to clients sockets only, must be > sndBlockSize. -->
                                <sndLoWatermark>64</sndLoWatermark>  <!-- Send block size. Must be multiple of 4. -->
                                <congestionControl>htcp</congestionControl> <!-- TCP_CONGESTION: this value replace/overwrite(!) all others cc settings: cc from http req args, http server settings, OS default -->
                        </skt>
                        <headersList> <!-- Custom HTTP headers (sended before stream). -->
                                <header>Pragma: no-cache</header>
                                <header>Content-Type: video/mpeg</header>
                                <header>ContentFeatures.DLNA.ORG: DLNA.ORG_OP=01;DLNA.ORG_CI=0;DLNA.ORG_FLAGS=01700000000000000000000000000000</header>
                                <header>TransferMode.DLNA.ORG: Streaming</header>
                        </headersList>
                </hubProfile>
        </hubProfileList>


        <sourceProfileList> <!-- Stream source profiles templates. -->
                <sourceProfile>
                        <skt>
                                <rcvBuf>512</rcvBuf> <!-- Multicast recv socket buf size. -->
                                <rcvLoWatermark>48</rcvLoWatermark> <!-- Actual cli_snd_block_min if polling is off. -->
                                <rcvTimeout>2</rcvTimeout> <!-- STATUS, Multicast recv timeout. -->
                        </skt>
                        <multicast> <!-- For: multicast-udp and multicast-udp-rtp. -->
                                <ifName>enp1s0</ifName> <!-- For multicast receive. -->
                        </multicast>
                </sourceProfile>
        </sourceProfileList>
</msd>

```
4. 修改msd_lite.conf中倒数第5行，将enp1s0改为自己的网卡名称（debian使用ip addr命令查看）
5. 执行命令 docker-compose up -d
6. 在智能电视上安装kodi，并安装PVR插件，并且配置播放源，如果都是按照上方操作，可以使用我的配置文件https://wryneck.github.io/perl/iptv/bj-unicom-iptv-int1-10.m3u

# 方案二：光猫拨号，使用支持udpxy功能的路由器上网，如图：
![](https://wryneck.github.io/perl/iptv/2.png)
1. 进入路由器管理后台，打开udpxy功能，并设置端口为7842
2. 智能电视安装kodi，并配置PVR的播放源。如果路由器ip为192.168.2.1，则可以使用我的配置文件https://wryneck.github.io/perl/iptv/bj-unicom-iptv-int2-1.m3u

# 方案三：光猫拨号，使用普通路由器（单主路由、组mesh、ac+ap均可），只有单口linux小主机，如下图：
![](https://wryneck.github.io/perl/iptv/3.png)
1. 设置与方案一完全相同，在家庭子网络（192.168.2.x）中的设备，均可访问到上级网络（192.168.1.x）中的服务。
2. 缺点就是这个小主机无法访问到局域网其他机器，可能某些服务没办法跑在这上面

# 方案四：光猫拨号，使用普通路由器（单主路由、组mesh、ac+ap均可），linux小主机至少有2个网口，linux主机在内网中还跑着其他服务，如下图：
![](https://wryneck.github.io/perl/iptv/4.png)
1. linux主机1个网口接光猫lan口，另一个网口接路由器lan口
2. 分别在光猫和路由器中设置静态DHCP地址表，分别在2个子网中绑定固定局域网地址，此处分别为192.168.1.3和192.168.2.10
3. 参考方案一的步骤，在linux主机中部署msd_lite服务，注意msd_lite.conf中配置的网卡名称为连接光猫的那个网卡
4. kodi播放源配置为https://wryneck.github.io/perl/iptv/bj-unicom-iptv-int2-10.m3u

其他说明：
1. 我自己对网络这方面的知识属于一知半解，只是根据论坛中各位大佬的帖子实际折腾出了这么几种用法，回馈给论坛
2. 关于光猫改桥接这个事，都说桥接之后网速更快，但是我看过几个博主实测，并没有看出来有什么差别，所以我个人觉得光猫拨号也没什么问题，反而实际操作中感觉更简单
3. 我自己的设备是一个四网口的j1900小主机，带1个设备看直播的时候，cpu占用4%左右，供大家参考
4. 为什么一直强调小主机，主要还是功耗小，同时可以在主机上跑其他服务24小时运行
5. 节目源抓包我没去研究，而是直接使用了其他人的数据，来源为https://github.com/wuwentao/bj-unicom-iptv
6. 其他运营商各位可以自行尝试，可以先用自己的电脑装个虚拟机试试能不能跑通，别先买了设备又不好用
7. 关于设备，如果家里使用的是华硕的路由器，应该就自带了udpxy功能，刷梅林固件之后也有udpxy，所以不用这么折腾。其他wifi6路由器，又要考虑组mesh的话，刷机不一定能找到合适的固件，索性不如再增加一个小主机专机专用，不影响原有的网络结构，耦合性低。
8. 关于udpxy和msd_lite，这两个软件都能实现功能，但msd_lite更省资源，而且配置文件中可以调整的项目比较多，更推荐。
