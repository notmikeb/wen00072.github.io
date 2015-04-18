---
layout: post
title: 'GNU LD  手冊略讀  (1): Chapter 3 ~ Chapter 3.5'
date: 2014-12-14 23:55
comments: true
categories: 
---
[下一篇](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command)
[回總目錄](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-0-table-of-contents)

## 本篇目錄

* [Linker script 格式概論](#fmt)
* [從Linker script 範例開始](#ex)
* [簡易script 命令格式](#cmd)
  * [檔案相關命令](#cmd-file)
  * [Object檔案相關格式命令](#cmd-obj)
  * [設定記憶體區塊alias命令](#cmd-alias)
  * [未分類的命令 (節錄）](#cmd-misc)
* [設定symbol的值](#assign)
  * [基本運算](#assign-op)
  * [HIDDEN命令](#assign-hid)
  * [PROVIDE命令](#assign-prov)
  * [PROVIDE_HIDDEN命令](#assign-hid-prov)
  * [談談source code和linker script symbol的關係](#assign-src)
  


<a name="fmt"></a>
## Linker script 格式概論

* 以文字檔存放
* 由多個command組成
* command可能是
	* keyword + 參數
  * 設定symbol
  * ...
* command 可以用`;`分開，空白會被忽略
* 使用/* .. */註解
* 字串直接打，如果有用到script保留的字元如`.`可以用`"`包住


<a name="ex"></a>
## 從Linker script 範例開始

```c link script 範例
SECTIONS
{
	. = 0x10000;
	.text : { *(.text) }
	. = 0x8000000;
	.data : { *(.data) }
	.bss : { *(.bss) }
}
```
這個[抄來](https://sourceware.org/binutils/docs/ld/Simple-Example.html#Simple-Example)的範例很簡單，只有一個命令`SECTIONS`。`SECTIONS`是用來描述執行的時候記憶體的規劃配置（layout）。

說明這個指令細節

* `.`表示記憶體位置counter，起始值為0。結束值則由linker 計算把所有input section的資料整合到output section的長度。而`.`如果沒有指定明確的記憶體位址的話，就會被設定為**上一個位址counter的結束位址**。[參考示意圖: (Jim Huang) How GNU Toolchain Works投影片 ](http://www.slideshare.net/jserv/from-source-to-binary-how-gnu-toolchain-works/46)。
* 設定記憶體位置counter為0x10000
* 接下來請把所有**輸入**object檔案的程式機械碼中(`{ *(.text) }`)存放到**輸出**object檔案的`.text`section中。
* 接設定記憶體位置counter為0x8000000
* 先放有初始值的全域變數（.data）
* 再放**沒有**初始值的全域變數（.bss）

另外要注意的是ld會自動幫你處理[alignment的問題](http://en.wikipedia.org/wiki/Data_structure_alignment)，所以不用擔心section之間的aligment問題。


<a name="cmd"></a>
## linker script 命令格式

* `ENTRY(symbol)`
   * 設定某個symbol為程式執行的第一個指令起始點，在我的預設linker script中是`ENTRY(_start)`，然後去反組譯隨便一個C編譯出來的執行檔，找字串`_start`可以看到裏面又去呼叫了`__libc_start_main@plt`。
   
```text hello_word執行檔
Disassembly of section .text:

0000000000400440 <_start>:
  400440:       31 ed                   xor    %ebp,%ebp
  400442:       49 89 d1                mov    %rdx,%r9
  400445:       5e                      pop    %rsi
  400446:       48 89 e2                mov    %rsp,%rdx
  400449:       48 83 e4 f0             and    $0xfffffffffffffff0,%rsp
  40044d:       50                      push   %rax
  40044e:       54                      push   %rsp
  40044f:       49 c7 c0 c0 05 40 00    mov    $0x4005c0,%r8
  400456:       48 c7 c1 50 05 40 00    mov    $0x400550,%rcx
  40045d:       48 c7 c7 2d 05 40 00    mov    $0x40052d,%rdi
  400464:       e8 b7 ff ff ff          callq  400420 <__libc_start_main@plt>
  400469:       f4                      hlt    
  40046a:       66 0f 1f 44 00 00       nopw   0x0(%rax,%rax,1)
```

<a name="cmd-file"></a>
## 檔案相關命令

* `INCLUDE filename`
  * 在看到這個命令的時候才去載入`filename`這個**linker script**。可以被放在不同的命令如SETCTION, MEMORY等。
* `INPUT(file1 file2 ...)`
  * 指定載入的輸入object檔案，如abc.o這樣的檔案。
* `GROUP(file1 file2 ...)`
  * 指定載入的輸入archieve檔案，如libabc.a這樣的檔案。
* `AS_NEEDED(file1 file2 ...)`
  * 在`INPUT`和`GROUP`使用的命令，用來告訴linker說如果object裏面的資料有被reference到才link進來，猜測應該可以減少儲存空間。範例（未測試請自行斟酌）：`INPUT(file1.o file2.o AS_NEEDED(file3.o file4.o))`
* `OUTPUT(filename)`
  * 和`gcc -o filename` 一樣
* `SEARCH_DIR(path)`
  * 和`-L path`一樣
* `STARTUP(filename)`
  * 和INPUT相同，唯一差別是ld保證這個檔案一定是第一個被link

<a name="cmd-obj"></a>
## Object檔案相關格式命令

* `OUTPUT_FORMAT(bfdname)`
  * 指定輸出object檔案的binary 檔案格式，可以使用`objdump -i`列出支援的binary 檔案格式
* `OUTPUT_FORMAT(default, big, little)`
  * 指定輸出object檔案預設的binary 檔案格式，big endian的binary 檔案格式以及little endian的binary 檔案格式。可以使用`objdump -i`列出支援的binary 檔案格式
* `TARGET(bfdname)`
  * 告訴ld用那種binary 檔案格式讀取輸入object檔案要，可以使用`objdump -i`列出支援的binary 檔案格式
 
<a name="cmd-alias"></a> 
## 設定記憶體區塊alias命令

* `REGION_ALIAS(alias, region)`
  * 設定`MEMORY`命令中區塊的alias，一般來說，用在不同的平台需要相同的memory layout時可以使用。舉例來說，當有3個平台，記憶體layout都是相同，那麼可以
      * 將他們平台相關的記憶體區塊`MEMORY`命令寫在個別的檔案如linkcmds.memory
      * 設定相同的alias
      * 在主要的linker script 使用`INCLUDE`載入linkcmds.memory，並且直接使用alias當作一般的區塊使用。

詳細的範例說明可以看[這邊](https://sourceware.org/binutils/docs/ld/REGION_005fALIAS.html#REGION_005fALIAS)
```c 範例
     INCLUDE linkcmds.memory
     
     SECTIONS
       {
         .text :
           {
             *(.text)
           } > REGION_TEXT
         .rodata :
           {
             *(.rodata)
             rodata_end = .;
           } > REGION_RODATA
         .data : AT (rodata_end)
           {
             data_start = .;
             *(.data)
           } > REGION_DATA
         data_size = SIZEOF(.data);
         data_load_start = LOADADDR(.data);
         .bss :
           {
             *(.bss)
           } > REGION_BSS
       }

```

<a name="cmd-misc"></a> 
## 未分類的命令 (節錄）

* `ASSERT(exp, message)`
	* 條件不成立就噴訊息並結束link
* `EXTERN(symbol1 symbol2 ...)`
  * 強迫讓指定的symbol設成undefined，手冊說一般用在刻意要使用非標準的API。例如自幹printf時可以用這個命令。 (不過變成了undefine symbol怎麼link??)
* `FORCE_COMMON_ALLOCATION`
  * 手冊和男人說和相容性有關，手冊上是說強迫分配空間給common symbols，即使是link relocate檔案。(common symbols不知道是什麼) 
   
* `OUTPUT_ARCH(bfdarch)`
  * 指定輸出的平台，可以透過`objdump -i`查詢支援平台
* `INSERT [ AFTER | BEFORE ] output_section`
  * 指定在預設linker script命令被執行之前或是之後加上或加入特定的輸入section到輸出section。以下是一個範例

```
SECTIONS
{
	OVERLAY :
	{
		.ov1 { ov1*(.text) }
		.ov2 { ov2*(.text) }
	}
}
INSERT AFTER .text;
```

<a name="assign"></a>

## 設定symbol的值
linker script提供設定**symbol**數值的方法。要注意的是，這邊的symbol可以指一個全域變數、`SECTION`命令中的location counter（就是`.`開頭的資料如`.text`）

使用方式介紹如下：

<a name="assign-op"></a>
## 基本運算


```c symbol assignment operations
symbol = expression ;
symbol += expression ;
symbol -= expression ;
symbol *= expression ;
symbol /= expression ;
symbol <<= expression ;
symbol >>= expression ;
symbol &= expression ;
symbol |= expression ;
```

關於expression是三小後面會再討論。

[手冊上提供的範例](https://sourceware.org/binutils/docs/ld/Simple-Assignments.html#Simple-Assignments)是

```c symbol assign範例
floating_point = 0;
SECTIONS
{
	.text :
	{
	*(.text)
	_etext = .;
	}
	_bdata = (. + 3) & ~ 3;
	.data : { *(.data) }
}
```

從這邊可以看到幾種assign

* 設定全域變數`floating_point`的symbol為0
* 設定全域變數`_etext`的值為輸入object檔案`.text`合體後的offset，個人猜測可以理解成end of text。(回顧一下`.`是offset counter)
* 設定全域變數`_bdata`的值為輸出object檔案`.text`結尾的offset 的**4的倍數**位址。這邊透露兩個資訊
  * 個人猜測可以理解成begin of data
  * 四的倍數和[alignment的問題](http://en.wikipedia.org/wiki/Data_structure_alignment)應該有關聯。

<a name="assign-hid"></a>
## HIDDEN命令
* `HIDDEN(要隱藏的symbol)`
可以把他理解成加了`static`的全域變數，也就是說這個symbol只在這個處理範圍中才能摸到。

<a name="assign-prov"></a>
## PROVIDE命令

* `PROVIDE命令(symbol = expression)`
  * 簡單來說，如果你的程式已經有這個symbol（函數或變數），就用你的；否則就使用這邊提供的symbol。手冊上說是給特殊的linker使用的。想知道他提的use case可以看[這邊](https://sourceware.org/binutils/docs/ld/PROVIDE.html#PROVIDE)，我是沒什麼感覺。

<a name="assign-hid-prov"></a>
## PROVIDE_HIDDEN命令

* `PROVIDE_HIDDEN(symbol = expression)` 
  * 和PROVIDE命令相同，差別是這個symbol只在這個處理範圍中才能摸到，一如HIDDEN命令。

<a name="assign-src"></a>
## 談談source code和linker script symbol的關係

這節很有趣，解答我的一些小問題。

* 變數如何存放在binary中？
  * 先把變數名稱放入symbol table內，換句話說symbol table會多一筆資料。這筆資料的欄位有
    * symbol的位址
    * symbol的flags
    * symbol屬於哪個SECTION
    * symbol佔的記憶體空間或是alignment規範
    * symbol的名稱
  * 典型的symbol table 資料：C語言的main()
        *  `00000000004005ed g     F .text	0000000000000101              main`
  * 而symbol的flag有7個groups
    * Group 1: 
      * `l`: local
      * `g`: global
      * `u`: unique global，GNU 用於ELF時的 symbol binding extenstion
      * `!`: 既是global也是local
    * Group 2:
      * `w`: weak symbol
      * `<空白>`: strong symbol
    * Group 3:
      * `C`: symbol 是一個constructor (不知道這邊constructor是指那個東西? )
      * `<空白>`: 一般 symbol
    * Group 4:
      * `W`: warning symbol (不知道是三小)
      * `<空白>`: 一般 symbol
    * Group 5:
      * `I`: 間接地reference其他的symbol
      * `i`: relocate 時要處理的function
      * `<空白>`: 一般 symbol
    * Group 6:
      * `D`: dynamic symbol (不知道是三小)
      * `d`: debug symbol
      * `<空白>`: 一般 symbol
    * Group 7:
      * `F`: 這是一個function
      * `f`: 這是一個檔案
      * `O`: 這是一個object
      * `<空白>`: 一般 symbol
  * 如果有初始值，順便設定初始值。
* 程式語言的取值`foo = 100` runtime發生什麼事？
  * 先去symbol table找foo存在記憶體的位址，把那個位址依照symbol table的size規則將100寫入該位址。
* 程式語言的取值`ptr = &foo` runtime發生什麼事？
  * 先去symbol table找foo存在記憶體的位址，把那個位址寫到ptr在symbol table對應的記憶體。
* symbol在symbol table中存放第一個欄位是**symbol的值**，而這個值**是一個位址**。
* 在linker script設定的symbol如`foo = 100`和在程式碼中轉出的symbol如`foo = 100`差別在那？
  * linker script的100代表的是symbol的位址，而程式碼中轉出的symbol的100代表的是foo對應記憶體存放的值。
* 如何從C語言程式碼中摸到linker script內定義的symbol?
  * 要先知道symbol名稱
  * 在程式碼中extern該symbol，型態為char。
  * 要知道symbol只是一個位址，所以要存取就要取位址。
  * 可以參考以前測試的[範例：Linux中使用C語言載入data object 檔案資料](http://wen00072-blog.logdown.com/posts/194317-loads-the-data-object-using-the-c-language-archives-data-in-linux)
  * 另外有趣的是似乎section的symbol對應的位址不一定是只當位址使用。詳細測試在[這邊：Linux中使用C語言載入data object 檔案資料 (續）](http://wen00072-blog.logdown.com/posts/194370-load-linux-using-c-language-data-object-archives-data-continued)
* 我可以反方向從linker script摸程式碼的symbol
  * 不一定，不同的程式語言和編譯器有不同的變數和函數命名方式，也就是說你原始程式碼的symbol名稱**可能不是**最後存在輸出object檔案的symbol 名稱。


[下一篇](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command)
[回總目錄](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-0-table-of-contents)
