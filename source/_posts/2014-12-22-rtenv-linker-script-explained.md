---
layout: post
title: 'rtenv的linker script解釋'
date: 2014-12-22 19:18
comments: true
categories: [C, rtenv, Linux, ARM]
---
[rtenv](https://github.com/southernbear/rtenv.git)是成功大學資訊工程系同學寫出來給CM3的小型作業系統，之前使用rtenv寫作業的時候曾經trace變數trace到C code裏面沒有，但是卻在linker script找到。可是那時候看的感覺就是一堆符號，所以就沒繼續追下去。這也是我想要了解linker script的起點。看完liner script語法後，自然要回來看一下是否可以了解他的描述，先看完整語法。

```c main.ld
ENTRY(main)
MEMORY
{
  FLASH (rx) : ORIGIN = 0x00000000, LENGTH = 128K
  RAM (rwx) : ORIGIN = 0x20000000, LENGTH = 20K
}

SECTIONS
{
    .text :
    {
        KEEP(*(.isr_vector))
         *(.text)
         *(.text.*)
        *(.rodata)
        *(.rodata*)
        _smodule = .;
        *(.module)
        _emodule = .;
        _sprogram = .;
        *(.program)
        _eprogram = .;
        _sromdev = .;
        *(.rom.*)
        _eromdev = .;
        _sidata = .;
    } >FLASH

    /* Initialized data will initially be loaded in FLASH at the end of the .text section. */
    .data : AT (_sidata)
    {
        _sdata = .;
        *(.data)        /* Initialized data */
        *(.data*)
        _edata = .;
    } >RAM

    .bss : {
        _sbss = .;
        *(.bss)         /* Zero-filled run time allocate data memory */
        _ebss = .;
    } >RAM

    _estack = ORIGIN(RAM) + LENGTH(RAM);
}
```

很長令人害怕嗎？先從大方向拆解

* 程式起始點是`main`
* 使用`MEMORY`指令設了兩個region，分別為`FLASH`和`RAM`
* 輸出object檔案有三個section，分別是`.text`, `.data` , `.bss`

然後我們再往下看`MEMORY`和`SETCIONS`命令的描述：

## `MEMORY`

```c
MEMORY
{
  FLASH (rx) : ORIGIN = 0x00000000, LENGTH = 128K
  RAM (rwx) : ORIGIN = 0x20000000, LENGTH = 20K
}
```

可以看到我們有兩個region

* `FLASH`
	* 唯讀、可執行
  * 起始位址0x00000000
  * 長度128k
* `RAM`
	* 可讀寫和執行
  * 起始位址0x20000000
  * 長度20k
  
口說無憑，可以看一下CM3的[Memory map](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dai0179b/CHDDBAEC.html)，確認一下0x00000000是不是寫程式的區段，而0x20000000是不是RAM的區段（好吧是SRAM）

## `SECTION`
剛才講過有三個section，我們一個一個分別討論：

## `.text`

```c
.text :
{
  KEEP(*(.isr_vector))
  *(.text)
  *(.text.*)
  *(.rodata)
  *(.rodata*)
  _smodule = .;
  *(.module)
  _emodule = .;
  _sprogram = .;
  *(.program)
  _eprogram = .;
  _sromdev = .;
  *(.rom.*)
  _eromdev = .;
  _sidata = .;
} >FLASH
```

* 這個section要放在FLASH的region
* 所有輸入object檔案的8個section會放入這個輸出object檔案section，分別為
	* `.isr_vector`，這個section不可以被garbage collect回收
  * `.text`
  * 所有.text.開頭的section
  * `.rodata`
	* 所有.rodata開頭的section
  * `.module`
  * `.program`
  * 所有.rom.開頭的section
* 這個section有7個symbol，分別是
	* `_smodule`
		* `.module`起始位址
  * `_emodule`
		* `.module`結束位址
  * `_sprogram`
		* `.program`起始位址
  * `_eprogram`
		* `.program`結束位址
	* `_sromdev`
		* 所有.rom.開頭的section集合的起始位址
  * `_eromdev`
		* 所有.rom.開頭的section集合的結束位址
  * `_sidata`
		* `.data` section起始位址
    
這些symbol是有意義的，你需要查詢程式原始碼看他們在做三小。相信我，這值得一看。

## `.data`

```c
/* Initialized data will initially be loaded in FLASH at the end of the .text section. */
.data : AT (_sidata)
{
  _sdata = .;
  *(.data)        /* Initialized data */
  *(.data*)
  _edata = .;
} >RAM
```

這部份表示

* `.data`的LMA (載入記憶體位址）是`_sidata`，就是`.text`結束的地方。另外這邊你要自己搬，有興趣請查原始碼。
* `.data`要放在RAM的region
* 所有輸入object檔案的2個section會放入這個輸出object檔案section，分別為
	* `.data`
  * 所有.data開頭的section
* 這個section有2個symbol，分別是
	* `_sdata`
		* `.data`的起始位址
	* `_edata`
		* `.data`的結束位址
    
這些symbol是有意義的，你需要查詢程式原始碼看他們在做三小。相信我，這值得一看。

## `.bss`

```c
.bss : {
	_sbss = .;
	*(.bss)         /* Zero-filled run time allocate data memory */
	_ebss = .;
} >RAM
```

* 所有輸入object檔案的`.bss`會放入這個輸出object檔案section
* `.bss`要放在RAM的region
* 這個section有2個symbol，分別是
	* `_sbss`
		* `.bss`的起始位址
	* `_ebss`
		* `.bss`的結束位址
    
這些symbol是有意義的，你需要查詢程式原始碼看他們在做三小。相信我，這值得一看。

## stack

```c
    _estack = ORIGIN(RAM) + LENGTH(RAM);
```

有印象程式使用的stack是由記憶體最後面往前面長的嘛？沒印象？那就估狗`linux`, `stack`, `text`的圖片就可以看到了。這邊也是同樣的概念，所以他的`_estack` symbol位址會是RAM的開頭位址加上RAM的size。

這些symbol是有意義的，你需要查詢程式原始碼看他們在做三小。相信我，這值得一看。
