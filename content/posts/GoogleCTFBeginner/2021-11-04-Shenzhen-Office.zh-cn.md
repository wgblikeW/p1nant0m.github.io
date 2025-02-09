---
title:      "Shenzhen-Office"
subtitle:   "记录一道在GoogleCTFBeginner2021中的一道很有意思的题，涉及编码和小众编程语言。"
date:       2021-11-04 12:00:00
author:     "p1nant0m"
lightgallery: true
tags:
    - GoogleCTFBeginner2021
    - MISC
---





#### 0X1 简要概述

题目给出了两个文件：

-encodings (这里面涉及的内容均为该Chall的提示)

-chall.txt （根据encodings编码后的内容）

本题的基本思路就是根据encodings所给的hints去一层层的解chall.txt中的编码。

#### 0x2 EXPLOITED

chall.txt中的内容大致如下，

{{< image src="https://image.p1nant0m.xyz/4d0c3ead-1e13-4622-b5d9-4935534a28d3.png" caption="chall.txt中的内容" src_s="https://image.p1nant0m.xyz/4d0c3ead-1e13-4622-b5d9-4935534a28d3.png" src_l="https://image.p1nant0m.xyz/4d0c3ead-1e13-4622-b5d9-4935534a28d3.png" >}}

观察到chall.txt文件中的内容均为可见字符，根据encodings中的提示{***a weird base, much higher than base64***} 考虑是base家族的编码并且该编码不常见。文件中的内容均为Unicode字符（UCS-2编码），范围达到65535，Googling base65535，找到一个在线[Decoder](https://www.better-converter.com/Encoders-Decoders/Base65536-Decode)。解码后得到以下内容，

{{< image src="https://image.p1nant0m.xyz/53870b26-d634-4b9e-ba70-eca12785fe2a.png" caption="解码后内容" src_s="https://image.p1nant0m.xyz/53870b26-d634-4b9e-ba70-eca12785fe2a.png" src_l="https://image.p1nant0m.xyz/53870b26-d634-4b9e-ba70-eca12785fe2a.png" >}}

均为Hex表示，使用xxd将其转换为二进制文件，

```
xxd -r -p base65535Decode.txt base65535_decoded.bin
```

使用file查看base65535_decoded.bin的类型，见下图

{{< image src="https://image.p1nant0m.xyz/204494e4-02a1-46e7-b8d9-916bf2ee2edc-16360259177581.png" caption="查看文件类型" src_s="https://image.p1nant0m.xyz/204494e4-02a1-46e7-b8d9-916bf2ee2edc-16360259177581.png" src_l="https://image.p1nant0m.xyz/204494e4-02a1-46e7-b8d9-916bf2ee2edc-16360259177581.png" >}}

我们得到一张jpeg格式的图片，转换扩展名打开图片，我们得到下图，

{{< image src="https://image.p1nant0m.xyz/4215c422-296a-43e0-853c-5f1a14242097-16360259268892.png" caption="提取出的 jpeg 文件" src_s="https://image.p1nant0m.xyz/4215c422-296a-43e0-853c-5f1a14242097-16360259268892.png" src_l="https://image.p1nant0m.xyz/4215c422-296a-43e0-853c-5f1a14242097-16360259268892.png" >}}

谢谢老番茄让我知道这是什么东西，这是画家蒙德里安的著作，根据encodings中的提示{***a language named after a painter***}，我们googling一下可以找到该语言Piet，并且还有其在线[解释器](https://www.bertnase.de/npiet/npiet-execute.php)，不过我把图片丢进去没啥效果，好像不是这副图片，我们得继续挖掘一下这幅图片，祭出我所有关于图片隐写的工具。在尝试了多种工具后，我在exiftool中提取到了有趣的数据，

```
exiftool base65535_decoded.jpeg > Piet.txt
```

稍微处理掉一些我们不感兴趣的信息，得到

{{< image src="https://image.p1nant0m.xyz/a6cc5067-80b6-4a78-899e-d17908f13d54.png" caption="处理后得到的文件内容" src_s="https://image.p1nant0m.xyz/a6cc5067-80b6-4a78-899e-d17908f13d54.png" src_l="https://image.p1nant0m.xyz/a6cc5067-80b6-4a78-899e-d17908f13d54.png" >}}

看似又是一种奇怪的语言：（  根据encodings中的提示{***a language that is the opposite of good***}，我一开始想的是Bad，找了半天没找到，后来我干脆直接去找Good的反义词，

{{< image src="https://image.p1nant0m.xyz/dafe187d-a7bc-4c5f-aac8-a483f4259fce.png" caption="good 的反义词" src_s="https://image.p1nant0m.xyz/dafe187d-a7bc-4c5f-aac8-a483f4259fce.png" src_l="https://image.p1nant0m.xyz/dafe187d-a7bc-4c5f-aac8-a483f4259fce.png" >}}

在互联网世界遨游了一番后终于确定了那门语言——[Evil](https://esolangs.org/wiki/Evil)，故技重施找在线Interpreter，但没找着。无奈只好在[主页](http://web.archive.org/web/20070103000858/www1.pacific.edu/~twrensch/evil/index.html)下载了用Java实现的Interpreter。

用javac编译一下后再执行我们得到的代码（Piet.txt）并将输出结果重定向至Evil.txt。我们又得到了一连串Hex，


{{< image src="https://image.p1nant0m.xyz/09e0e3a3-3939-4754-a377-18215964cf59.png" caption="代码执行后输出内容" src_s="https://image.p1nant0m.xyz/09e0e3a3-3939-4754-a377-18215964cf59.png" src_l="https://image.p1nant0m.xyz/09e0e3a3-3939-4754-a377-18215964cf59.png" >}}

故技重施，用xxd转换为二进制文件，并用file检查文件类型，发现是一个zlib压缩文件，使用zlib-flate解压该文件，

```
zlib-flate -uncompress < Evil.bin > Evil_uncomp
```

使用File检查该文件，发现是一个gzip包，解压后得到一个Netpbm格式的文件，googling一下，

> **Netpbm** (formerly Pbmplus) is an [open-source](https://en.wikipedia.org/wiki/Open-source_software) package of **graphics programs** and a programming library. It is used mainly in the [Unix](https://en.wikipedia.org/wiki/Unix) world, where one can find it included in all major open-source [operating system](https://en.wikipedia.org/wiki/Operating_system) distributions, but also works on [Microsoft Windows](https://en.wikipedia.org/wiki/Microsoft_Windows), [macOS](https://en.wikipedia.org/wiki/MacOS), and other operating systems.[[2\]](https://en.wikipedia.org/wiki/Netpbm#cite_note-2)

这时候我们可以将这个图片丢进Piet的Interpreter里了，我们又得到了一连串Hex，重复上述转换过程，最终我们得到一个ASCII文档，其中的内容为

{{< image src="https://image.p1nant0m.xyz/79392036-914e-4c39-81df-b573ea9c217f.png" caption="文本文件内容" src_s="https://image.p1nant0m.xyz/79392036-914e-4c39-81df-b573ea9c217f.png" src_l="https://image.p1nant0m.xyz/79392036-914e-4c39-81df-b573ea9c217f.png" >}}

根据encodings中的提示{***language that looks like a rainbow cat***}，googling后定位到[nya~](http://esolangs.org/wiki/Nya~)语言。由于nya~的语法比较简单，我们可以用python自实现。

| Command | Description         |
| :-----: | ------------------- |
|    n    | Decrement integer   |
|    y    | Increment integer   |
|    a    | Put char from ASCII |
|    ~    | Reset integer to 0  |



```python
#!/usr/bin/env python3
with open('./Piet_out_uncomp', 'r') as f:
    data = f.read().splitlines()

for sent in data:
    print(chr(len(sent)-4), end='')
```

得到一个长整数，

```
440697918422363183397548356115174111979967632241756381461523275762611555565044345243686920364972358787309560456318193690287799624872508559490789890532367282472832564379215298488385593860832849627398865422864710999039787979733217240717198641619578634620231344233376325369569117210379679868602299244468387044128773681334105139544596909148571184763654886495124023818825988036876333149722377075577809087358356951704469327595398462722928801
```

这里就不能够再转换为Binary了，至此我们还剩下两个Hints{***a language that is too vulgar to write here；a language that ended in 'ary' but I don't remember the full name***}

经过一番艰难的Googling之后，我最终确定了剩下的两门语言，Brainfuck和Unary。

上面的大整数可以转换为二进制按照Unary转换表转换为Brainfuck可以识别的语言，使用的Python脚本如下，

```python
data = 440697918422363183397548356115174111979967632241756381461523275762611555565044345243686920364972358787309560456318193690287799624872508559490789890532367282472832564379215298488385593860832849627398865422864710999039787979733217240717198641619578634620231344233376325369569117210379679868602299244468387044128773681334105139544596909148571184763654886495124023818825988036876333149722377075577809087358356951704469327595398462722928801
trans2_brain_fuck = bin(data)[3:]

dict_map = {'000': '>', '001': '<',  '010': '+', '011': '-', '100': '.', '101': ',',
            '110': '[', '111': ']'}

output = ''
for i in range(0, len(trans2_brain_fuck), 3):
    print(trans2_brain_fuck[i:i+3])
    output += dict_map[trans2_brain_fuck[i:i+3]]
print(output)
# [-]>[-]<++++++[>++++++++++<-]>+++++++.<+[>++++++++++<-]>+++++++.<+[>----------<-]>----.<+++++[>++++++++++<-]>+++.<+[>----------<-]>-.<[>----------<-]>----.<+++++[>----------<-]>-------.<[>++++++++++<-]>+.<++++++[>++++++++++<-]>+++.<++++++[>----------<-]>----.<++++[>++++++++++<-]>++++.<+[>++++++++++<-]>+++++.<++++++[>----------<-]>--.<++++[>++++++++++<-]>+++++++.<+[>++++++++++<-]>++++.<++++++[>----------<-]>-.<[>++++++++++<-]>++++.<++++++[>++++++++++<-]>++.<+[>++++++++++<-]>+.<
```

最后再Googling找到Brainfuck的[Online Interpreter](https://copy.sh/brainfuck/)后运行程序得到Flag : : **CTF{pl34s3_n0_m04r}**



一些”实用“（神奇）的网站：

[Esolang](https://esolangs.org/wiki/Main_Page) 各种奇奇怪怪的编程语言