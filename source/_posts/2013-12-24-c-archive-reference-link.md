---
layout: post
title: 'C語言archive 明明有symbol卻link時出現unsolved symbol錯誤'
date: 2013-12-24 02:29
comments: true
categories: [C, Linux, Linker]
---
## 說明
[GNU ld 手冊](#參考資料)提到，archive (可能是*.a或是*.o檔)在link時預設只會reference一次，如果沒有注意的話會發現明明有symbol卻link時出現unsolved symbol錯誤

--- 
## 測試
下面是`測試程式碼`、`liba.a包含的程式碼`和`liba.a包含的程式碼`，可以看到liba.a的function去呼叫libb的function，反之亦然。

```c test.c
#include "libb.h"
#include "liba.h"

int main(void)
{
    test_liba();
    test_libb();

    return 0;
}
```

```c liba.c
#include "libb.h"
#include <stdio.h>

void test_liba(void)
{
    from_libb();
}

void from_liba()
{
    printf("%s\n", __PRETTY_FUNCTION__);
}
```

```c libb.c
#include "liba.h"
#include <stdio.h>

void test_libb(void)
{
    from_liba();
}

void from_libb()
{
    printf("%s\n", __PRETTY_FUNCTION__);
}
```
而測試的Makefile如下，變數說明如下

- $^: prerequisites
- $@: target
- $>: 第一個prerequisite

```makefile Makefile
all: liba.a libb.a test.o
	gcc -o test $^

liba.a: liba.o liba.h
	ar crv $@ $<

libb.a: libb.o libb.h
	ar crv $@ $<

clean:
	rm -rf *.o *.a *~
```

編譯輸出如下，正如前面的預想
```text Result
$ make
cc    -c -o liba.o liba.c
ar crv liba.a liba.o
a - liba.o
cc    -c -o libb.o libb.c
ar crv libb.a libb.o
a - libb.o
cc    -c -o test.o test.c
gcc -o test liba.a libb.a test.o
test.o: In function `main':
test.c:(.text+0x5): undefined reference to `test_liba'
test.c:(.text+0xa): undefined reference to `test_libb'
collect2: ld returned 1 exit status
make: *** [all] Error 1
```
---
## 解法
### 解法一：調整link順序

- 將Makefile的`all: liba.a libb.a test.o`順序改成`all: test.o liba.a libb.a`可以正常編譯
    - 延伸測試
        - 將Makefile的`all: liba.a libb.a test.o`順序改成`all: liba.a test.o libb.a`會發生下面的錯誤
- 關於link怎麼resovle symbol部份就不在本篇中討論，抱歉。

```text Result
$ make
cc    -c -o liba.o liba.c
ar crv liba.a liba.o
a - liba.o
cc    -c -o test.o test.c
cc    -c -o libb.o libb.c
ar crv libb.a libb.o
a - libb.o
gcc -o test liba.a test.o libb.a
test.o: In function `main':
test.c:(.text+0x5): undefined reference to `test_liba'
libb.a(libb.o): In function `test_libb':
libb.c:(.text+0x5): undefined reference to `from_liba'
collect2: ld returned 1 exit status
make: *** [all] Error 1

```
        
### 解法二：使用`--start-group` ... `--end-group`參數

- 將Makefile 的`gcc -o test $^`改成`gcc -o test -Wl,--start-group $^ -Wl,--end-group`
- 參數說明如下
    - `-Wl`: 告訴gcc傳參數給linker
    - `--start-group` ... `--end-group`
        - 告訴linker在中間的`...`重複找尋symbol
        - **這樣的行為有效能代價，請謹慎使用**

- 編譯並執行結果如下

```text Result
$ make
cc    -c -o liba.o liba.c
ar crv liba.a liba.o
a - liba.o
cc    -c -o libb.o libb.c
ar crv libb.a libb.o
a - libb.o
cc    -c -o test.o test.c
gcc -o test -Wl,--start-group liba.a libb.a test.o -Wl,--end-group

$ ./test
from_libb
from_liba
```

---
## 版本資訊

- GNU ld 2.2

---
<a name="參考資料"></a>
## 參考資料

- man ld
    - 搜尋start-group
