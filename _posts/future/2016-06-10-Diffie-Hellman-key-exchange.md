---
layout: post
title: 迪菲 - 赫尔曼密钥交换
category: 未来
tags: 安全协议 加密 算法
keywords: 加密 安全
description:
---

### 简介

迪菲-赫尔曼密钥交换（英语：Diffie–Hellman key exchange，缩写为D-H） 是一种安全协议。它可以让双方在完全没有对方任何预先信息的条件下通过不安全信道创建起一个密钥。这个密钥可以在后续的通讯中作为对称密钥来加密通讯内容。

公钥交换的概念最早由瑞夫·墨克（Ralph C. Merkle）提出，而这个密钥交换方法，由惠特菲尔德·迪菲（Bailey Whitfield Diffie）和马丁·赫尔曼（Martin Edward Hellman）在1976年首次发表。马丁·赫尔曼曾主张这个密钥交换方法，应被称为迪菲-赫尔曼-墨克密钥交换（英语：Diffie–Hellman–Merkle key exchange）。

迪菲－赫尔曼密钥交换的同义词包括:
- 迪菲－赫尔曼密钥协商
- 迪菲－赫尔曼密钥创建
- 指数密钥交换
- 迪菲－赫尔曼协议

虽然迪菲－赫尔曼密钥交换本身是一个匿名（无认证）的密钥交换协议，它却是很多认证协议的基础，并且被用来提供传输层安全协议的短暂模式中的完备的前向安全性(秘密的整数a和b在会话结束后会被丢弃。因此，迪菲－赫尔曼密钥交换本身能够天然地达到完备的前向安全性，因为私钥不会存在一个过长的时间而增加泄密的危险。)。
### 历史

迪菲－赫尔曼密钥交换是在美国密码学家惠特菲尔德·迪菲和马丁·赫尔曼的合作下发明的，发表于1976年。它是第一个实用的在非保护信道中创建共享密钥方法。它受到了瑞夫·墨克的关于公钥分配工作的影响。约翰·吉尔（John Gill）提出了离散对数问题的应用。该方案首先被英国GCHQ的马尔科姆·J·威廉森（Malcolm J. Williamson）在稍早的几年前发明，但是GCHQ直到1997年才决定将其公开，这时在学术界已经没有了研究这个算法的热潮了。

这个方法被发明后不久出现了RSA，另一个进行公钥交换的算法。它使用了非对称加密算法。

2002年，马丁·赫尔曼写到：

这个系统...从此被称为“迪菲－赫尔曼密钥交换”。 虽然这个系统首先是在我和迪菲的一篇论文中描述的，但是这却是一个公钥交换系统，是墨克提出的概念，因此如果加上他的名字，这个系统实际上应该称为“Diffie–Hellman–Merkle密钥交换”。我希望这个小小的讲坛可以帮助我们认识到墨对公钥密码学的同等重要的贡献。

> The system...has since become known as Diffie–Hellman key exchange. While that system was first described in a paper by Diffie and me, it is a public key distribution system, a concept developed by Merkle, and hence should be called 'Diffie–Hellman–Merkle key exchange' if names are to be associated with it. I hope this small pulpit might help in that endeavor to recognize Merkle's equal contribution to the invention of public key cryptography. [1]

描述了这个算法的美国专利 4,200,770，已经于1997年4月29日后过期，专利文件表明了Hellman、Diffie和Merkle是算法的发明者。
### 描述

迪菲－赫尔曼通过公共信道交换一个信息，就可以创建一个可以用于在公共信道上安全通信的共享秘密（shared secret）。

下面展示这个算法：
1. 爱丽丝与鲍伯协定使用 p=23 以及 base g=5.
2. 爱丽丝选择一个秘密整数a(秘)=6(秘), 计算A = g^a mod p并发送给鲍伯。
   `A = 5^6 mod 23 = 8.`
3. 鲍伯选择一个秘密整数b(秘)=15(秘), 计算B = g^b mod p并发送给爱丽丝。
   `B = 5^15 mod 23 = 19.`
4. 爱丽丝计算s(秘) = B^a(秘) mod p
   `19^6 mod 23 = 2(秘).`
5. 鲍伯计算s(秘) = A^b(秘) mod p
   `8^15 mod 23 = 2(秘).`

爱丽丝和鲍伯最终都得到了同样的值，因为在模p下`g^{ab}`和`g^{ba}` 相等。 注意`a, b 和 g^ab mod p = g^ba mod p` 是秘密的。 其他所有的值 – p, g, ga mod p, 以及 gb mod p – 都可以在公共信道上传递。

一旦爱丽丝和鲍伯得出了公共秘密，他们就可以把它用作对称密钥，以进行双方的加密通讯，因为这个密钥只有他们才能得到。 当然，为了使这个例子变得安全，必须使用非常大的a, b 以及 p， 否则可以实验所有 g^{ab} mod {23} 的可能取值(总共有最多22个这样的值, 就算a和b很大也无济于事)。 如果 p 是一个至少 300 位的质数，并且a和b至少有100位长， 那么即使使用全人类所有的计算资源和当今最好的算法也不可能从g, p和g^a mod p 中计算出 a。这个问题就是著名的离散对数问题。注意g则不需要很大, 并且在一般的实践中通常是2或者5。IETF RFC3526 文档中有几个常用的大素数可供使用。
### 图示

下面的图示可以方便你理解每个信息都只有谁知道。（伊芙是一个窃听者—她可以看到爱丽丝和鲍伯的通讯内容，但是无法改变它们）

Let s = 共享密钥。 s = 2
Let a = 爱丽丝的私钥。如 a = 6
Let A = 爱丽丝的公钥。如 A = ga mod p = 8
Let b = 鲍伯的私钥。如 b = 15
Let B = 鲍伯的公钥。如 B = gb mod p = 19
Let g = 公共原根。如 g=5
Let p = 公共质数. 如 p = 23

![image](https://cloud.githubusercontent.com/assets/8087928/15953168/67fadc9a-2e8d-11e6-82fb-3b7afd1d25c9.png)
