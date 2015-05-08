---
layout: post
title: 'GNU LD 手冊略讀 (3): Chapter 3.7 ~ Chapter 3.11'
date: 2014-12-15 00:06
comments: true
categories: [binutils, linker script]
---
[上一篇](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-2-setcion-command)
[回總目錄](http://wen00072.github.io/blog/2014/12/14/study-on-the-linker-script-0-table-of-contents)

## 本篇目錄

* [MEMORY命令](#mem)
* [PHDRS命令](#phdr)
* [VERSION命令](#ver)
* [Linker script 中使用的expression](#expr)
	* [常數](#expr-const)
	* [Symbolic 常數](#expr-sym-const)
	* [Symbol命名規則](#expr-sym)
	* [孤兒 Section](#expr-oph-sec)
	* [Location Counter](#expr-lcnt)
	* [Operators](#expr-op)
	* [計算結果](#expr-eval)
	* [Expression 計算結果和absolute/relative address的關係](#expr-sec)
	* [內建函數](#expr-btfun)
* [Implicit Linker Scripts](#imp)
* [待釐清項目](#todo)
* [參考資料](#ref)


<a name="mem"></a>
## MEMORY命令
預設的linker會將所有的memory space視為可以分配的。然而現實生活這個假設不一定成立，例如你寫資料到ROM的記憶體就保證GG。所以linker script提供了`MEMORY`命令讓你畫地盤，告訴linker那塊地盤有什麼樣的特性。該命令會描述

* 你給這塊記憶體取的名稱，也就是說前面一直講的region
* 起始位址
* 上面位址後面的記憶體大小
* 這塊記憶體有什麼限制

命令語法如下：

```c
MEMORY
{
	name [(attr)] : ORIGIN = origin, LENGTH = len
	...
}
```

每個欄位說明如下

* `name`
	* 你給這塊記憶體取的名稱，也就是說前面一直講的region(以下以region稱呼)。這個名稱不可以和同個linker script中以下的名稱相同：
		* symbol名稱
		* section名稱
		* 檔案名稱
  * 每塊region都要給個名字，這些名字可以給他取alias，這部份請參考[`REGION_ALIAS`命令](http://wen00072-blog.logdown.com/posts/246069-study-on-the-linker-script-1#cmd-alias)
* `attr`
	* optional
  * 告訴linker這塊記憶體有什麼值得注意的地方，一個region可以有多個屬性，列出如下
		* `R`: Read only
		* `W`: 可讀寫
		* `X`: executable
		* `A`: 可allocate
		* `I`和`L`: Initialized section (三小？)
		* `!`: 將該符號後面所有的屬性inverse
	* 如果一個section符合上面的條件，就可以放在這個region中
* `ORIGIN`
	* 一個expression，表示該region的起始位址
* `LENGTH`
	* region 大小，單位為byte

下面的範例可以看到

* 有兩個region
* region `rom`的資訊：
	* 唯讀、可執行
	* 起始位址為`0`
	* 長度為256k
* region `ram`的資訊：
	* 非唯讀、不可執行
	* 起始位址為`0x40000000`
  * 長度為4M
	* 使用了縮寫，縮寫規則不想翻，請自己[看這邊](https://sourceware.org/binutils/docs/ld/MEMORY.html#MEMORY)
  
```c
MEMORY
{
	rom (rx)  : ORIGIN = 0, LENGTH = 256K
	ram (!rx) : org = 0x40000000, l = 4M
}
```

region和section的合體部份[前面](http://wen00072-blog.logdown.com/posts/246070-study-on-the-linker-script-2-setcion-command#sec-output-attr-region)有提過了。如果沒有指定region的話，linker會從目前region挑一個給你用。除此之外，你的section空間region塞不下的話linker會幫你偵測出來。

另外ORIGIN和LENGTH可以當作查詢region的資訊，範例如下

```c
_fstack = ORIGIN(ram) + LENGTH(ram) - 4;
```


<a name="phdr"></a>
## PHDRS命令
資源回收上一篇講的東西。**基本上我不知道elf是三小，所以很有可能這部份有錯誤，請自行斟酌！**

PHDR 是[ELF](http://en.wikipedia.org/wiki/Executable_and_Linkable_Format)的program header縮寫，又稱為segment（以後以segment稱之）。當ELF loader載入ELF執行檔的時候，會看這些segment決定要如何把讀入的檔案放在記憶體中，這部份和[ABI](http://en.wikipedia.org/wiki/Application_binary_interface)有關係，按下不表，等我那天心情好再來看ELF和ABI。你可以透過`objdump -p`觀察program header。

一般來說，linker預設都幫你弄好elf相關的segment。但是如果你因故需要自幹的話，就可以用`PHDRS`命令，一旦使用了這個命令，linker預設的相關segment設定將被取消。另外這個命令只對elf格式輸出有意義，非elf格式輸出這部份的指令一律失效。

`PHDRS`命令格式如下：

```c
PHDRS
{
	name type [ FILEHDR ] [ PHDRS ] [ AT ( address ) ]
			[ FLAGS ( flags ) ] ;
}
```

* `name`
	* 配合section命令使用，語法可以看[這邊](http://wen00072-blog.logdown.com/posts/246070-study-on-the-linker-script-2-setcion-command#sec-output-attr-output-phdr)
	* segment名稱因為存放在另外的name space，所以不用擔心和symbol, 檔案, section衝突。
* `type`
	* 規範為
		* `PT_NULL` (對應值： 0)
			* 沒使用的segment
		* `PT_LOAD` (對應值： 1)
			* 該segment應從檔案中載入
		* `PT_DYNAMIC` (對應值： 2)
			* 存放dynamic link的資訊
		* `PT_INTERP` (對應值： 3)
			* 指定program interpretor 路徑
			* `readelf -l ls`可以看到該INTERP segment的資料是是`/lib64/ld-linux-x86-64.so.2`，這邊似乎有些好玩的線索，一樣等到想起來再來看看。
		* `PT_NOTE` (對應值： 4)
			* `man elf`說這個是存放輔助資料
		* `PT_SHLIB` (對應值： 5)
			* 保留未使用
		* `PT_PHDR` (對應值： 6)
			* program header存放的segment
		* expression
			* 除了以上自訂的數字，應該是保留給使用自行使用...吧？
	* 每個`type`後面都可以加上`FILEHDR`或`PHDRS`，其中
		* `FILEHDR`：表示該segment應該內含ELF file header
		* `PHDRS`：表示該segment應該內含ELF program header
* `AT`
	* 指定load 位址。和section的[AT](http://wen00072-blog.logdown.com/posts/246070-study-on-the-linker-script-2-setcion-command#sec-output-attr-lma)相同
* `FLAGS(數字)`
	* 數字是ELF的`p_flags`。`man elf`可以查到`p_flags`定義，數值我猜要去看程式碼或是ELF規格了。
		* `PF_X`: executable segment
		* `PF_W`: write segment
		* `PF_R`: read segment

單個segment通常map到一個section，linker依照順序處理header給之後的loader使用。另外要注意的是如果你在某個section指定了`:phdr`後，之後的section就算沒指令，都會放在該segment。如果之後的section有`:phdr`設成`:NONE `的話，linker才不會把之後的section放到任何segment。
  
  
如果有需要，你可以指定不同的segment都要有某個section的內容，使用方式就是在`section`命令中用多個`:phdr`。範例如下：

```c
.interp : { *(.interp) } :text :interp
```

手冊上面提供了一個比較完整的範例。望文生義應該不難理解，所以就不解釋了。

```c
PHDRS
{
	headers PT_PHDR PHDRS ;
	interp PT_INTERP ;
	text PT_LOAD FILEHDR PHDRS ;
	data PT_LOAD ;
	dynamic PT_DYNAMIC ;
}

SECTIONS
{
	. = SIZEOF_HEADERS;
	.interp : { *(.interp) } :text :interp
	.text : { *(.text) } :text
	.rodata : { *(.rodata) } /* defaults to :text */
	...
	. = . + 0x1000; /* move to a new page in memory */
	.data : { *(.data) } :data
	.dynamic : { *(.dynamic) } :data :dynamic
	...
}
```


<a name="ver"></a>
## VERSION命令
ELF檔案格式支援動態link的時候指定shared library版本。這項功能需要linker配合，`VERSION`命令就是來描述版本資訊。

語法如下

```c
VERSION [extern "lang"] { version-script-commands }
```

其中 extern &quot;lang&quot; 的lang有支援

* `C`
* `C++`
* `Java`


至於version-script-commands，手冊上面說和Sun(已被併購)在Solaris 2.5上的linker語法相同，估狗查version-script-commands沒查到語法，只能從手冊提供的範例來看。如果有人知道語法link請跟我說。手冊上面說這是一個樹狀結構，基本單位為一個version node。你可以在version node中設定

* version node名稱
* version node和相依性
* 設定哪些symbol出現在該version node
* 在該version node中指定global symbol變成local。如此一來，這些symbols就不會被shared library以外看到。

從手冊範例可以推測version node格式如下

```c
name {
	[global:]
				symbol1;
        ...
	[local:]
				symbol_a;
  			...
} [depend_name];
```

好啦，有這樣的概念後我們來看手冊範例

```c
VERS_1.1 {
	global:
			foo1;
	local:
			old*;
			original*;
			new*;
};

VERS_1.2 {
	foo2;
} VERS_1.1;

VERS_2.0 {
			bar1; bar2;
	extern "C++" {
					ns::*;
					"f(int, double)";
	};
} VERS_1.2;
```

OK，開始解釋：

* 有三個Version node，名稱為`VERS_1.1`, `VERS_1.2`, `VERS_2.0`
* `VERS_1.1`沒有相依性，`VERS_1.2`相依於`VERS_1.1`, `VERS_2.0`相依於`VERS_1.2`
* `VERS_1.1`中
	* symbol `foo1`和`VERS_1.1`有關
	* old開頭、orignal開頭和new開頭的symbol都不會被外面看到
* `VERS_1.2`中
	* symbol `foo2`和`VERS_1.2`有關
* `VERS_2.0`中
	* `bar1`和`bar2`和`VERS_2.0`有關

看完些描述後，可以問**啊沒有指定和version node相關的symbol怎麼辦？**手冊說會分配給library的base version（好吧我不知道base version是三小。）。如果你要將沒指定version node的symbol全部設成和某個version node有關的話，請在該version node中加上以下的描述：

* `global: *;`

一般來說這個描述加再最後的version node才有意義，否則在前面的version node中把所有的symbol都被設定完畢的話，那接下來的version node就沒有辦法設定symbol關聯性的。

手冊中指出version node名稱是給人看的，對於linker在乎的只有他們的關係。所以你要故意取成讓人看不懂的名稱也可以滴。

如果你要指定所有的版本都使用同樣的symbol設定，那麼寫一份就好。重點是這份描述不用寫version node名稱，範例如下。

```c
{ global: foo; bar; local: *; };
```

至於在程式碼中指定版本的方式，你需要使用GNU extention語法，例如加入下面的~~咒與~~描述到你的程式碼中。
語法如下：

```c
__asm__(.symver name, name2@version_node_name);
```

* `.symver`：你應該用的指令
* `name`：你程式用到的symbol
* `name2@version_node_name`：實際上你真正用的symbol以及對應的version node

範例如下：

```c
__asm__(".symver original_foo,foo@VERS_1.1");
```

你也可以分別指定自己程式的symbol對應到不同版本的symbol，範例如下：

```c
__asm__(".symver original_foo,foo@");
__asm__(".symver old_foo,foo@VERS_1.1");
__asm__(".symver old_foo1,foo@VERS_1.2");
__asm__(".symver new_foo,foo@@VERS_2.0");
```

* `foo@`表示未指定版號的symbol就用該symbol
* `foo@@VERS_2.0`的`@@`表示預設使用該設定


`.symver`詳細的語法說明可以看[這邊](https://sourceware.org/binutils/docs/as/Symver.html)

**下面這段是囈言囈語，因為我在描述一個我不知道什麼、以及不知道我在描述什麼的東西，請當作夢話跳過！**
當你的程式要使用shared library的symbol的時候，你的程式應該要知道要用哪個版本的symbol以及這些symbol是在哪個version node宣告（怎麼做？我寫程式還要管shared library symbol版本，看linker script？不合理）。所以runtime的時候dynamic loader可以幫你搞定resolve symbol的事情。

跳過解釋需要version 的原因，想知道的可以看[原文](https://sourceware.org/binutils/docs/ld/VERSION.html#VERSION)，看懂順便跟我說。

Demangled names的注意事項懶得看，一併跳過。


<a name="expr"></a>
## Linker script 中使用的expression
Linker script 的 expression有幾點特性

* expression和C語言相同
* 型態都是整數
* 變數size相同，target和host為32-bit的話size就是32-bit，否則就為64-bit(為啥？那麼可不可以8, 16-bit?）。
* expression中允許設定和讀取symbol的值

接下來我們來討論linker 中Expression可以使用的內建功能

* [常數](#expr-const)
* [Symbolic 常數](#expr-sym-const)
* [Symbol命名規則](#expr-sym)
* [孤兒 Section](#expr-oph-sec)
* [Location Counter](#expr-lcnt)
* [Operators](#expr-op)
* [計算結果](#expr-eval)
* [Section內的Expression](#expr-sec)
* [內建函數](#expr-btfun)


<a name="expr-const"></a>
## 常數   
設定常數規則如下

* 8進位
	* `0`開頭
	* `o`結尾, `O`結尾
* 16進位
	* `0x`開頭, `0X`開頭：
	* `h`結尾, `H`結尾 
* 10進位
	* `d`結尾, `D`結尾
* 不屬於上面的數字表示為10進位
* `K`
	* 1024
* `M`
	* 1024 * 1024
* `K`和`M`不能跟下面的描述混用
	* `o`結尾, `O`結尾
	* `h`結尾, `H`結尾
	* `d`結尾, `D`結尾

範例：

```c
_fourk_1 = 4K;
_fourk_2 = 4096;
_fourk_3 = 0x1000;
_fourk_4 = 10000o;
```


<a name="expr-sym-const"></a>
## Symbolic 常數
指令：

* `CONSTANT(name)`
	* 合法的`name`如下，可以望文生義所以就不解釋
		* `MAXPAGESIZE`
		* `COMMONPAGESIZE`
    
範例：指定`.text` section要和最大的page size對齊。

```c
.text ALIGN (CONSTANT (MAXPAGESIZE)) : { *(.text) }
```


<a name="expr-sym"></a>
## Symbol命名規則
沒有被`"`引用的情況下：

* 允許的開頭字元
	* 大小寫英文字母
  * `_`
  * `.`
* 名稱中間允許字元
	* 大小寫英文字母
  * 數字
  * `_`
  * `.`
	* `-`
* 不可以和linker script keyword相同名稱

如果你symbol名稱要用奇怪的字元或是和keyword相同的話，請將symbol名稱的開始結尾加上`"`符號，範例如下：

```c
"SECTION" = 9;
"with a space" = "also with a space" + 10;
```

由於symbol名稱可以有非英文字母和數字，所以不建議中間有空白字元。舉例來說，`A-B`是一個符號，但是`A - B`就是一個expression操作，表示symbol `A`減去symbol `B`


<a name="expr-oph-sec"></a>
## 孤兒 Section 
孤兒 Section 指的是在輸入object檔案中的section，而這些section linker script裏面並沒有描述該怎麼處理。遇到這種狀況，linker還是會把這些section放到輸出object檔案中，規則為：

* 放在輸出object檔案相同性質section的後面，如程式碼或是資料、是否要load到記憶體等。如果是ELF格式的話，ELF flag也是性質比較的一部份。
* 如果放不進去，就塞在檔案後面

如果這些孤兒section符合C語言[identifier規範](http://cs.smith.edu/~thiebaut/classes/C_Tutor/node4.html)(通常不是以`.`開頭），linker會幫忙加入兩個symbol：`__start_SECNAME`和`__stop_SECNAME`表示該section的起始和結束位址。而`SECNAME`就是該孤兒section名稱。


<a name="expr-lcnt"></a>
## Location Counter 
前面有提過`.`這個符號是location counter。而location counter本身的涵意就是**目前輸出位置**。而`.`
可以出現在`SECTIONS`命令中的任何expression。

除了使用`.`來代表目前輸出位置外，你也可以直接更改`.`的數值，這樣做就是更動目前輸出位置。不過要注意的是不要用做減法運算，這樣代表把目前位置往前移動，往前移動表示接下來寫入的東西就很有可能蓋掉前面重疊部份的資料。另外你可以直接把`.`的值加上你要的數量，那麼下一個symbol或是section就會和目前位置有一段保留空間可以使用。我們看下面的範例：

```c
SECTIONS
{
	output :
	{
		file1(.text)
		. = . + 1000;
		file2(.text)
		. += 1000;
		file3(.text)
	} = 0x12345678;
}
```

這個範例我們可以看到

* 輸出object檔案有一個section，該section名稱為`output`
* `output` section裏面存放了
	* `file1`的`.text` section
	* `file2`的`.text` section
	* `file3`的`.text` section
* `file1`的`.text` section和`file2`的`.text` section中間相距1000
* `file2`的`.text` section和`file3`的`.text` section中間也相距1000
* 未始用到的空間請填入`0x12345678`，想要劇情回顧的請看[這邊](http://wen00072-blog.logdown.com/posts/246070-study-on-the-linker-script-2-setcion-command#sec-input-desc-ex)

`.`雖然是location counter，然而在不同的區塊使用會有不同的意義。
先看一個例子

```c
SECTIONS
{
  . = 0x100
  .text: {
  	*(.text)
  . = 0x200
  }
  . = 0x500
  .data: {
  	*(.data)
  	. += 0x600
  }
}
```

我們可以看到`.`出現的地方有四個地方，在`.text`和`.data`以內有各有一個，不在`.text`和`.data`以內有兩個。也就是說一種是在section描述（就是`.text`和`.data`)之內，另外就是section描述之外。

* `.`放在section描述**裏面**的話它的location是從section開頭開始算
* `.`放在section描述**外面**的話它的location是從0

有了這樣的概念後，我們再回去看這範例在講三小？

* 輸出object檔案有兩個section，分別為`.text`和`.data`
* 所有輸入object檔案中的`.text` section請放到`.text`中
* 所有輸入object檔案中的`.data` section請放到`.data`中
* `.text` section 起始點為0x100
* 由於`.`被設成0x200，以致於輸入object檔案的`.text`存放超過0x100 + 0x200的空間都有可能被後面的資料複寫掉。
* `.text` section 結束後，請保留0x500的空間
* `.text` section 最後0x500的位置為`.data` section的起始點
* 最後一個輸入object檔案的`.data` section放入輸出object的`.data` section後，再從section中目前位置保留0x600的空間。

手冊中有特別提到在section描述外面使用`.`需要特別注意的地方，它舉的例子如下：

```c
SECTIONS
{
  start_of_text = . ;
  .text: { *(.text) }
  end_of_text = . ;
  
  start_of_data = . ;
  .data: { *(.data) }
	end_of_data = . ;
}
```

這個範例在`.data`和`text`前後都加了一個symbol，數值為當時location counter的位置。看起來一切安好，然而如果輸入object檔案中有section不是`.data`也不是`.text`，例如放&quot;Hello world\n&quot;的`.rodata` ([參考資料](http://www.lisha.ufsc.br/teaching/os/exercise/hello.html 你知道你寫的"Hello world"放在什麼地方嘛？))，linker還是要把這些section放到輸出object檔案中。你覺得他會放在邊呢？手冊上說linker script中的symbol會被視為接在**前一個**section 後面，所以最後就會變成這樣：

```c
SECTIONS
{
  start_of_text = . ;
  .text: { *(.text) }
  end_of_text = . ;
  
  start_of_data = . ;
  .rodata: { *(.rodata) }
  .data: { *(.data) }
	end_of_data = . ;
}
```

如此一來，如果你以為`start_of_data`就是`.data`開始位址，在你的程式中拿來做事，保證ＧＧ。因為`start_of_data`現在變成`.rodata`的起始位址了。

要確保`start_of_data`一定在`.data` section前面的話，正確的做法是在`start_of_data = . ;`前面加上`. = .`強迫更新location counter，列出完整script如下：

```c
SECTIONS
{
  start_of_text = . ;
  .text: { *(.text) }
  end_of_text = . ;
  
  . = . ;
  start_of_data = . ;
  .data: { *(.data) }
	end_of_data = . ;
}
```


<a name="expr-op"></a>
## Operators 
和C相容，請[自己看](https://sourceware.org/binutils/docs/ld/Operators.html#Operators)，反正沒多少英文。


<a name="expr-eval ></a>
## 計算結果 
原來標題是「Evaluation」直接翻成評估，估算都很詭異。自作主張就是計算結果，反正這個章節就是在講這回事。

linker很懶惰，所以不到要用的時候就不會去算expression的結果。以下是他的計算順序

* 第一個section的起始位址，memory region size等一開始就需要取得的expression值
* 當開始linker後才可以確定的expression，例如某個section內的symbol需要前面section處理完畢才能取得目前section的位置，然後才能開始計算裏面symbol相關的expression
* section size也要等該section link完畢才可能計算
* 和`.`相關的expression同要等該section link完畢才可能計算

由於有這樣的先後關係，如果你的script沒寫好可能就會遇到時間順序不同造成需要的expression裏面的element還沒計算完畢，然後你的linker就會噴錯誤出來。

手冊的[範例](https://sourceware.org/binutils/docs/ld/Evaluation.html#Evaluation)看不懂，不解釋了。


<a name="expr-sec"></a>
## Expression 計算結果和absolute/relative address的關係
** 這邊我不是很確定我有理解正確，請自行斟酌。另外看完後感覺上relative/absolute symbol和relative/absolute address是相同的東西，但是手冊上又沒有明講。所以我這邊語法會有點混亂。 **

這邊要先定義兩個名詞才能理解這個section在講三小。列出如下

* relative symbol/address
* absolute symbol/address


這兩個東西都是在講輸出object檔案的`SECTIONS`命令中的symbol或address。而這些symbol或 address可能宣告在`section`的裏面或外面。知道這樣的前提後，我們可以開始定義：

* relative symbol/address
	* 這個symbol或address的值代表的是section到該symbol的offset
* absolute symbol/address
  * 這個symbol或address的值和section無關，而是寫死的
  
知道的這樣定義後，我們可以再問，然後呢？

然後有相對特性的在relocate的時候只要更動section數值就好，而寫死的就沒有辦法動手腳。所以， relative symbol可以relocate而absolute symbol不行。

還不是很瞭長怎麼樣嘛？先看手冊的的範例好了：

```c
SECTIONS
{
  . = 0x100;
  __executable_start = 0x100;
  .data :
  {
    . = 0x10;
    __data_start = 0x10;
    *(.data)
  }
  ...
}
```

我們可以看到

* 輸出object檔案的起始位址和`__executable_start`為0x100，這是一個**absolute address/symbol**
* `.data`的真正資料開始位置距離`.data`位置0x10，`__data_start`也是相同。這兩個是**relative address/symbol**

好啦，知道這兩個關係後，我們再回來討論計算symbol值的expression。由於linker script 命令處理回來的值有些是relative有些是absolution，所以在寫script的時候要注意。手冊描述的地方目前看不懂，懶得搞懂。不論如何，手冊提供了linker處理expresssion時對於absolute 和 relative的行為準則。

* 計算結果為absolute
  * unary操作(如~0x11)absolute位置的address結果為absolute address
  * binary操作(如A + B)，兩個operand都是absolute address的結果為absolute address
  * binary操作(如A + B)，兩個operand都是數字的話的結果為absolute 
  * binary操作(如A + B)，兩個operand中一個是absolute address，另一個為數字的結果為absolute address
* 計算結果為relative，假設在同一個section下
  * unary操作(如~0x11) relative位置的address結果為relative address
	* binary操作(如A + B)，兩個operand都是relative address的結果為relative address
  * binary操作(如A + B)，一個operand是relative address另外一個是數字的結果為relative address
* 計算結果需要轉換後變成absolute address的情況
	* binary操作(如A + B)，兩個operand都是relative address，但是是不同的section，需要先轉成absolut address再操作，的結果為absolute address 
	* binary操作(如A + B)，兩個operand中一個是relative address，另外一個是absolute address，需要把relative address先轉成absolut address再操作，的結果為absolute address

[sub-expression](http://chortle.ccsu.edu/java5/Notes/chap09B/ch09B_10.html)（就是express裏面合法的express如a+b-c，可以拆成a+b，他的結果再跟c相加，而a+b就是一個sub-expression），的處理absolute/relative規範如下：

* 操作處理數字結果為數字
* 比較(|| &&)的結果也是數字
* binary操作包含邏輯操作(如A + B)，兩個operand都是relative address的結果為數字
* binary操作包含邏輯操作(如A + B)，兩個operand都是absolute address的結果為數字
* 不是以上的操作，兩個operand都是relative address的結果為relative address
* 不是以上的操作，一個operand是relative address另外一個是數字的結果為relative address
* 不是以上的操作，有absolute address的操作結果為absolute
  
如果有需要，你可以使用`ABSOLUTE()`命令強迫section裏面的symbol值為absolutio，範例如下。

```c
SECTIONS
{
	.data : { *(.data) _edata = ABSOLUTE(.); }
}
```

`_edata`沒用`ABSOLUTE()`命令的話會是一個relative symbol，因為加了`ABSOLUTE()`命令所以linker把他視為absolution symbol。



<a name="expr-btfun"></a>
## 內建函數 

* `ABSOLUTE(expr)`
	* 把expr內的結果視為absolute的值，通常會在section內使用，用了以後這個結果將會無法relocate
* `ADDR(section)`
	* 取得section名稱的VMA

前面兩個命令可以用下面範例說明

```c
SECTIONS 
{ 
	...
  .output1 :
  {
  	start_of_output_1 = ABSOLUTE(.);
  	...
  }
  .output :
  {
  	symbol_1 = ADDR(.output1);
 	  symbol_2 = start_of_output_1;
    ...
  }
  ... 
}
```

這邊我們可以看到： `start_of_output_1`，`symbol_1`, 和`symbol_2`的值理論上是相同的。但是性質上`symbol_1`是relative，而其他兩個symbol是absolute。

* `ALIGN(align)`
* `ALIGN(exp,align)`
先講`ALIGN(exp,align)`，這個命令是計算expr後，回傳`align`位址後面第一個符合alignment的位址。而`ALIGN(align)`可以視為`ALIGN(., align)`，也就是說這個命令會回傳`.`的後面符合alignment的位址。手冊提供的範例如下，因為很容易望文生義，就不解釋了：

```c
SECTIONS 
{ 
	...
	.data ALIGN(0x2000): 
  {
		*(.data)
		variable = ALIGN(0x8000);
	}
	... 
}
```

* `ALIGNOF(section名稱)`
取得`section`後面符號alignment的位置。要注意的是section要已經被分配出來，否則linker會噴錯誤給你看。範例一樣容易望文生義，不解釋。

```c
SECTIONS
{ 
	...
  .output
  {
  	LONG (ALIGNOF (.output))
  	...
  }
    ... 
}
```

* `BLOCK(exp)`

和`ALIGN()`相同。是舊版的linker使用的命令。

* `DATA_SEGMENT_ALIGN(maxpagesize, commonpagesize)`
* `DATA_SEGMENT_END(exp)`
* `DATA_SEGMENT_RELRO_END(offset, exp)`

**看不懂，不想弄懂。跳過。**

* `DEFINED(symbol)`
如果symbol已經被收進symbol table就回傳1，否則回傳0。手冊示範如果沒定義該symbol就自己生一個如下。

```c
SECTIONS 
{ 
	...
  .text : 
  {
  	begin = DEFINED(begin) ? begin : . ;
  	...
  }
  ...
}
```

* `LENGTH(region)`
回傳你在`MEMORY`命令中設定的region size

* `LOADADDR(section名稱)`
回傳section的名稱的LMA位址

* `LOG2CEIL(exp)`
取exp的log，不知道用在啥子地方。

* `MAX(exp1, exp2)`
回傳exp1和exp2比較大的數值

* `MIN(exp1, exp2)`
回傳exp1和exp2比較小的數值

* `NEXT(exp)`
回傳exp計算結果的數值記憶體之後的可使用的空間。如果沒有使用`MEMORY`命令設定不連續的空間，這指令效果和`ALIGN`命令相同。

* `ORIGIN(region名稱)`
回傳你在`MEMORY`命令設定的region的起始位址

* `SEGMENT_START(segment名稱, default)`
回傳`segment`的起始位置。還記得ELF program header的segment？我不知道和這個是不是相同。`default`除非有透過`ld -T`參數更動，否則就是預設值。手冊沒有寫預設值是多少。但是從ld --verbose看到的使用範例是用在指定程式碼開始執行的地方。有沒有覺得0x400000很眼熟呢？不熟？那算了。

```c
  PROVIDE (__executable_start = SEGMENT_START("text-segment", 0x400000)); . = SEGMENT_START("text-segment", 0x400000) + SIZEOF_HEADERS;
  . = SEGMENT_START("ldata-segment", .);
```

* `SIZEOF(section名稱)`
回傳section的size，如果該section還沒被分配，linker就吐錯誤給你看。
下面的例子中的`symbol_1`和`symbol_2`的值是相同的。這蠻容易理解，我列出來主要是要讓大家多看例子。`.start`, `.end`的用法我看過好幾次。

```c
SECTIONS
{ 
  ...
  .output 
  {
  .start = . ;
  ...
  .end = . ;
  }
  symbol_1 = .end - .start ;
  symbol_2 = SIZEOF(.output);
... 
}
```

* `SIZEOF_HEADERS`
* `sizeof_headers`
取得輸出object檔案的header size。如果你使用ELF格式，又有自行加programer header的話，ld會噴錯誤。原因是ld預期的是ELF規範的program header，因此放不下新增的program header。所以你有多的program header的話，請不要用這個指令。


<a name="imp"></a>
## Implicit Linker Scripts
linker吃的檔案處理順序如下

1 object檔，開始link
2 不是object檔，就當linker script吃進去
3 不是object檔案也不是linker script檔案，噴錯誤然後 GG

所以，Implicit Linker Script指的是項目2吃進來的script。linker會把這個當作目前linker script的補強，而不是取代。另外由於吃進來的script順序不同，可能會出現先讀入並link 三個object檔案後，才讀到Implicit Linker Script，所以這個Implicit Linker Script無法對已經link處理。
 
<a name="todo"></a>
## 待釐清項目
* dynamic symbol (不知道是三小)
* warning symbol (不知道是三小)
* constructor symbol (不知道是三小)
* 3.6.6看不懂，跳過。
* dynamic loader
* Initialize section
* PT_INTERP和/lib64/ld-linux-x86-64.so.2的關係
* DATA_SEGMENT_ALIGN(maxpagesize, commonpagesize)和他的朋友們
* ld -M 

<a name="ref"></a>
## 參考資料

* [GNU linker ld: Linker Scripts](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)
* [from Source to Binary: How GNU Toolchain Works](http://www.slideshare.net/jserv/from-source-to-binary-how-gnu-toolchain-works)
* [Linker Scripts - OSDev Wiki](http://wiki.osdev.org/Linker_Scripts)
* [Embedded Programming with the GNU Toolchain](http://www.bravegnu.org/gnu-eprog/index.html)
* [Stackoverflow: What does each column of objdump's Symbol table mean?](http://stackoverflow.com/questions/6666805/what-does-each-column-of-objdumps-symbol-table-mean)
* [System V Application Binary Interface - DRAFT - 24 April 2001](http://refspecs.linuxfoundation.org/elf/gabi4+/contents.html)
* [Linux Foundation: Referenced Specifications](http://refspecs.linuxfoundation.org/)
* [Linux Standard Base Core Specification 4.1](http://refspecs.linuxfoundation.org/LSB_4.1.0/LSB-Core-generic/LSB-Core-generic/book1.html)
* [virtual and physical addresses of sections in elf files](http://stackoverflow.com/questions/6218384/virtual-and-physical-addresses-of-sections-in-elf-files)
* [Understanding ld-linux.so.2](http://www.cs.virginia.edu/~dww4s/articles/ld_linux.html)
* [Using as: 7.11](https://sourceware.org/binutils/docs/as/Symver.html)
* [The True Story of Hello World](http://www.lisha.ufsc.br/teaching/os/exercise/hello.html)
* [Absolute symbol](http://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_chapter/ld_3.html#SEC13)
* [Sub-expression](http://chortle.ccsu.edu/java5/Notes/chap09B/ch09B_10.html)
* [Wikipedia: .bss](http://en.wikipedia.org/wiki/.bss)
* [Wikipedia: Code segment](http://en.wikipedia.org/wiki/Code_segment)
* [Wikipedia: Data segment](http://en.wikipedia.org/wiki/Data_segment)
* [stackoverflow: Flags in objdump output of object file](http://stackoverflow.com/questions/11196048/flags-in-objdump-output-of-object-file)
