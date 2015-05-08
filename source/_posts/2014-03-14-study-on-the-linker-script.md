---
layout: post
title: 'Linker script初探 - GNU linker ld手冊略讀'
date: 2014-03-14 09:01
comments: true
categories: [linker script, Linux]
---
關於一個程式的binary要怎麼存放其實是很有趣的問題，我以前都沒有去想這個問題。後來當組裝工久了以後就忍不住會想知道這些。隨便想一下就有很多問題，例如：

* 程式碼和資料要怎麼放？
* 怎麼做到不同的source code共用global 變數？
* global 變數和local變數放的地方應該不一樣吧？那麼確實不一樣的點是？
* 呼叫副函數這回事一定是要先找到副函數再跳過去吧？那麼「找到」到底是什麼意思？
* 如果是用shared library的話，runtime才會找到副函數所在的地方，那麼為什麼編譯的時候不會有錯誤呢？
...

這些問題列出來真的是「罄竹難書」，不過我想整體來說至少在Linux下面從binutils下手應該是沒錯。第一個問題應該和linker有關係。所以我先去看ld文件中的linker script，希望可以解決我的疑惑。就算和我的問題無關，至少可以留下一些中文參考資料，造福需要的朋友。

## 目錄

* [簡介](#intro)
* [背景知識](#bkg)
	* [Section](#bkg-sec)
		* [Section 記憶體位址](#bkg-layout)
	* [Symbol](#bkg-sym)
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
* [SECTIONS命令](#sec)
	* [輸出object檔案的section描述](#sec-output-desc)
		* [輸出object檔案的section 命名](#sec-output-name)
		* [輸出object檔案section 命令: address欄位](#sec-output-addr)
		* [輸入object檔案的section描述](#sec-input-desc)
		* [輸入object檔案的section 基礎概念](#sec-input-desc-basic)
* [待釐清項目](#todo)
* [參考資料](#ref)

---
<a name="intro"></a>
## 簡介
`ld`是GNU linker的程式。`ld`吃多個object (*.o)檔或archive (*.a)檔，將他們的資料relocate還有symbol reference資訊一併輸出到新的binary。link通常是compile產生binary的最後步驟。`ld`在執行的時候依照**Linker command language**檔案描述去產生binary。ld支援不同的binary format [(BFD: Binary File Descriptor)](https://sourceware.org/binutils/docs/ld/BFD.html#BFD)

每次link的時候，都會依照特定的命令去產生新的object檔。而這些命令就是linker script；換句話說，linker script提供一連串的命令讓linker照表操課。Linker script描述的命令有

* ld吃的object檔案中的section要怎麼map到要輸出的binary檔案。
* 要輸出的binary檔案要在記憶體中的**layout**
* 其他

因為每次link一定會依據linker script去link，所以當`ld`沒有指定linker script的時候，系統會使用預設的linker script。而`ld --verbose`可以顯示預設的linkder script。link時指定自幹的linker script則使用`ld -T 自己的linker script`。

---
<a name="bkg"></a>
## 背景知識

* object 檔格式：輸入檔案和輸出檔案所遵循的格式
* object 檔案：輸入檔案和輸出檔案
* executable：ld輸出的檔案，有時候會這樣稱呼
* 每個object檔案都有好幾個section，而
  * input section：輸入object檔案中的section
  * output section：輸出object檔案中的section

<a name="bkg-sec"></a>
### Section

* obj檔案內部有一組section
* section包含
	* 自己的名稱
  * section contents
  * section長度資訊
  * 狀態
		* loadable: 執行時該section是否需要被載入到記憶體
 		* allocatable: 先保留記憶體的一塊空間讓程式執行時使用，如.bss
    * section不是loadable 或allocatable 的話一般來說都是給debug用的
* [參考示意圖: (Jim Huang) How GNU Toolchain Works投影片 ](http://www.slideshare.net/jserv/from-source-to-binary-how-gnu-toolchain-works/46)

<a name="bkg-layout"></a>
#### Section 記憶體位址

* Output section如果被載入記憶體，會存放兩種記憶體位址
	* VMA: Virtual Memory Address
  	* Runtime 的記憶體位址
  * LMA: Load Memory Address
  	* load time的記憶體位址
* 一般來說，VMA = LMA。不同情況有東西要燒到ROM時參考LMA。從ROM載入到記憶體執行的時候參考VMA
* 可以使用objdump -h看VMA/LMA資訊
	
<a name="bkg-sym"></a>
### Symbol

* 一個object 檔案存放多個symbol，又稱為symbol table
* 將名稱對應到一個記憶體位址的symbol稱為defined symbol，名稱沒有對應到記憶體位址的稱為undefined symbol
* 名稱通常就是全域變數、靜態變數或是函數的名稱
* 一般來說，如果把單獨的c編譯成object file時
	* defined symbol為該檔案內的global variable, static varible 和funciton
  * undefined symbol為該檔案內的extern variable和外部funciton
* 可以使用objdump -t或是nm看symbol資訊

---
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

---
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
* 接下來請把所有**輸入**object檔案的程式機械碼中(`{ *(.text) }`)存放到**輸出**object檔案的`.text`區塊中。
* 接設定記憶體位置counter為0x8000000
* 先放有初始值的全域變數（.data）
* 再放**沒有**初始值的全域變數（.bss）

另外要注意的是ld會自動幫你處理[alignment的問題](http://en.wikipedia.org/wiki/Data_structure_alignment)，所以不用擔心section之間的aligment問題。

---
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
### 檔案相關命令

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
### Object檔案相關格式命令

* `OUTPUT_FORMAT(bfdname)`
  * 指定輸出object檔案的binary 檔案格式，可以使用`objdump -i`列出支援的binary 檔案格式
* `OUTPUT_FORMAT(default, big, little)`
  * 指定輸出object檔案預設的binary 檔案格式，big endian的binary 檔案格式以及little endian的binary 檔案格式。可以使用`objdump -i`列出支援的binary 檔案格式
* `TARGET(bfdname)`
  * 告訴ld用那種binary 檔案格式讀取輸入object檔案要，可以使用`objdump -i`列出支援的binary 檔案格式
 
<a name="cmd-alias"></a> 
### 設定記憶體區塊alias命令

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
### 未分類的命令 (節錄）

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
---
## 設定symbol的值
linker script提供設定**symbol**數值的方法。要注意的是，這邊的symbol可以指一個全域變數、`SECTION`命令中的location counter（就是`.`開頭的資料如`.text`）

使用方式介紹如下：

<a name="assign-op"></a>
### 基本運算

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
### HIDDEN命令

* `HIDDEN(要隱藏的symbol)`
可以把他理解成加了`static`的全域變數，也就是說這個symbol只在這個處理範圍中才能摸到。

<a name="assign-prov"></a>
### PROVIDE命令

* `PROVIDE命令(symbol = expression)`
  * 簡單來說，如果你的程式已經有這個symbol（函數或變數），就用你的；否則就使用這邊提供的symbol。手冊上說是給特殊的linker使用的。想知道他提的use case可以看[這邊](https://sourceware.org/binutils/docs/ld/PROVIDE.html#PROVIDE)，我是沒什麼感覺。

<a name="assign-hid-prov"></a>
### PROVIDE_HIDDEN命令

* `PROVIDE_HIDDEN(symbol = expression)` 
  * 和PROVIDE命令相同，差別是這個symbol只在這個處理範圍中才能摸到，一如HIDDEN命令。

<a name="assign-src"></a>
### 談談source code和linker script symbol的關係
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


---
<a name="sec"></a>
## SETCION命令
其實一開始是為了看懂這個命令才會想看linker script的。如果接觸過很小型的Embedded OS就會發現很多都是自幹linker script；而這些scripts主要的描述命令就是`SETCION`。

好了，廢話少說，進入主題。SECTION命令的功用是

* 告訴linker怎麼把輸入object檔案中的SECTION對應到輸出object檔案中的SECTION
* 告訴loader object檔案中的SECTION要放到記憶體那些地方

典型的SECTION命令長這樣子：

```c
SECTIONS
{
	sections-command
	sections-command
	...
}
```

望文生義地猜測可以這樣理解：
輸出object有一些大方向的規範，並且分為不同的section，每個section有他自己的規範。

而`sections-command`可以分為下面幾種功能

* [ENTRY命令](#cmd)
* [設定symbol的值](#assign)
* [描述輸出object檔案的setcion](#sec-output-desc)
* Overlay描述 (不知道是三小)

要注意的事，如果你自幹的linker script沒有描述輸出object檔案的setcion的話，linker會

* 讀輸入object檔案section時，如果該section第一次出現，就在輸出object檔案中加入同樣名稱的section，直到處理完所有的輸入object檔案
* 第一個吃到的輸入object檔案section將當作位址0的起始點

<a name="sec-output-desc"></a>
### 輸出object檔案的section描述

```c 輸出object檔案的section描述格式
section [address] [(type)] :
	[AT(lma)]
	[ALIGN(section_align) | ALIGN_WITH_INPUT]
	[SUBALIGN(subsection_align)]
	[constraint]
	{
		output-section-command
		output-section-command
		...
	} [>region] [AT>lma_region] [:phdr :phdr ...] [=fillexp]
```

其中`output-section-command`的功能有

* [設定symbol的值](#assign)
* 描述輸入object檔案中的section要怎麼放到輸出object檔案的setcion
* 輸出object檔案的setcion的資料存放格式如alignment等
* 其他

這邊很多術語需要先搞清楚，先列出來，希望之後可以看到解答

* type
* region
* AT(lma)
* lma_region
* =fillexp

<a name="sec-output-name"></a>
### 輸出object檔案的section 命名

* 必須符合你要輸出object檔案binary format規定。

<a name="sec-output-addr"></a>
#### 輸出object檔案section 命令: address欄位
address是[section](#sec-output-desc)的一個optional欄位，使用的記憶體空間為[VMA](#bkg-layout)。如果沒有指定的話，linker會依下面的方式設定輸出object檔案section 的VMA。該VMA會遵循section 的alignment規範。

* 有設定`region`的話就從region內剩餘空間開始位址
* 有使用`MEMORY`命令定義硬體記憶區塊的話，從定義的區塊中挑**第一個**符合SECTION的區塊。再將address設成該區塊內剩餘空間開始位址
* 以上皆非的情況下，位址設成locale counter

address欄位因為可以使用exression所以可能有下面的陷阱

* `.text . : { *(.text) }`
* `.text : { *(.text) }`
這兩個差一個`.`，意義就差很多。沒有`.`那個，表示沒有設定address，所以就是設成locale counter，並且linker會保證alignment。而有`.`的就表示hardcode成locale counter，所以有可能會有alignment的問題。

另外一點要注意的設定後locale counter也會跟著改變。

<a name="sec-input-desc"></a>
#### 輸入object檔案的section描述
這部份可以說是整個`output-section-command`的重點，目的是告訴linker讀取輸入object檔案後，怎麼把這些檔案裏面的section複製到輸出object檔案裏面**適當地**section。

<a name="sec-input-desc-basic"></a>
#### 輸入object檔案的section 基礎概念
格式為`檔案(section1  section2 ...)`，檔案支援[萬用字元](https://sourceware.org/binutils/docs/ld/Input-Section-Wildcards.html#Input-Section-Wildcards)。

所以常看到的`*(.text)`的意思是：所有輸入object檔案裏面的`.text` section。

指定多個section的方式有兩種

* `*(.sec1 .sec2)`：如果輸入object有兩個檔案的話，輸出object檔案裏面section會變成
![ld1_section.png](http://user-image.logdown.io/user/3993/blog/4041/post/188334/f6Crrs5MTT2tl66gz08O_ld1_section.png)

* `*(.sec1) *(.sec2)`: 如果輸入object有兩個檔案的話，輸出object檔案裏面section會變成
![ld2_section2.png](http://user-image.logdown.io/user/3993/blog/4041/post/188334/LqhscLhZSbeN8XckduAG_ld2_section2.png)

--- 
<a name="todo"></a>

* 待釐清項目
  * dynamic symbol (不知道是三小)
  * warning symbol (不知道是三小)
  * constructor symbol (不知道是三小)
	* Overlay描述 (不知道是三小)
* 概念
  * 名詞解釋
    * bss
    * text
    * data
  * locale counter
  * region 
  
  
---
<a name="ref"></a>
## 參考資料
* [GNU linker ld: Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)
* [from Source to Binary: How GNU Toolchain Works](http://www.slideshare.net/jserv/from-source-to-binary-how-gnu-toolchain-works)
* [Linker Scripts - OSDev Wiki](http://wiki.osdev.org/Linker_Scripts)
* [Embedded Programming with the GNU Toolchain](http://www.bravegnu.org/gnu-eprog/index.html)
* [Stackoverflow: What does each column of objdump's Symbol table mean?](http://stackoverflow.com/questions/6666805/what-does-each-column-of-objdumps-symbol-table-mean)
