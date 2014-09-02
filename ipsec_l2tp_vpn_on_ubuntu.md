# 利用 openswan, xl2tpd, ppp 在 Ubuntu 14.04 上搭建 IPSEC/L2TP VPN #

大多数情况我都是在个人电脑上利用`ssh -D`访问一些国内受限网站, 主要还是去 [Google](https://www.google.com)
查资料。想在移动终端上耍一些应用就麻烦了，如 google plus, twitter, facebook, dropbox, youtube 等等。
网上也有很多免费的 VPN，考虑到安全因素和长期稳定，我也爱折腾这些新玩意儿，所以有了这篇文章。本文主要摘自
[Raymii.org](https://raymii.org/s/tutorials/IPSEC_L2TP_vpn_with_Ubuntu_14.04.html)，原文详细介绍了搭建过程，
您也可以参考原文。

## 准备工作 ##

+ 注册并安装 Ubuntu 14.04 的 VPS, 推荐 [Digital Ocean](https://www.digitalocean.com/?refcode=bf1570fa2334)
+ 客户端系统支持 IPSEC/L2TP
+ 确保以下防火墙端口打开, 1701 TCP, 4500 UDP, 500 UDP

**注意: 以下安装过程使用超级用户root登录**

## 安装 openswan, xl2tpd, ppp ##

    # 安装所需包
    apt-get install openswan xl2tpd ppp lsof

在安装 openswan 过程中会询问一些问题, 默认安装即可。

## 配置防火墙并确保系统转发 IP 包 ##

    # 防火墙允许 VPN 数据流
    iptables -t nat -A POSTROUTING -j SNAT --to-source %SERVERIP% -o eth+

%SERVERIP% 为您 VPS 的外部IP地址。如不清楚IP地址，可在命令行输入 `curl http://ip.mtak.nl`，
返回的结果即为您的IP地址。
**注意：全文 _SERVERIP%_ 均由此地址替代**

    # 开启系统 IP 包转发并禁用 ICP 重定向
    echo "net.ipv4.ip_forward = 1" |  tee -a /etc/sysctl.conf
    echo "net.ipv4.conf.all.accept_redirects = 0" |  tee -a /etc/sysctl.conf
    echo "net.ipv4.conf.all.send_redirects = 0" |  tee -a /etc/sysctl.conf
    echo "net.ipv4.conf.default.rp_filter = 0" |  tee -a /etc/sysctl.conf
    echo "net.ipv4.conf.default.accept_source_route = 0" |  tee -a /etc/sysctl.conf
    echo "net.ipv4.conf.default.send_redirects = 0" |  tee -a /etc/sysctl.conf
    echo "net.ipv4.icmp_ignore_bogus_error_responses = 1" |  tee -a /etc/sysctl.conf
    
    # 将以上设置应用到其它网络接口
    for vpn in /proc/sys/net/ipv4/conf/*; do echo 0 > $vpn/accept_redirects; echo 0 > $vpn/send_redirects; done
    
    # 应用配置
    sysctl -p
    
    # 为保证开机自动加载以上配置，添加以下代码至文件 /etc/rc.local
    for vpn in /proc/sys/net/ipv4/conf/*; do echo 0 > $vpn/accept_redirects; echo 0 > $vpn/send_redirects; done
    iptables -t nat -A POSTROUTING -j SNAT --to-source %SERVERIP% -o eth+

确保以上代码在 `exit 0` 前添加。

## 配置 openswan (ipsec) ##

使用您最喜欢的编辑器编辑 */etc/ipsec.conf* 文件，并将以下代码替代源文件

    version 2 # conforms to second version of ipsec.conf specification

    config setup
    dumpdir=/var/run/pluto/
    #in what directory should things started by setup (notably the Pluto daemon) be allowed to dump core?

    nat_traversal=yes
    #whether to accept/offer to support NAT (NAPT, also known as "IP Masqurade") workaround for IPsec

    virtual_private=%v4:10.0.0.0/8,%v4:192.168.0.0/16,%v4:172.16.0.0/12,%v6:fd00::/8,%v6:fe80::/10
    #contains the networks that are allowed as subnet= for the remote client. In other words, the address ranges that may live behind a NAT router through which a client connects.

    protostack=netkey
    #decide which protocol stack is going to be used.

    force_keepalive=yes
    keep_alive=60
    # Send a keep-alive packet every 60 seconds.

    conn L2TP-PSK-noNAT
    authby=secret
    #shared secret. Use rsasig for certificates.

    pfs=no
    #Disable pfs

    auto=add
    #the ipsec tunnel should be started and routes created when the ipsec daemon itself starts.

    keyingtries=3
    #Only negotiate a conn. 3 times.

    ikelifetime=8h
    keylife=1h

    ike=aes256-sha1,aes128-sha1,3des-sha1
    phase2alg=aes256-sha1,aes128-sha1,3des-sha1
    # https://lists.openswan.org/pipermail/users/2014-April/022947.html
    # specifies the phase 1 encryption scheme, the hashing algorithm, and the diffie-hellman group. The modp1024 is for Diffie-Hellman 2. Why 'modp' instead of dh? DH2 is a 1028 bit encryption algorithm that modulo's a prime number, e.g. modp1028. See RFC 5114 for details or the wiki page on diffie hellmann, if interested.

    type=transport
    #because we use l2tp as tunnel protocol

    left=%SERVERIP%
    #fill in server IP above

    leftprotoport=17/1701
    right=%any
    rightprotoport=17/%any

    dpddelay=10
    # Dead Peer Dectection (RFC 3706) keepalives delay
    dpdtimeout=20
    #  length of time (in seconds) we will idle without hearing either an R_U_THERE poll from our peer, or an R_U_THERE_ACK reply.
    dpdaction=clear
    # When a DPD enabled peer is declared dead, what action should be taken. clear means the eroute and SA with both be cleared.

如您升级了您的系统，请重新配置ipsec。

### 设置共享密钥 ###

尽可能长，您可以使用以下密钥，也可以自己生成随机密钥。

    # 修改 /etc/ipsec.secrets 内容为
    %SERVERIP%  %any:   PSK "69EA16F2C529E74A7D1B0FE99E69F6BDCD3E44"
    
    # 生成随机密钥
    openssl rand -hex 30

比如：397ad024ea7c931c88282b1af4451cbbf222642ca13ea81f81e6c7ee5c57

### 验证 IPSEC 配置 ###

    # 确保 IPSEC 是否正常工作
    ipsec verify
    
    # 以下是我的输出
    Checking your system to see if IPsec got installed and started correctly:
    Version check and ipsec on-path                             	[OK]
    Linux Openswan U2.6.38/K3.13.0-32-generic (netkey)
    Checking for IPsec support in kernel                        	[OK]
    SAref kernel support                                       	    [N/A]
    NETKEY:  Testing XFRM related proc values                  	    [OK]
	    [OK]
	    [OK]
    Checking that pluto is running                              	[OK]
    Pluto listening for IKE on udp 500                         	    [OK]
    Pluto listening for NAT-T on udp 4500                      	    [OK]
    Checking for 'ip' command                                   	[OK]
    Checking /bin/sh is not /bin/dash                           	[WARNING]
    Checking for 'iptables' command                             	[OK]
    Opportunistic Encryption Support                            	[DISABLED]

忽略 `/bin/sh` 和 `Opportunistic Encryption` 警告。第一个为openswan的bug，第二个使得xl2tpd可以trip。

## 配置 xl2tpd ##

编辑器打开文件 */etc/xl2tpd/xl2tpd.conf*，用以下代码替换源文件

    [global]
    ipsec saref = yes
    saref refinfo = 30
    
    ;debug avp = yes
    ;debug network = yes
    ;debug state = yes
    ;debug tunnel = yes
    
    [lns default]
    ip range = 172.16.1.30-172.16.1.100
    local ip = 172.16.1.1
    refuse pap = yes
    require authentication = yes
    ;ppp debug = yes
    pppoptfile = /etc/ppp/options.xl2tpd
    length bit = yes

其中，
- ip range = range of IPs to give to the connecting clients
- local ip = IP of VPN server
- refuse pap = refure pap authentication
- ppp debug = yes when testing, no when in production

### 本地用户鉴权 ###

为了通过pam使用本地用户账号而不使用明文密码，修改文件 */etc/xl2tpd/xl2tpd.conf*

    # 增加以下代码
    unix authentication = yes
    
    # 删除以下代码
    refuse pap = yes

在文件 */etc/ppp/options.xl2tpd* 中添加一行

    login

将文件 */etc/pam.d/ppp* 的内容替换为

    auth    required        pam_nologin.so
    auth    required        pam_unix.so
    account required        pam_unix.so
    session required        pam_unix.so

在文件 */etc/ppp/pap-secrets* 中添加此行

    *       l2tpd           ""              *

如果下面chap-secrets添加了用户可以忽略此步骤。

## 配置 ppp ##

编辑文件 */etc/ppp/options.xl2tpd*，并将内容替换为以下代码

    login
    ms-dns 8.8.8.8
    ms-dns 8.8.4.4
    auth
    mtu 1200
    mru 1000
    crtscts
    hide-password
    modem
    name l2tpd
    proxyarp
    lcp-echo-interval 30
    lcp-echo-failure 4

其中，
- ms-dns = The dns to give to the client. I use googles public DNS
- proxyarp = Add an entry to this systems ARP [Address Resolution Protocol] table with the IP address of the peer and the Ethernet address of this system. This will have the effect of making the peer appear to other systems to be on the local ethernet
- name l2tpd = is used in the ppp authentication file

### 添加用户 ###

每一个登录用户都通过 */etc/ppp/chap-secrets* 文件添加，如：

    # Secrets for authentication using CHAP
    # client       server  secret                  IP addresses
    alice          l2tpd   0F92E5FC2414101EA            *
    bob            l2tpd   DF98F09F74C06A2F             *

其中，
- client = username for the user
- server = the name we define in the ppp.options file for xl2tpd
- secret = password for the user
- IP Addresses = leave to * for any address or define addresses from were a user can login

## 测试 ##

到此为止，IPSEC/L2TP VPN 已全部配置完，测试是否成功。
为保证加载最新的配置，需重启 openswan 和 xl2tpd

    /etc/init.d/ipsec restart
    /etc/init.d/xl2tpd restart

### 配置成功 :) ###



