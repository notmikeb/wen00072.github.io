---
layout: post
title: 'nm 的欄位說明'
date: 2014-12-08 14:45
comments: true
categories: [binutils, Linux]
---
在閱讀ld手冊的時候看到symbol table突然想起來我nm只知道`t`、`U`這兩個flag而已。想起來真是慚愧，趕快問一下男人nm顯示出來的欄位意義。一個典型的nm輸出為：
`00000000004005ed T main`

這些代表什麼呢，整理如下：

* symbol存放的值，應該是位址或是offset我猜
* symbol flags，一般來說大寫表示global而小寫表示local
* symbol 名稱


symbol flags詳細意義為：

* `A`: symbol的值是絕對值，之後的link全部就hardcode不會更動這個symbol的值
* `B`: global symbol。該symbol存放在**沒有初始化**的data section (猜測是`.bss`)
* `b`: local symbol。該symbol存放在**沒有初始化**的data section (猜測是`.bss`)
* `C`: common symbol。一種未初始化的資料。**不知道是三小**，男人只有說link多個檔案可能會有不同的common symbol使用相同的名稱，結果還是看不懂。
* `D`: global symbol。該symbol存放在**有初始化**的data section (猜測是`.data`)
* `d`: local symbol。該symbol存放在**有初始化**的data section (猜測是`.data`)
* `G`: global symbol。該symbol存放在是針對小量資料做最佳化、**有初始化**的data section(不是`.data`)
* `g`: local symbol。該symbol存放在是針對小量資料做最佳化、**有初始化**的data section(不是`.data`)
* `I`:表示這個symbol間接地reference 其他的symbol。不過**我還是沒有搞懂是什麼意義**。
* `i`: 在elf下面的話，表示這個symbol是一個間接的function。這表示當link的時候不會解析位址，而是在runtime才會去呼叫真實的位址function。以下是範例，不過**我還是沒有搞懂是什麼意義**。

```c strcat.o
x86_64-linux-gnu/libc.a:strcat.o:0000000000000040 T __GI_strcat
x86_64-linux-gnu/libc.a:strcat.o:                 U __init_cpu_features
x86_64-linux-gnu/libc.a:strcat.o:0000000000000000 i strcat
x86_64-linux-gnu/libc.a:strcat.o:0000000000000040 T __strcat_sse2
x86_64-linux-gnu/libc.a:strcat.o:                 U __strcat_sse2_unaligned
x86_64-linux-gnu/libc.a:strcat.o:                 U __strcat_ssse3
```

* `N`: debug symbol
* `p`: symbol放在[unwind stack](http://en.wikipedia.org/wiki/Call_stack#Unwinding) section。 **不知道這邊的stack unwind和nm的意義是不是一樣?**
* `R`: global symbol。symbol放在read only data section(猜測是`.rodata`)。
* `r`: local symbol。symbol放在read only data section(猜測是`.rodata`)。
* `S`: global symbol。該symbol存放在是針對小量資料做最佳化、**沒有初始化**的data section(不是`.bss`)
* `s`: local symbol。該symbol存放在是針對小量資料做最佳化、**沒有初始化**的data section(不是`.bss`)
* `T`: global symbol。symbol放在程式碼section(猜測是`.text`)。
* `t`: local symbol。symbol放在程式碼section(猜測是`.text`)。
* `U`: undefined symbol
* `u`: unique symbol，GNU 用於ELF時的 symbol binding extenstion。**我還是沒有搞懂是什麼意義**
* `V`: global symbol。weak object，當link時該symbol可以link到正確的symbol就和一般的symbol相同，然而如果link時無法resolve symbol也不會噴錯誤，而是將該symbol的值設成0。(**用在什麼地方？**)
* `v`: local symbol。weak object，當link時該symbol可以link到正確的symbol就和一般的symbol相同，然而如果link時無法resolve symbol也不會噴錯誤，而是將該symbol的值設成0。(**用在什麼地方？**)
* `W`: global symbol。weak defined symbol，weak object的特例。當link時該symbol可以link到正確的symbol就和一般的symbol相同，然而如果link時無法resolve symbol也不會噴錯誤，而是**根據系統實作做適當的處理**。(**用在什麼地方？**)
* `w`: local symbol。weak defined symbol，weak object的特例。當link時該symbol可以link到正確的symbol就和一般的symbol相同，然而如果link時無法resolve symbol也不會噴錯誤，而是**根據系統實作做適當的處理**。(**用在什麼地方？**)
* `-`: stabs symbol，用於a.out格式，**看不懂又和elf無關故跳過**。
* `?`: 不明的symbol
    
## 待釐清

* 名詞解釋
  * common symbol
  * indirect reference
  * indirect function
  * unwind stack section
  * unique symbol
* 用法不明
  * weak object
  * weak defined symbol

## 參考資料

* `man nm`
