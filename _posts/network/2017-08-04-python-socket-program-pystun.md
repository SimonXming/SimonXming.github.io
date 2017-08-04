---
layout: post
title: python socket 编程 - pystun
category: 网络
tags: socket stun
keywords: socket
description:
---


# 基础

> The `AF_*` and `SOCK_*` constants are now AddressFamily and SocketKind IntEnum collections.
>
> https://docs.python.org/3/library/socket.html

## What is Address Family?

```
Name                   Purpose                 
    AF_UNIX, AF_LOCAL      Local communication              
    AF_INET                IPv4 Internet protocols        
    AF_INET6               IPv6 Internet protocols
    AF_IPX                 IPX - Novell protocols
    AF_NETLINK             Kernel user interface device    
    AF_X25                 ITU-T X.25 / ISO-8208 protocol 
    AF_AX25                Amateur radio AX.25 protocol
    AF_ATMPVC              Access to raw ATM PVCs
    AF_APPLETALK           Appletalk                      
    AF_PACKET              Low level packet interface   
```

## What is SocketKind?

```
Name                   Purpose                 
    socket.SOCK_STREAM     流式 socket , for TCP
    socket.SOCK_DGRAM      数据报式 socket , for UDP
    socket.SOCK_RAW        原始套接字，普通的套接字无法处理 ICMP、IGMP 等网络报文，而 SOCK_RAW 可以；其次，SOCK_RAW 也可以处理特殊的 IPv4 报文；此外，利用原始套接字，可以通过 IP_HDRINCL 套接字选项由用户构造 IP 头。
    socket.SOCK_SEQPACKET  可靠的连续数据包服务
```

# 简介

STUN，首先在 RFC3489 中定义，作为一个完整的 NAT 穿透解决方案，英文全称是 Simple Traversal of UDP Through NATs，即简单的用 UDP 穿透 NAT。
在新的 RFC5389 修订中把 STUN 协议定位于为穿透 NAT 提供工具，而不是一个完整的解决方案，英文全称是 Session Traversal Utilities for NAT，即 NAT 会话穿透效用。RFC5389 与 RFC3489 除了名称变化外，最大的区别是支持 TCP 穿透。

PyStun follows RFC3489: http://www.ietf.org/rfc/rfc3489.txt

## PyStun stun/__init__.py 代码注释

```python
def get_ip_info(source_ip="0.0.0.0", source_port=54320, stun_host=None,
                stun_port=3478):
    # 创建一个 IPv4 协议的 UDP socket
    s = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    # Set a timeout on blocking socket operations.
    s.settimeout(2)
    # 设置额外的 socket 行为：关闭 socket 之后想继续重用该 socket
    # setsockopt 设置 socket 详细用法(http://blog.chinaunix.net/uid-25324849-id-207869.html)
    s.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    # Bind the socket to address.
    s.bind((source_ip, source_port))
    # 通过 stun 服务器获取 nat_type 和 nat 设备信息
    nat_type, nat = get_nat_type(s, source_ip, source_port,
                                 stun_host=stun_host, stun_port=stun_port)
    external_ip = nat['ExternalIP']
    external_port = nat['ExternalPort']
    s.close()
    return (nat_type, external_ip, external_port)
```

```python
def stun_test(sock, host, port, source_ip, source_port, send_data=""):
    # 设置期望的返回数据结构
    retVal = {'Resp': False, 'ExternalIP': None, 'ExternalPort': None,
              'SourceIP': None, 'SourcePort': None, 'ChangedIP': None,
              'ChangedPort': None}
    # 构造要发送的数据
    # "%#04x" % 17 = 0x11 && "%04x" % 17 = 0011
    # "%#04d" 看上去和 "%04d" 没什么不同
    # 消息长度
    str_len = "%#04d" % (len(send_data) / 2)
    # 生成事务 ID
    tranid = gen_tran_id()
    # 生成要发出的数据 str_data
    # BindRequestMsg 0001 => 绑定消息类型
    # str_len 0000 => 消息长度
    # tranid 32位 => 事务 ID
    # send_data '' => 要发出的数据(空字符串)
    str_data = ''.join([BindRequestMsg, str_len, tranid, send_data])
    # binascii => Conversion between binary data and ASCII
    data = binascii.a2b_hex(str_data)
    # 标志返回消息是否正确
    recvCorr = False
    while not recvCorr:
        recieved = False
        # UDP 数据包重发 3 次
        count = 3
        while not recieved:
            log.debug("sendto: %s", (host, port))
            try:
                sock.sendto(data, (host, port))
            except socket.gaierror:
                retVal['Resp'] = False
                return retVal
            try:
                # 返回数据读取至 buf
                buf, addr = sock.recvfrom(2048)
                log.debug("recvfrom: %s", addr)
                recieved = True
            except Exception:
                recieved = False
                if count > 0:
                    count -= 1
                else:
                    retVal['Resp'] = False
                    return retVal
        # 解析返回消息的类型原始数据
        msgtype = binascii.b2a_hex(buf[0:2])
        # 判断消息类型等于 BindResponseMsg
        bind_resp_msg = dictValToMsgType[msgtype] == "BindResponseMsg"
        # 判断事务 ID 匹配
        tranid_match = tranid.upper() == binascii.b2a_hex(buf[4:20]).upper()
        if bind_resp_msg and tranid_match:
            recvCorr = True
            retVal['Resp'] = True
            # 解析消息长度
            len_message = int(binascii.b2a_hex(buf[2:4]), 16)
            len_remain = len_message
            base = 20
            # 解析所有返回值到 retVal
    ..............
```

```python

def get_nat_type(s, source_ip, source_port, stun_host=None, stun_port=3478):
    """
    RFC3489 文档:
    客户首先从测试 I 开始。如果该测试没有响应，客户就知道他不能使用UDP连接。如果测试产生响应，则客户检查MAPPED-ADDRESS属性。
    如果其中的地址和端口与用来发送请求的套接口的本地IP地址和端口相同，则客户知道他没有通过NAT。然后执行测试II。

    如果收到响应，则客户知道他与互联网之间可互访（或者，至少，他在表现如同完全锥型NAT且没有进行转换的防火墙后）。
    如果没有响应，则客户知道他在对称型UDP防火墙之后。

    在套接口的 IP 地址和端口与测试 I 响应的 MAPPED-ADDRESS 属性中的不同的情况下，客户就知道他在 NAT 后。他执行测试 II。
    如果收到响应，则客户知道他在完全锥型NAT后。如果没有响应，他再次执行测试I，但这次，使测试I响应的 CHANGED-ADDRESS 属性中的地址和端口。
    如果MAPPED-ADDRESS中返回的IP地址和端口与第一次测试I中的不同，客户就知道他在对称型NAT后。如果地址和端口相同，则客户要么在限制 NAT 后，要么在端口限制NAT后。
    要判断出到底是哪种，客户进行测试 III。如果收到响应，则他在限制NAT后，如果没有响应，则他在端口限制 NAT 后。

    该过程产生关于客户应用程序的操作条件的实际信息。在客户与互联网间存在多级NAT的情况下，发现的类型将是客户与互联网间限制最多的NAT的类型。
    NAT的类型接照限制排序从高到低是对称型，端口限制锥型，限制锥型和完全锥型。
    """
    # 初始化 RFC3489/STUN 定义的数据结构，详情 https://tools.ietf.org/html/rfc3489
    _initialize()
    port = stun_port
    # 开始 测试-1
    log.debug("Do Test1")
    # 标志 stun_test() 返回是否成功
    resp = False
    if stun_host:
        ret = stun_test(s, stun_host, port, source_ip, source_port)
        resp = ret['Resp']
    else:
        for stun_host in stun_servers_list:
            log.debug('Trying STUN host: %s', stun_host)
            ret = stun_test(s, stun_host, port, source_ip, source_port)
            resp = ret['Resp']
            if resp:
                break
    if not resp:
        return Blocked, ret
    log.debug("Result: %s", ret)
    # 获取第一次测试的
    # ExternalIP
    # ExternalPort
    # ChangedIP
    # ChangedPort
    exIP = ret['ExternalIP']
    exPort = ret['ExternalPort']
    changedIP = ret['ChangedIP']
    changedPort = ret['ChangedPort']
    if ret['ExternalIP'] == source_ip:
        # 构建第二次测试的消息体
    .......
```

![nat_类型发现过程](/assets/img/post/20170804/nat_类型发现过程.png)