---
layout: post
title: '系統函式庫的debug 資訊放在那邊？'
date: 2015-02-20 17:17
comments: true
categories: 
---
在查詢hello world中的`_start`呼叫`__libc_start_main`部份，使用到了反組譯工具。觀察反組譯的部份發現有可能需要看一下libc本身的`__libc_start_main`組合語言的行為。以前的經驗，這種情況先拉有debug 資訊的套件來看，所以拉了libc6-dbg下來，結果下來的結果，<font color="red">完全無法反組譯</font>。後來請教網友[Scott Tasi](http://scottt.tw/)才發現我錯很大。

先講結論：

* libc的debug 資訊和本身的binary完全隔開。所以反組譯要看的仍然是在原本的libc.so這個檔案。

年紀大了才發現能夠從結論中問問題收穫會更多，所以我就來問

* 問題一：誰來用這些debug 資訊?
* 問題二：既然debug 資訊不在binary內？那麼怎麼找到額外的debug 資訊?

先來回答誰來用這些debug 資訊？其實這個根本就是廢話，gdb用心酸的啊。其實這只是在鋪梗，這表示找第二個問題的答案就會和gdb有很大的關係，也就是說我們可以把問題縮小到和gdb相關 [（註一）](#rk1)。

好，我們來重複問題二

* 既然debug 資訊不在binary內？那麼怎麼找到額外的debug 資訊?

從常理來猜測，顯然是原本的binary有地方告訴gdb「有額外的除錯檔案，請你載入的時候去那邊抓除錯資訊」。我們直接看[GDB手冊](https://sourceware.org/gdb/onlinedocs/gdb/Separate-Debug-Files.html)怎麼說

* gdb允許把debug 資訊放在binary外面，而gdb可以透過兩種方式取得
	* 執行的binary內提供link讓gdb可以摸到有debug 資訊的檔案。如果是執行檔，可能就稱為`執行檔.debug`。
		* 這個link資料，放兩個東西
			* 不包含directory的檔案名稱
			* 該檔案名稱算出的CRC 碼
  * 透過build ID，因為和我找的無關，跳過。我們focus在第一個方式。

有了debug link資訊後，仍然有幾個細節需要釐清

* 問題三：debug 資訊檔案放在哪個目錄？
* 問題四：debug link放在binary 的哪裡？

一樣來看手冊。手冊上說debug 資訊檔案會依以下的順序搜尋

* 該執行檔的存放目錄
* 該執行檔的存放目錄下面同樣目錄名稱，但是加上.debug。如/usr/bin -> /usr/bin.debug/
* 系統預設的debug 目錄

以上是問題三的的解答，剩下問題四我就認為可以把整個故事說完。所以來看問題四吧。

* 問題四：debug link放在binary 的哪裡？

同樣的在手冊中有提到，binary 中有特別的section稱為`.gnu_debuglink`，就是存放debug 資訊檔案的資訊。存放的資訊為

* 存放debug 資訊的檔案名稱
* padding for 4-byte alignment
* 4 byte CRC checksum

好啦，有了完整的故事，當然要看是不是在唬爛。我們來看libc.so吧

```
$ pwd
/lib/arm-linux-gnueabi
$ objdump -h libc-2.19.so | grep debug
 68 .gnu_debuglink 00000014  00000000  00000000  001336a0  2**0
$ objdump -s -j .gnu_debuglink libc-2.19.so 

libc-2.19.so:     file format elf32-littlearm

Contents of section .gnu_debuglink:
 0000 6c696263 2d322e31 392e736f 00000000  libc-2.19.so....
 0010 1b8f248c                             ..$. 
 
$ sudo apt-get install libc6-dbg
Reading package lists... Done
Building dependency tree       
Reading state information... Done
The following NEW packages will be installed:
  libc6-dbg
0 upgraded, 1 newly installed, 0 to remove and 0 not upgraded.
Need to get 3,290 kB of archives.
After this operation, 18.1 MB of additional disk space will be used.
Get:1 http://不告訴你/debian/ jessie/main libc6-dbg armel 2.19-13 [3,290 kB]
Fetched 3,290 kB in 4s (701 kB/s)      
Selecting previously unselected package libc6-dbg:armel.
(Reading database ... 46987 files and directories currently installed.)
Preparing to unpack .../libc6-dbg_2.19-13_armel.deb ...
Unpacking libc6-dbg:armel (2.19-13) ...
Setting up libc6-dbg:armel (2.19-13) ...
$ find /usr/lib/debug/ |grep libc-2
/usr/lib/debug/lib/arm-linux-gnueabi/libc-2.19.so
```

那麼這個和我原本要反組譯libc的結果有什麼關係？嘛，旅行的精華就是在迷路不是嗎？

---
## 測試環境

```
$ lsb_release -a
No LSB modules are available.
Distributor ID:	Debian
Description:	Debian GNU/Linux 8.0 (jessie)
Release:	8.0
Codename:	jessie

$ uname -a
Linux debian 3.2.0-4-versatile #1 Debian 3.2.65-1+deb7u1 armv5tejl GNU/Linux

$ gcc -dumpmachine
arm-linux-gnueabi
```

---
## 參考資料

* [GDB Manuel: Debugging Information in Separate Files](https://sourceware.org/gdb/onlinedocs/gdb/Separate-Debug-Files.html)
* [fcamel: gdb 如何找到 debug symbol](http://fcamel-life.blogspot.tw/2012/01/gdb-debug-symbol.html)
* [.gnu_debuglink or Debugging system libraries with source code](https://blogs.oracle.com/dbx/entry/gnu_debuglink_or_debugging_system)

---
## 註解
<a name="rk1"></a>
一：精確來說，這應該不是只有gdb才能用，我不過偷懶而已。
