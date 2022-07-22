# FileStore




#### 0X1 Source Code

```python
# Copyright 2021 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.


import os, secrets, string, time
from flag import flag



def main():
    # It's a tiny server...
    blob = bytearray(2**16)
    files = {}
    used = 0


    # Use deduplication to save space.
    def store(data):
        nonlocal used
        MINIMUM_BLOCK = 16
        MAXIMUM_BLOCK = 1024
        part_list = []
        while data:
            prefix = data[:MINIMUM_BLOCK]
            ind = -1
            bestlen, bestind = 0, -1
            while True:
                ind = blob.find(prefix, ind+1)
                if ind == -1: break
                length = len(os.path.commonprefix([data, bytes(blob[ind:ind+MAXIMUM_BLOCK])]))
                if length > bestlen:
                    bestlen, bestind = length, ind


            if bestind != -1:
                part, data = data[:bestlen], data[bestlen:]
                part_list.append((bestind, bestlen))
            else:
                part, data = data[:MINIMUM_BLOCK], data[MINIMUM_BLOCK:]
                blob[used:used+len(part)] = part
                part_list.append((used, len(part)))
                used += len(part)
                assert used <= len(blob)


        fid = "".join(secrets.choice(string.ascii_letters+string.digits) for i in range(16))
        files[fid] = part_list
        return fid


    def load(fid):
        data = []
        for ind, length in files[fid]:
            data.append(blob[ind:ind+length])
        return b"".join(data)


    print("Welcome to our file storage solution.")


    # Store the flag as one of the files.
    store(bytes(flag, "utf-8"))


    while True:
        print()
        print("Menu:")
        print("- load")
        print("- store")
        print("- status")
        print("- exit")
        choice = input().strip().lower()
        if choice == "load":
            print("Send me the file id...")
            fid = input().strip()
            data = load(fid)
            print(data.decode())
        elif choice == "store":
            print("Send me a line of data...")
            data = input().strip()
            fid = store(bytes(data, "utf-8"))
            print("Stored! Here's your file id:")
            print(fid)
        elif choice == "status":
            print("User: ctfplayer")
            print("Time: %s" % time.asctime())
            kb = used / 1024.0
            kb_all = len(blob) / 1024.0
            print("Quota: %0.3fkB/%0.3fkB" % (kb, kb_all))
            print("Files: %d" % len(files))
        elif choice == "exit":
            break
        else:
            print("Nope.")
            break


try:
    main()
except Exception:
    print("Nope.")
time.sleep(1)
```

#### 0x2 ANALYSIS

简要的审计chall源码可以发现如下几个事实：

- 程序为我们提供了三种功能：load，store，status
  - load：我们提供一个文件UID，系统返回我们该文件中的内容
  - store：系统存储我们输入信息并返回该文件对应的UID
  - status：系统虚拟磁盘所使用的存储空间大小
- Flag存储在系统中并且大小至少（这里说是至少是因为该文件系统使用了一种压缩）为26B
- 文件系统将整个存储池分为多个Blocks，每个Blocks的大小为16B，存储采用最长前缀匹配压缩存储，如果两个文件（HEX表示）具有匹配的前缀，则后者的加入不需要新占用存储空间（不需要一个新的Block），每个文件以线性表结构描述，表中的内容为Block（index，len）
- 我们能够利用该文件系统的特性泄露flag信息，即使用BF逐字符Store进文件系统，观察Status中已使用内存大小变化来确定我们输入的字符是否正确

#### 0X3 EXPLOITED

```python
from pwn import *
from time import sleep
import string, time
def main():
    match = "0.026"


    def process_conn_action(store_item):
        nonlocal match
        conn = remote("filestore.2021.ctfcompetition.com", 1337)
        conn.recv()
        conn.send('store\r\n')
        conn.recv()
        conn.send(store_item + '\r\n')
        conn.recv()
        conn.send('status\r\n')
        info = conn.recvall(timeout=1)
        info = info.decode()
        info = info[info.find('Quota') +7: info.find('Quota') +12]
        if match == info:
            return True
        else:
            return False
    
    
    # print(process_conn_action('c'))
    probchar = ['0', '1', '3', '4', 'c', 'f', 'i', 'n', 'p', 't', 'u', 'C', 'F', 'M', 'R', \
        'T', '_', '{', '}', 'd']
    # dict = string.printable
    # for char in string.printable:
    #     if char not in probchar and process_conn_action(char):
    #         probchar.append(char)
    #         print(probchar)
    #         print(len(probchar))
    
    # print(len(probchar))


    flag = "}" # 实践中发现前序遍历会有Bug，便将这里改为由后向前匹配
    while True:
        for character in probchar:
            temp = character + flag
            print("Temp: " + temp)
            print("Current Character: " + character)
            if process_conn_action(temp):
                flag = character + flag
                print("Found! ! !")
                sleep(1)
                break
            else:
                continue


main()
# CTF{CR1M3_0f_d3dup1ic4ti0n}
```
