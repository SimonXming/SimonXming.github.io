---
layout: post
title: OpenVPN基础
category: 工具
tags: 第三方包
keywords: 工具
description:
---


system: Ubuntu 15.04

虚拟专用网（VPN）常指几种通过其它网络建立连接技术。它之所以被称为“虚拟”，是因为各个节点间的连接不是通过物理线路实现的，而“专用”是指如果没有网络所有者的正确授权是不能被公开访问到。

![image](https://cloud.githubusercontent.com/assets/8087928/16722656/b3d5753e-477a-11e6-815b-353110cf2cf7.png)

OpenVPN软件借助TUN/TAP驱动使用TCP和UDP协议来传输数据。UDP协议和TUN驱动允许NAT后的用户建立到OpenVPN服务器的连接。此外，OpenVPN允许指定自定义端口。它提供了更多的灵活配置，可以帮助你避免防火墙限制。

OpenVPN中，由OpenSSL库和传输层安全协议（TLS）提供了安全和加密。TLS是SSL协议的一个改进版本。

OpenSSL提供了两种加密方法：对称和非对称。下面，我们展示了如何配置OpenVPN的服务器端，以及如何配置使用带有公共密钥基础结构（PKI）的非对称加密和TLS协议。

## 服务端配置
### 安装依赖

```shell
sudo apt-get install openvpn
sudo apt-get unstall easy-rsa    # 通过openssl或者easy-rsa配置一个密钥对

# 开始之前，我们需要拷贝“easy-rsa”到openvpn文件夹。

mkdir /etc/openvpn/easy-rsa
cp -r /usr/share/easy-rsa /etc/openvpn/easy-rsa
mv /etc/openvpn/easy-rsa/easy-rsa /etc/openvpn/easy-rsa/2.0

cd /etc/openvpn/easy-rsa/2.0
```
这里，我们开始密钥生成进程。

### 密钥生成
#### 首先
我们编辑一个“vars”文件。为了简化生成过程，我们需要在里面指定数据。这里是“vars”文件的一个样例：

```shell
export KEY_COUNTRY="CN"
export KEY_PROVINCE="BJ"
export KEY_CITY="Beijing"
export KEY_ORG="Linux.CN"
export KEY_EMAIL="open@vpn.linux.cn"
export KEY_OU=server
```
#### 其次
我们需要拷贝openssl配置。另外一个版本已经有现成的配置文件，如果你没有特定要求，你可以使用它的上一个版本。这里是1.0.0版本。

cp openssl-1.0.0.cnf openssl.cnf
#### 第三
我们需要加载环境变量，这些变量已经在前面一步中编辑好了。

source ./vars
生成密钥的最后一步准备工作是清空旧的证书和密钥，以及生成新密钥的序列号和索引文件。可以通过以下命令完成。

./clean-all
现在，我们完成了准备工作，准备好启动生成进程了。让我们先来生成证书。

./build-ca
在对话中，我们可以看到默认的变量，这些变量是我们先前在“vars”中指定的。我们可以检查一下，如有必要进行编辑，然后按回车几次。对话如下
```shell
Generating a 2048 bit RSA private key
.............................................+++
...................................................................................................+++
writing new private key to 'ca.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [CN]:
State or Province Name (full name) [BJ]:
Locality Name (eg, city) [Beijing]:
Organization Name (eg, company) [Linux.CN]:
Organizational Unit Name (eg, section) [Tech]:
Common Name (eg, your name or your server's hostname) [Linux.CN CA]:
Name [EasyRSA]:
Email Address [open@vpn.linux.cn]:
```
接下来，我们需要生成一个服务器密钥

./build-key-server server
该命令的对话如下：
```shell
Generating a 2048 bit RSA private key
........................................................................+++
............................+++
writing new private key to 'server.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [CN]:
State or Province Name (full name) [BJ]:
Locality Name (eg, city) [Beijing]:
Organization Name (eg, company) [Linux.CN]:
Organizational Unit Name (eg, section) [Tech]:
Common Name (eg, your name or your server's hostname) [Linux.CN server]:
Name [EasyRSA]:
Email Address [open@vpn.linux.cn]:

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
Using configuration from /etc/openvpn/easy-rsa/2.0/openssl-1.0.0.cnf
Check that the request matches the signature
Signature ok
The Subject's Distinguished Name is as follows
countryName :PRINTABLE:'CN'
stateOrProvinceName :PRINTABLE:'BJ'
localityName :PRINTABLE:'Beijing'
organizationName :PRINTABLE:'Linux.CN'
organizationalUnitName:PRINTABLE:'Tech'
commonName :PRINTABLE:'Linux.CN server'
name :PRINTABLE:'EasyRSA'
emailAddress :IA5STRING:'open@vpn.linux.cn'
Certificate is to be certified until May 22 19:00:25 2025 GMT (3650 days)
Sign the certificate? [y/n]:y
1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```
这里，最后两个关于“签署证书”和“提交”的问题，我们必须回答“yes”。

现在，我们已经有了证书和服务器密钥。下一步，就是去生成Diffie-Hellman密钥。执行以下命令，耐心等待。在接下来的几分钟内，我们将看到许多点和加号。

./build-dh
该命令的输出样例如下
```shell
Generating DH parameters, 2048 bit long safe prime, generator 2
This is going to take a long time
................................+................<许多的点>
```
在漫长的等待之后，我们可以继续生成最后的密钥了，该密钥用于TLS验证。命令如下：

openvpn --genkey --secret keys/ta.key
现在，生成完毕，我们可以移动所有生成的文件到最后的位置中。

cp -r /etc/openvpn/easy-rsa/2.0/keys/ /etc/openvpn/

最后，我们来创建OpenVPN配置文件。让我们从样例中拷贝过来吧：
```shell
cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/
cd /etc/openvpn
gunzip -d /etc/openvpn/server.conf.gz

vim /etc/openvpn/server.conf
```
我们需要指定密钥的自定义路径
```shell
ca /etc/openvpn/keys/ca.crt
cert /etc/openvpn/keys/server.crt
key /etc/openvpn/keys/server.key # This file should be kept secret
dh /etc/openvpn/keys/dh2048.pem
```
一切就绪。在重启OpenVPN后，服务器端配置就完成了。

service openvpn restart

## 客户端配置

假定我们有一台装有类Unix操作系统的设备，比如Ubuntu 15.04，并安装有OpenVPN。我们想要连接到前面建立的OpenVPN服务器。首先，我们需要为客户端生成密钥。为了生成该密钥，请转到服务器上的对应目录中：

cd /etc/openvpn/easy-rsa/2.0
加载环境变量

source vars
然后创建客户端密钥

./build-key client
我们将看到一个与先前关于服务器密钥生成部分的章节描述一样的对话，填入客户端的实际信息。

如果需要密码保护密钥，你需要运行另外一个命令，命令如下

./build-key-pass client
在此种情况下，在建立VPN连接时，会提示你输入密码。

现在，我们需要将以下文件从服务器拷贝到客户端/etc/openvpn/keys/文件夹。

服务器文件列表：
```shell
ca.crt,
dh2048.pem,
client.crt,
client.key,
ta.key.
```
在此之后，我们转到客户端，准备配置文件。配置文件位于/etc/openvpn/client.conf，内容如下
```shell
dev tun
proto udp

# 远程 OpenVPN 服务器的 IP 和 端口号
remote 111.222.333.444 1194

resolv-retry infinite

ca /etc/openvpn/keys/ca.crt
cert /etc/openvpn/keys/client.crt
key /etc/openvpn/keys/client.key
tls-client
tls-auth /etc/openvpn/keys/ta.key 1
auth SHA1
cipher BF-CBC
remote-cert-tls server
comp-lzo
persist-key
persist-tun

status openvpn-status.log
log /var/log/openvpn.log
verb 3
mute 20
```
在此之后，我们需要重启OpenVPN以接受新配置。

service openvpn restart
好了，客户端配置完成。
