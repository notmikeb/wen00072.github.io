---
layout: post
title: 'objdump -t 的欄位說明'
date: 2014-12-09 23:24
comments: true
categories: [binutils, Linux]
---
`objdump -t` 可以列出object檔案的symbol table內容，這是一個我們很熟練的C 語言 main()在symbol table存放範例。

000000000040052d g     F .text	000000000000002c              main

不知道是三小朋友對不對？我也不知道，問了男人後回答如下：

* 這筆資料的欄位有
	* symbol的對應的值，猜測是位址或是offset
	* symbol的flags
	* symbol屬於哪個SECTION
	* symbol佔的記憶體空間或是alignment規範
	* symbol的名稱
* 而symbol的flag有7個groups
	* Group 1: 
		* `l`: local
		* `g`: global
		* `u`: unique global，GNU 用於ELF時的 symbol binding extenstion (不知道是三小)
		* `!`: 既是global也是local
	* Group 2:
		* `w`: weak symbol
		* `<空白>`: strong symbol
	* Group 3:
		* `C`: symbol 是一個constructor (不知道是三小)
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
    
## 參考資料
* `man objdump`
