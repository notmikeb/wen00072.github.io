---
layout: post
title: 'GNU: The C Preprocessor 導讀'
date: 2013-10-13 05:13
comments: true
categories: [C, GCC, C Preprocessor, MACRO]
---
C語言的前置處理(CPP: C preprocessor)是個很有趣的題目，知道在對的時機使用他們絕對能夠對於開發軟體有巨大的幫助。以下是我閱讀[GNU: The C Preprocessor](http://gcc.gnu.org/onlinedocs/cpp/index.html)的筆記。請注意這只是筆記，詳細資料請自行參考原本網頁。

<a name="目錄"></a>
## 目錄

- [測試環境](#測試環境)
- [概論](#測試環境)
- [Header Files](#Header Files)
- [巨集](#巨集)
- [Preprocessor Output](#Preprocessor Output)
- [gcc和CPP相關的參數](#gcc和CPP相關的參數)
- [參考資料](#參考資料)


<a name="測試環境"></a>
## 測試環境

- Ubuntu 12.04

- [回目錄](#目錄)


<a name="概論"></a>
## 概論

- Preprocesser: 輸入文字，產生更動過的文字x。產生出來文字會當作另外執行程式的輸入資料。 [出處](http://en.wikipedia.org/wiki/Preprocessor)
- CNU C有自訂專用的Preprocessor指令，如果程式有移植性考量，gcc可以加入`-std=c90`, `-std=c99`或`-std=c11`參數檢查。全開就用` -pedantic`參數

### 編碼

- Source character set
    - 為gcc內部CPP處理的字元編碼
    - gcc 讀入原始碼後會以UTF-8同構的字元編碼處理程式碼，CPP處理時的文字輸出(如gcc -E)也是使用Source character set
    - 可使用`-finput-charset= `參數指定source file 字元編碼
- Execution character set
    - CPP處理結束後會將結果轉換成指定的編碼，預設仍然是UTF8
    - 後面看不懂，不想看

### 初始行為

- Load檔案到記憶體內。GCC 支援不同Line end 格式(LF, CR LF, CR)
    - 標準C 最後一行沒有line end是未定義行為，GCC會跳出警告訊息
- 處理trigraphs
    - 史前遺跡，早期有些電腦沒有C需要的字元而使用trigraphs代替。如`??/` 代表 `\`
    - 需要gcc參數設定才會支援
- 將\斷行的statemenet合併成一行
- 把註解換成空白字元
    
### 取Token

- 空白鍵為分隔空白的單位
- 由左至右順序取token `a+++++b` -> `a++ ++ +b`而不是`a++ + ++b`
- Preprocessor token分類
    - identifiers: 單純的C合法token 如keyword, 字母和數字的組合
    - preprocessing numbers: 一般來說就是數字
    - String literals: 兩個`"`中間包含的字串，包括`#include "test.h"`
    - Punctuators: @ $ ` 以外ASCII的標點符號
    - other: 直接pass到preprocessr output，一般來說C compiler會把other token打槍。
        - Other chars:
            - @
            - $
            - `
            - 不是NUL的control char
            - ASCII 0x7F–0xFF (因為要支援國際字元、未來會討論存廢)
        - NUL在註解中會被忽略，一般程式碼則視為空白
- 基本上，preprocesser 輸出就是一個token

```c
#define MY() ddd
int main(void)
{
    int i,MY()__;
    return
}
```

展開後就是

```c
## 1 "b.c"
## 1 "<built-in>"
## 1 "<command-line>"
## 1 "b.c"

int main(void)
{
    int i,ddd __;
    return
}
```

### Preprossessor 語法概要

- 包含指令(directives)和巨集(macros)
- 提供
    - include檔案
    - 巨集展開
    - 條件式編譯
    - 行號控制：目前不是很懂，看起來是可以告訴compiler編譯intermediate時該statement是從哪個檔案的那一行過來的
    - compile時可以吐error或warning
- 除了gcc內建的directives外，directives以`#`開頭，前後都可以有空白

- [回目錄](#目錄)


<a name="Header Files"></a>
## Header Files

- Header file就是檔案，該檔案包含下面的元件，讓不同原始碼檔案共用。
    - C declarations
    - 巨集
- 原始碼檔案透過#include去取用這些Header Files的資料
- `#include`效果和複製header file到原始碼檔案一樣。你可以用`gcc -E 原始碼檔名`看到header file被放入。之所以這樣做就是讓使用者省去複製和多次修改相同介面的時間。

### #include語法

- `#include <file>` 
    - 使用系統內建的標頭檔。標頭檔除了放在預設的目錄外，還可以在GCC下使用`-I`參數指定標頭檔路徑
- `#include "file"`     
    - 自己原始碼專用的標頭檔。GCC搜尋順序
        - 同一份原始檔的目錄
        - `-iquote`指定的目錄
        - `-I`指定的目錄

## 標頭擋路徑

- `echo | gcc -v -x c -E -`可以列出gcc 搜尋標頭檔目錄
- gcc搜尋標頭檔順序是
    - 系統預設路徑
    - `-I`參數，一個以上`-I`路徑則由左至右開始搜尋
- 可以使用`-nostdinc`讓gcc不去搜尋系統標準的include path

## 避免因為多重#include重複定義

- 狀況說明
    - 原始碼檔案中include "file1"和"file2"
    - "file2"也有include "file1"
    - 最後變成原始碼檔案中放了兩份file1的內容，一方面多餘另一方面也會發生重複定義的情況
- 建議方式: [Include guard](http://en.wikipedia.org/wiki/Include_guard)

```c Include guard example
#ifndef MY_HEADER_H
#define MY_HEADER_H
 
#define MY_VAR (1)
 
#endif /* MY_HEADER_H */
```

## Computed Includes

- 用來節省太多條件式#include如

```c 條件式include
#ifdef LINUX
    #include <linux_plat.h>
#else
    #include <windows_plat.h>
#endif
```

- 使用方式

```c C檔案
#include MY_DEF_H
```

```makefile Makefile
CFLAGS=-DMY_DEF_H="<linux_plat.h>"
```

## System Headers

- GCC會忽略掉系統標頭檔的Warning，如果其他一般標頭檔想要有相同的處理方式，可以使用`-isystem 路徑`讓GCC將該目錄下的標頭檔視為系統標頭導

- [回目錄](#目錄)


<a name="巨集"></a>
## 巨集

- Marco分成
    - 擬函數巨集: `#define foo()`: 重點是()，()前面不得有空格
        - 定義完後可以直接當作function使用，所以可以玩`callback = foo;`這樣的描述
    - 資料相關巨集

### 擬函數巨集

- 擬函數巨集可以吃參數，以`,`分開。所以` MAC(buf[x = 1, u + 1])`這樣的程式碼會被拆成兩個參數，請使用`MAC((buf[x = 1, u + 1]))`
- 擬函數巨集的參數可以空白，多個參數請用`,`隔開。

```c 擬函數巨集參數空白範例
#define MY_MAC(x,y) my_mac(x,y)
...
MY_MAC(,);
MY_MAC(a,b);
MY_MAC(a,);
MY_MAC(,b);
```

- 擬函數巨集定義的描述如果有使用`"`並且和參數相同，最後並不會被替換。
    - `FOO(X) _bar(x, "x");` -> `FOO(bar) _bar(bar, "x");` 

### 字串轉換

```c
#define ENCLOSE_QUOTE(VAR) #VAR
```
- `ENCLOSE_QUOTE(test);` -> `"test"`

```c
#define PLAY 0
#define ENCLOSE_QUOTE(VAR) #VAR
#define TO_STR(VAR) ENCLOSE_QUOTE(VAR)
```

- `ENCLOSE_QUOTE(PLAY);` -> `"PLAY"`
- `TO_STR(PLAY)` -> `ENCLOSE_QUOTE(0)` -> `"0"`
    - 利用擬函數巨集展開參數的特性

### 合體
`echo -e "#define  MYDEF(PARAM1, PARAM2)  PARAM1##PARAM2 \n\n MYDEF(xxx, yyy)" | gcc -E -x c -`

### Variadic Macros:  兩種方式，不能一個巨集同時使用

- C99語法：`#define my_printf(...) printf("myprintf: " __VA_ARGS__)`
- GNU語法：`#define my_printf(arg...) printf("myprintf: " args)`
- 可以在varidic marco中明確指定參數
    - `#define my_printf(format, ...) fprintf(stderr, format, __VA_ARGS__)`	
        - 缺點: 使用`my_printf("test");`會GG，因為被轉成`fprintf(stderr, "test",);`
        - 解法: `#define my_printf(format, ...) fprintf(stderr, format, ##__VA_ARGS__)`	
            - GNU會把`,` 硬食掉
            
### GNU C支援的predefined Macros (節錄)
#### 標準Macro: 可以望文生義就不解釋了
- `__FILE__`
- `__LINE__`
- `__func__`
    - GCC也有一樣功能的`__FUNCTION__`
- `__DATE__`
- `__TIME__`
- `__STDC__`：顯示目前是否使用Standard C編譯程式碼
- `__STDC_VERSION__`：Standard C版本
- `__cplusplus`：是否為C++編譯環境
- `__OBJC__`：是否為object C編譯環境
- `__ASSEMBLER__`：是否在組合語言環境

#### GNU Marco

- `__COUNTER__`：回傳一個從0開始的數字，每次呼叫一次加一
- GCC版本
    - `__VERSION__`
    - 細部版號
        - `__GNUC__`
        - `__GNUC_MINOR__`
        - `__GNUC_PATCHLEVEL__`
- `__GNUG__`：是否是用GNU C++
- `__INCLUDE_LEVEL__`：標頭檔被引用的深度，從0開始算。如a->b->c，c就是2
- `__ELF__`：輸出格式是否為elf
- `__TIMESTAMP__`

### 系統內建Macro

- 和平台有關，使用下面指令查詢：`echo "" | gcc -x c -E -dM -`
- C標準規範使用`__`包夾或是`_`+`大寫`為系統內建Macro，其name space保留。

### Undefine/Redefine

```c Undefine/Redefine範例
#define FOO BAR
...
#undef FOO
#define FOO BAR2
```

### 密技和怪招
####  Misnesting

- 間接呼叫

```c 間接呼叫範例
#define MY_PRT(X) printf(x)
#define P_TEST(M) M("test")
...
P_TEST(MY_PRT);
```

- 不對稱的括號

```c 不對稱的括號範例
#define TEST(x) func1(x, "test"
...
test(my_param));
```

- 硬食分號

```c test.c
#define MY_TEST(x) { printf("test\n"); }
...
    if (bit)
    	MY_TEST(bit);
    else
    	printf("else\n");
```

編譯會錯誤，原因是因為分號會讓gcc判斷if statement已經結束

```text 錯誤訊息
cc     test.c   -o test
test.c: In function ‘main’:
test.c:23:5: error: ‘else’ without a previous ‘if’
make: *** [test] Error 1
```

- 解法: `do {...} while (0)`

```c test.c
#define MY_TEST(x) do { printf("test\n"); } while(0)
...
    if (bit)
    	MY_TEST(bit);
    else
    	printf("else\n");
```     

- 除非明確了解巨集內容，否則避免直接傳函數進去巨集。`MY_TEST(func(xy,xx), 10));`
    - 原因是巨集內也許會呼叫參數好幾次，如果傳進去的函數執行成本很高，就會發生悲劇。


- [回目錄](#目錄)


<a name="Preprocessor Output"></a>
## Preprocessor Output

- 可以使用`gcc -E 檔案`或是`cpp 檔案`觀察CPP的展開，和CPP相關格式請參考[這邊](http://gcc.gnu.org/onlinedocs/cpp/Preprocessor-Output.html#Preprocessor-Output)。
- [回目錄](#目錄)


<a name="gcc和CPP相關的參數"></a>
## gcc和CPP相關的參數

- 大部分選項和參數可以以空白隔開(`-I /usr/include`)或是直接連在一起(`-I/usr/include`)

### 節錄

- `-D 文字`: 定義`文字`巨集為1
    - 測試`echo "TEST" | gcc -E -xc -DTEST -`
- `-D 文字=文字`：定義`文字`巨集
    - 測試`echo "TEST" | gcc -E -xc -DTEST=CPP -`
- `-U`: undefine 巨集，如果參數同時有`-U`和`-D`的話優先先出現的先做
- `-I dir`:指定搜尋header file目錄
    - `dir`以`=`開頭，gcc會取代成sysroot路徑，參考`--sysroot`和`-isysroot`
- `-M`:列出要檔案要吃的source code和header files
    - 測試:

```makefile -M範例
$ gcc -M b.c
b.o: b.c /usr/lib/gcc/x86_64-linux-gnu/4.6/include/stdint.h \
 /usr/include/stdint.h /usr/include/features.h \
 /usr/include/x86_64-linux-gnu/bits/predefs.h \
 /usr/include/x86_64-linux-gnu/sys/cdefs.h \
 /usr/include/x86_64-linux-gnu/bits/wordsize.h \
 /usr/include/x86_64-linux-gnu/gnu/stubs.h \
 /usr/include/x86_64-linux-gnu/gnu/stubs-64.h \
 /usr/include/x86_64-linux-gnu/bits/wchar.h /usr/include/stdio.h \
 /usr/lib/gcc/x86_64-linux-gnu/4.6/include/stddef.h \
 /usr/include/x86_64-linux-gnu/bits/types.h \
 /usr/include/x86_64-linux-gnu/bits/typesizes.h /usr/include/libio.h \
 /usr/include/_G_config.h /usr/include/wchar.h \
 /usr/lib/gcc/x86_64-linux-gnu/4.6/include/stdarg.h \
 /usr/include/x86_64-linux-gnu/bits/stdio_lim.h \
 /usr/include/x86_64-linux-gnu/bits/sys_errlist.h b.h
```

- `-MM`:列出要檔案要吃的source code和非系統的header files

```text 範例
$ gcc -MM b.c
b.o: b.c b.h
```

- `-MF file`:配合`-M`, `-MM`時將結果寫入`file`
- `-MG`: `-M`, `-MM`parse到header file不存在會跳錯誤，而加上`-MG`後會假裝他們存在，照樣吐出結果。

```text -MG範例
$ mv b.h a
$ gcc   -MM b.c
b.c:3:15: fatal error: b.h: No such file or directory
compilation terminated.
$ gcc -MG  -MM b.c
b.o: b.c b.h

```

- `-MP`: 順便把depend的target加入

```text -MP範例 
$ gcc -MP -MM b.c
b.o: b.c b.h

b.h:
```

- `-MT` :指定target

```text -MT範例 
$ gcc -MT test -MM b.c
test: b.c b.h
```

- `-MD`: 直接編譯，同時把會吃的檔案和header files放在*.d檔

```text -MD範例
$ rm a.out
$ gcc -MD b.c
$ ls a.out
a.out
$ cat b.d
b.o: b.c /usr/lib/gcc/x86_64-linux-gnu/4.6/include/stdint.h \
 /usr/include/stdint.h /usr/include/features.h \
 /usr/include/x86_64-linux-gnu/bits/predefs.h \
 /usr/include/x86_64-linux-gnu/sys/cdefs.h \
 /usr/include/x86_64-linux-gnu/bits/wordsize.h \
 /usr/include/x86_64-linux-gnu/gnu/stubs.h \
 /usr/include/x86_64-linux-gnu/gnu/stubs-64.h \
 /usr/include/x86_64-linux-gnu/bits/wchar.h /usr/include/stdio.h \
 /usr/lib/gcc/x86_64-linux-gnu/4.6/include/stddef.h \
 /usr/include/x86_64-linux-gnu/bits/types.h \
 /usr/include/x86_64-linux-gnu/bits/typesizes.h /usr/include/libio.h \
 /usr/include/_G_config.h /usr/include/wchar.h \
 /usr/lib/gcc/x86_64-linux-gnu/4.6/include/stdarg.h \
 /usr/include/x86_64-linux-gnu/bits/stdio_lim.h \
 /usr/include/x86_64-linux-gnu/bits/sys_errlist.h b.h
```

- `-MMD`: 直接編譯，同時把會吃的檔案和非系統header files放在*.d檔

```text -MMD範例
$ rm a.out
$ gcc -MMD b.c
$ ls a.out
a.out
$ cat b.d
b.o: b.c b.h
```

- `-dM`: 列出編譯時所有的巨集，不列出展開巨集的結果, 通常配合`-E`
- `-dD`: 列出編譯時不包含predefine的巨集以及展開巨集的結果, 通常配合`-E`
- `-dN`: 和`-dD`的差別是巨集只列出名稱不列出要展開的內容, 通常配合`-E`
    - `#define foo bar` `-dD`輸出 `#define foo bar `，`-dN`結果 `#define foo`
- `-dI`:不顯是巨集，而是顯示`#include`的指令
- `-C`: 保留註解，不包含巨集展開的部份註解
- `-CC`: 保留註解，包含巨集展開的部份註解
- `-H`:列出編譯時會參考的header files

- [回目錄](#目錄)



<a name="參考資料"></a>
## 參考資料

- [GNU: The C Preprocessor](http://gcc.gnu.org/onlinedocs/cpp/index.html)
- 測試檔案

```c b.c 
#include <stdint.h>
#include <stdio.h>
#include "b.h"


#define foo  (bit,lfsr) /* XXX */
#define bar(x) lose(x)
#define lose(x) (1 + (x))

#if aaa
#define ddd ddds
#endif

int main(void)
{
    uint16_t lfsr = time(0);
    unsigned bit;
    unsigned period = 0;
    int c= 644;

    printf("%s\n", __BASE_FILE__);
    printf("%s\n", __VERSION__);
    if (bit)
        MY_TEST(bit);
    else
        MY_TEST(bit);
        printf("%d\n", (lfsr,bit,c));
        printf("%d\n", (bit, lfsr));

#line 10
printf("test:%d, %s\n", __LINE__, __FILE__); 

#if 0
    lfsr = typeof(bit) 10;
#endif

    bar(foo);
printf("test:%d, %s\n", __LINE__, __FILE__); 

    return 0;
}

```
```c b.h
#define MY_TEST(x) do { printf("test\n"); } while(0)
```
- [回目錄](#目錄)
