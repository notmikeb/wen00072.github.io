---
layout: post
title: 'GNU LD 手冊略讀  (0): 目錄和簡介'
date: 2014-12-14 23:45
comments: true
categories: 
---
關於一個程式的binary要怎麼存放其實是很有趣的問題，我以前都沒有去想這個問題。後來當組裝工久了以後就忍不住會想知道這些。隨便想一下就有很多問題，例如：

* 程式碼和資料要怎麼放？
* 怎麼做到不同的source code共用global 變數？
* global 變數和local變數放的地方應該不一樣吧？那麼確實不一樣的點是？
* 呼叫副函數這回事一定是要先找到副函數再跳過去吧？那麼「找到」到底是什麼意思？
* 如果是用shared library的話，runtime才會找到副函數所在的地方，那麼為什麼編譯的時候不會有錯誤呢？
...

這些問題列出來真的是「罄竹難書」，不過我想整體來說至少在Linux下面從binutils下手應該是沒錯。第一個問題應該和linker有關係。所以我先去看[GNU ld手冊的linker script部份](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)，希望可以解決我的疑惑。就算和我的問題無關，至少可以留下一些中文參考資料，造福需要的朋友。

為了讓單篇的篇幅不要太過冗長，我把內容切割成幾個部份。這邊就先放全部的目錄和簡介的部份。

## 目錄
## [第一部份](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-1/)

* [簡介](#intro)
* [背景知識](#bkg)
	* [Section](#bkg-sec)
	* [Section 記憶體位址](#bkg-layout)
	* [Symbol](#bkg-sym)
* [Linker script 格式概論](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-1/#fmt)
* [從Linker script 範例開始](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-1/#ex)
* [簡易script 命令格式](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-1/#cmd)
  * [檔案相關命令](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-1/#cmd-file)
  * [Object檔案相關格式命令](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-1/#cmd-obj)
  * [設定記憶體區塊alias命令](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-1/#cmd-alias)
  * [未分類的命令 (節錄）](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-1/#cmd-misc)
* [設定symbol的值](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-1/#assign)
  * [基本運算](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-1/#assign-op)
  * [HIDDEN命令](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-1/#assign-hid)
  * [PROVIDE命令](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-1/#assign-prov)
  * [PROVIDE_HIDDEN命令](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-1/#assign-hid-prov)
  * [談談source code和linker script symbol的關係](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-1/#assign-src)


## [第二部份](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command)

* [SECTIONS命令](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec)
* [輸出object檔案的section描述](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-output-desc)
	* [輸入object檔案的section 基礎概念](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-input-desc-basic)
	* [輸入object檔案的section 語法的萬用字元](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-input-desc-wildcard)
	* [輸入object檔案的COMMOM section](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-input-desc-comm)
	* [KEEP指令](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-input-desc-keep)
	* [輸入object檔案放到輸出object檔案範例](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-input-desc-ex)
* [輸出object檔案section 命名](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-output-name)
* [輸出object檔案section 命令: address欄位](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-output-addr)
* [輸入object檔案的section描述](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-input-desc)
* [輸出object檔案內指定固定資料長度](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-output-data)
* [輸出object檔案捨棄的section](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-output-discard)
* [輸出object檔案section其他屬性](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-output-attr)
	* [輸出object檔案 Section Type](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-output-attr-type)
	* [輸出object檔案 Section LMA](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-output-attr-lma)
	* [強制輸出object檔案的 Alignment](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-output-attr-output-align)
	* [強制輸入object檔案的 Alignment](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-output-attr-input-align)
	* [輸出object檔案 Section 限制](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-output-attr-limit)
	* [輸出object檔案 Section Region](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-output-attr-region)
	* [輸出object檔案 Section Phdr](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-output-attr-output-phdr)
	* [指定輸出object檔案 Section 填空的資料](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-output-attr-output-fill)
* [OVERLAY命令](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command/#sec-overlay)


## [第三部份](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/)

* [MEMORY命令](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#mem)
* [PHDRS命令](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#phdr)
* [VERSION命令](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#ver)
* [Linker script 中使用的expression](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#expr)
	* [常數](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#expr-const)
	* [Symbolic 常數](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#expr-sym-const)
	* [Symbol命名規則](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#expr-sym)
	* [孤兒 Section](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#expr-oph-sec)
	* [Location Counter](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#expr-lcnt)
	* [Operators](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#expr-op)
	* [計算結果](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#expr-eval)
	* [Expression 計算結果和absolute/relative address的關係](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#expr-sec)
	* [內建函數](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#expr-btfun)
* [Implicit Linker Scripts](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#imp)
* [待釐清項目](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#todo)
* [參考資料](http://wen00072.github.io/blog/2014/12/15/study-on-the-linker-script-3/#ref)



<a name="intro"></a>
## 簡介
`ld`是GNU linker的程式。`ld`吃多個object (*.o)檔或archive (*.a)檔，將他們的資料relocate還有symbol reference資訊一併輸出到新的binary。link通常是compile產生binary的最後步驟。`ld`在執行的時候依照**Linker command language**檔案描述去產生binary。ld支援不同的binary format [(BFD: Binary File Descriptor)](https://sourceware.org/binutils/docs/ld/BFD.html#BFD)

每次link的時候，都會依照特定的命令去產生新的object檔。而這些命令就是linker script；換句話說，linker script提供一連串的命令讓linker照表操課。Linker script描述的命令有

* ld吃的object檔案中的section要怎麼map到要輸出的binary檔案。
* 要輸出的binary檔案要在記憶體中的**layout**
* 其他

因為每次link一定會依據linker script去link，所以當`ld`沒有指定linker script的時候，系統會使用預設的linker script。而`ld --verbose`可以顯示預設的linkder script。link時指定自幹的linker script則使用`ld -T 自己的linker script`。


<a name="bkg"></a>
## 背景知識

* object 檔格式：輸入檔案和輸出檔案所遵循的格式
* object 檔案：linker處理時讀入的輸入檔案和將結果存放的輸出檔案
* executable：ld輸出的檔案，有時候會這樣稱呼
* 每個object檔案都有好幾個section，而
  * input section：輸入object檔案中的section
  * output section：輸出object檔案中的section
* bss
  * 存放沒有初始值的全域變數的地方 ex: `int g_var;`
* text
  * 存放編譯過的執行機械碼的地方
* data
  * 存放沒有初始值全域變數的地方 ex: `int g_var = 0xdeadbeef;`
* locale counter
  * 代表目前輸出object檔案位置的最後端
* region 
  * 執行平台實體的記憶體區塊。如0x1000~0x1999是ROM, 0x5000~0x9999是RAM。那麼這個平台就可以設定成有兩個region。
  

<a name="bkg-sec"></a>
## Section

* obj檔案內部有一組section
* section包含
  * 自己的名稱
  * section contents
  * section長度資訊
  * 狀態
    * loadable: 執行時該section是否需要被載入到記憶體
    * allocatable: 如果section本身沒資料（如.bss）可以設成這個狀態，讓loader先保留記憶體的一塊空間
    * section不是loadable 或allocatable 的話一般來說都是給debug用的
    * `objdump -h`顯示的狀態([出處](http://stackoverflow.com/questions/11196048/flags-in-objdump-output-of-object-file))，不要問我為何和手冊不一樣，因為我也不知道。
    * `LOAD`
      * 表示這個section需要從檔案載入到記憶體
    * `DATA`
      * 表示這個section存放資料，不可以被執行
    * `READONLY`
      * 可以望文生義吧？
    * `ALLOC`
      * 表示該section會吃記憶體，你可能會想說廢話，section不放記憶體放檔案是放心酸的嘛？還真的有，例如放除錯的section。
    * `CONTENTS`
      * 表示這個section是執行程式所需要的資訊，如程式碼或是資料。
* [參考示意圖: (Jim Huang) How GNU Toolchain Works投影片 ](http://www.slideshare.net/jserv/from-source-to-binary-how-gnu-toolchain-works/46)

<a name="bkg-layout"></a>
### Section 記憶體位址

* Output section如果被載入記憶體，會存放兩種記憶體位址
	* VMA: Virtual Memory Address
  	* Runtime 的記憶體位址
  * LMA: Load Memory Address
  	* load time的記憶體位址
* 一般來說，VMA = LMA。不同情況有東西要燒到ROM時參考LMA。從ROM載入到記憶體執行的時候參考VMA
* 可以使用objdump -h看VMA/LMA資訊
	
<a name="bkg-sym"></a>
## Symbol

* 一個object 檔案存放多個symbol，又稱為symbol table
* 將名稱對應到一個記憶體位址的symbol稱為defined symbol，名稱沒有對應到記憶體位址的稱為undefined symbol
* 名稱通常就是全域變數、靜態變數或是函數的名稱
* 一般來說，如果把單獨的c編譯成object file時
	* defined symbol為該檔案內的global variable, static varible 和funciton
  * undefined symbol為該檔案內的extern variable和外部funciton
* 可以使用objdump -t或是nm看symbol資訊
