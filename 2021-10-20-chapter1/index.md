# ·密码学原理课程 —— Introduction


### Course Overview

**Cryptology**: Science concerned with data communication and storage in secure and usually secret form.

- Cryptography (密码学)
- Cryptanalysis (密码分析学)

**Our goals**: the basic principles of modern cryptography

- mathematical assumptions (数学假设)
- security proof (安全证明)
- some standard (secure) constructions (一些现行标准密码方案的构造)

**Textbook:** [Introduction to MODERN CRYPTOGRAPHY 3rd edition](http://www.cs.umd.edu/~jkatz/imc.html)



### Course Structure

{{< image src="https://raw.githubusercontent.com/wgblikeW/blog-imgs/main/Introduction%20To%20Modern%20Cryptography.svg" caption="Introduction To Modern Cryptography" src_s="https://raw.githubusercontent.com/wgblikeW/blog-imgs/main/Introduction%20To%20Modern%20Cryptography.svg" src_l="https://raw.githubusercontent.com/wgblikeW/blog-imgs/main/Introduction%20To%20Modern%20Cryptography.svg" >}}

|                    | Symmetric                    | Asymmetric            |
| :----------------: | ---------------------------- | --------------------- |
|   **Encryption**   | Private-Key Encryption       | Public-Key Encryption |
| **Authentication** | Message Authentication Codes | Digital Signature     |

  课程由古典密码学开始，讲述经典的古典密码（*Caesar Cipher* & *Vigenère Cipher*）以及从其中所抽象出来的编码思想（单表置换密码与多表置换密码）。这个时期的密码更像是个人的灵光乍现（*heuristic*），没有什么固定的范式和数学基础，其更多的被认为是一种艺术( *the art of writing or solving codes*)，一种个人通过巧妙的构思运用简单/相对复杂的规则将明文空间映射到密文空间上规则（这已经运用到了现代密码学的术语）。接着课程教授展示了运用现代密码分析学技巧破解上述两种古典密码方案的攻击方式（频率分析方法），由于古典密码学不是本课程的重点，因此不在赘述，有兴趣的朋友可以参考教材（<u>Pg. 9</u>）。

> 多表位移（*poly-alphabetic shift*）密码成为了现代密码学中流密码（*Stream Cipher*）的源泉，而多表代换密码（*poly-alphabetic shift*）成为了现代密码学中分组密码（*Block Cipher*）的源泉。

  紧接着是现代密码学。在现代密码学中我们可以看到一种严谨的数学形式化表达，不同于古典密码，现代密码学有着一套严格且规范的数学定义去描述密码方案以及密码方案安全性，于此同时发展出了一种证明密码学方案安全性的规约证明方式（*Deduction*）。现代密码学有两条明显的主干，一是对称密码学也叫做私钥密码学（*Private-Key Cryptography OR Symmetric Cryptography*），另一是公钥密码学也叫做非对称密码学（*Public-Key Cryptography OR Asymmetric Cryptography*）。

> Dolev-Yao威胁模型：The **Dolev–Yao model**,[[1\]](https://en.wikipedia.org/wiki/Dolev–Yao_model#cite_note-DolevYao-1) named after its authors [Danny Dolev](https://en.wikipedia.org/wiki/Danny_Dolev) and [Andrew Yao](https://en.wikipedia.org/wiki/Andrew_Yao), is a [formal model](https://en.wikipedia.org/wiki/Mathematical_model) used to prove properties of interactive cryptographic protocols. （From WiKipedia）
>
> Pseudo-random Number Gerneration & Multi-party Secure Computation (MPC) 以及上述的Dolev-Yao Model都是 Yao 在密码学领域的开创性工作
>
> By the way, Andrew Yao 是清华大学交叉信息研究院院长姚期智院士

  在非对称密码学中我们将讲述公钥加密方案（*Public-Key Encryption*）与数字签名（*Digital Signature*），这两者涉及到了密码学中的两大方向，认证（*Authentication*）与加密（*Encryption*）也就是关于如何保护信息的机密性与完整性。

  在对称密码学中我们将讲述消息认证码（*Message Authentication Code* *MAC*）、流密码（*Stream Cipher*）、分组密码（*Block Cipher*）和哈希函数及其应用（*Hash Function and applications*）。

  这里引用教材中由古典密码学转向现代密码学期间发生的方法论上的重大转变，

> In short, cryptography has gone from a **heuristic set of tools** concerned with ensuring **secret communication** for **the military** to a **science** that helps **secure systems** for **ordinary people** all across the globe. 

  在本课程中，我们会对上述两种不同的密码学方向以及组成它们的各个部件给出其精确的形式化数学定义，并建立Dolev-Yao Model对密码方案的安全性作出证明。在学习本课程中要始终牢记一点，即密码学是建立在一定的基础假设之上的，我们需要假设攻击者所具有的能力，需要考虑攻击者所拥有的可能攻击算法的算法复杂度，至一点至关重要。我们考虑的始终是攻击者在拥有某种时间复杂度算法的条件下，该方案被攻破的概率，因此要想充分理解这其中的精髓我们还需要对概率论与计算复杂度理论有一定的了解，明白了这些我们才能更加深入的理解密码学理解信息科学。



### Private-Key Setting

​	在本主题下我们简要介绍一下当我们说Symmetric Encryption时我们究竟在说什么？Symmetric Encryption中文译为对称加密，与其简单的称其为对称加密不如完整的称其为对称加解密。在古典密码学一章中我们知道了通信双方想要在一个公开信道下实现秘密通信需要对原先要传输的信息/明文（*plaintext*）（我们不妨将其认为是具有语义的信息）进行一个变换，使其转变为窃听者无法从其获取到明文的信息的密文(*ciphertext*)数据，在这里我们需要对变换作出定义，这种定义就构成了我们的密码方案，而所有的这些密码方案都需要基于一个保密的信息，也就是我们所熟知的密钥（*Key*）。在本主题Private-Key下，我们所讨论的密钥在通信的双方的背景之下是相同的。

考虑以下这个场景

> Alice和Bod是需要秘密通信的双方，它们事先私下会面，共同商讨了一个用于他们后续秘密通信的密钥，并对此密钥在接下来通信中的使用达成了共识。紧接着Alice和Bod回到各自的住所（空间上是相互隔离的），开始建立通信（这里的通信信道多种多样，你可以飞鸽传书也可以使用互联网），Alice和Bod使用它们预先商讨好的密钥对消息进行加密，并把加密好的密文推入信道。如果这时候他们通信的信道被一个第三者Eve截获（Eve只是个监听者），Eve无法在没有密钥的情况下获得关于明文的任何信息，这样便做到了在信息被敌手（*adversary*）截获的条件下保证消息的机密性。而Alice/Bod将从信道中取出密文并使用密钥对其进行解密，便可以恢复对方所传输的明文消息。（不考虑信息的完整性，即消息在传输的过程中是否被篡改）在这里加密和解密所使用到的密钥都是相同的，这也是Symmetric的含义，即通信双方实际上拥有的东西是等同的。

{{< image src="https://image.p1nant0m.xyz/image-20211121145210046.png" caption="通信双方使用一个安全信道进行通信" src_s="https://image.p1nant0m.xyz/image-20211121145210046.png" src_l="https://image.p1nant0m.xyz/image-20211121145210046.png" >}}

​	在这里我们注意到一点就是我们一直强调的是密钥的机密性而不是强调密码方案的机密性，这点我们将会在后面的Kerckhoffs' principle中提到。

​	书中还给出了另外一种同一用户在本地存储介质上存储机密数据的场景，在此便不在赘述，有兴趣的朋友可以参考教材（<u>Pg. 4</u>）



### The Synatax of Encryption

​	在本主题中我们简要介绍一下加密的语义。对于一个Private-Key加密方案来说，我们首先需要指定一个明文空间 $M$,一个用于生成随机数的随机数生成器 $Gen$ ，一个用于加密消息的加密器 $Enc$ 和一个用于解密的解密器 $Dec$ . 我们对者四者 $(M,Gen,Enc,Dec)$ 有以下说明

-   $Gen$ 所使用的密钥生成算法是一个概率算法，这意味着每一次输入都可能输出符合某个概率分布的随机变量，我们在这并不对 $Gen$ 的概率分布作任何要求。（不过在现实的应用中 $Gen$ 的输出最好应该使得信息熵最大化）
- $Enc$ 接受两个输入，一个是密钥Key记作 $k$ ，一个是消息Message记作 $m$，其输出一个密文Ciphertext记作 $c$ ，我们将 $Enc_k(m)$ 称作使用密钥 $k$ 加密消息 $m$
- $Dec$ 接受两个输入，一个是密钥Key记作 $k$，一个是密文Ciphertext记作 $c$ ，其输出密文对应的明文Message记作 $m$，我们将 $Dec_k(c)$称作使用密钥 $k$ 解密密文 $c$

​	一个加密方案必须满足下列的正确性需求：对于任意密钥生成器 $Gen$ 输出的密钥 $k$ 和所有 $m \in M$，下列等式恒成立
$$
Dec_k(Enc_k(m)) = m
$$
所有由密钥生成算法（*key-generation algorithm*）生成的密钥集合称为密钥空间，记为 $K$ 。在大多数情况下 $Gen$ 的输出在密钥空间 $K$ 中服从均匀分布（*uniformly distribution*）。使用上述提到的工具，我们能够使用数学语言来描述上述我们提到的 Alice-Bob 秘密通信的场景。

> Alice 与 Bod 通过秘密信道协商并确定使用的密钥 $k<- Gen(1^n)$ 
>
> Alice 使用密钥 $k$ 与 Bob 在公开信道下进行加密通信 $c:=Enc_k(m)\space m \in M$
>
> Bod 在公开信道中收到密文 $c$ 并用密钥 $k$ 恢复明文 $m$，$m:=Dec_k(c)$



### Kerckhoffs' principle

​	下列是教材中对Kerckhoffs' principle 的描述

> The cipher method must not be required to be secret, and it must be able to fall into the hands of the enemy without inconvenience.

​	简而言之，一个密码方案的安全性不应该依赖于整个密码方案的保密性，而应该仅仅依赖于密钥的保密性。（保密性：除了合法的通信双方，该信息不为第三方所知）

​	这么做的好处可以简要概括为以下几点：

- 对一个短密钥的持续保密要比持续保密一整个加密方案要来的简单
- 当密钥泄露时，依照 Kerckhoffs' principle，我们可以仅仅替换密钥来最大程度的降低信息系统所面临的风险而不需要大费周章的更换一整套加密方案
- 再者公开机密方案有助于让同行审议以找出现存密码方案的弱点，并可以促进相关标准的制定以及有关密码方案的商业化

### Top Conferences

对学术感兴趣的小伙伴可以在下列 Top Conferences 中获得快乐：）

- [IACR](https://www.iacr.org/) (International Association for Cryptologic Research)
- [CACR](https://cacr.iu.edu/) (Center for Applied Cybersecurity Research)
- [S&P](https://www.ieee-security.org/) (IEEE Computer Society's Technical Committee on Security and Privacy)
-  [CCS](https://dl.acm.org/conference/ccs) (The ACM Conference on Computer and Communication Security)
- [USENIX Security](https://www.usenix.org/) 
- [NDSS](https://www.ndss-symposium.org/) (The Network and Distributed System Security Symposium)



### 杂谈

开设这个专栏的本意是为了重新梳理一下密码学原理课程的知识点还有就是为学弟学妹们留下一点东西，可能有些表述不太准确，言不达意，还望各位大佬不吝赐教。有关文章中出现的问题，各位可以将意见发送邮件至这个邮箱 `wgblike@gmail.com` 

