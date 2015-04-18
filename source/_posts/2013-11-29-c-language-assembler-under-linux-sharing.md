---
layout: post
title: 'C語言在Linux下組裝經驗分享'
date: 2013-11-29 14:19
comments: true
categories: [C, Linux, 組裝]
---
整理組裝經驗如下，全部是Command line的文字模式

- [非Runtime](#非Runtime)
    - [找出static library symbols](#nr1)
    - [找出library搜尋路徑](#nr2)
    - [列出CC 預設define和相關資訊](#nr3)
- [Runtime](#Runtime)
    - [dump console 輸出文字](#r1)
    - [當dynamic library 不在/無法放到linker搜尋路徑造成檔案無法執行](#r2)
    - [看程式system call的行為](#r3)
    - [Linker/loader 相關](#r4)
- [參考資料](#參考資料)

---
<a name="非Runtime"></a>
## 非Runtime
<a name="nr1"></a>

- 找出static library symbols
   - `find | grep *\\.a$ | xargs nm -A | grep 要找的symbol`

<a name="nr2"></a>

- 找出library搜尋路徑
    - `ld --verbose | grep SEARCH | tr "; " "\n\r"`

<a name="nr3"></a>

- 列出CC 預設define和相關資訊
    - `echo "" | gcc -E -xc - -dM -v`
- grep 加regular expression萬歲

---
<a name="Runtime"></a>
## Runtime
<a name="r1"></a>

- dump console 輸出文字
  - script
    - `script 文字輸出檔檔名`
    - 操作
    - 按`ctrl+d`結束dump
  - 執行檔 2>&1 | tee 輸出檔名
    - 變形
      - 執行檔 2>&1 | grep 想找的文字 | tee 輸出檔名
<a name="r2"></a>

- 當dynamic library 不在/無法放到linker搜尋路徑造成檔案無法執行
    - $`LD_LIBRARY_PATH=$LD_LIBRARY_PATH:你的library路徑 執行檔`

<a name="r3"></a>

- 看程式system call的行為
    - `strace 執行檔`
    - `strace -e open 執行檔`
        - 看open system call，常用，因為有時候程式爛掉是因為開檔案失敗，用這個找很快
    - `strace -f 執行檔`
        - 包含fork的process

<a name="r4"></a>

- Linker/Loader 相關
    - ld.so會和路徑以及平台相關，以Ubuntu LTS 12.04 X86有
        - /lib64/ld-linux-x86-64.so.2
            - 64-bit
        - /lib/ld-linux.so.2
            - 32-bit
    - 透過dynamic linker/loader 看檔案link的library
        - `LD_TRACE_LOADED_OBJECTS=1 ld.so 執行檔`
    - 觀察dynamic linker/loader行為
        - `LD_DEBUG=help /bin/ls`
            - 顯示選項
        - `LD_DEBUG=all 執行檔`
            - 全部都顯示
- 合體
    - ` LD_DEBUG=all strace -f ld.so strace -f 執行檔`

<a name="參考資料"></a>
## 參考資料

- `man ld.so`
- [TLDP: Shared Libraries](http://tldp.org/HOWTO/Program-Library-HOWTO/shared-libraries.html)
