---
layout: post
title: 'Hello World'
date: 2014-03-25 17:10
comments: true
categories: [C, Linux]
---
C語言學習者第一個程式

```c hello.c
#include <stdio.h>

int main(int argc, char **argv)
{
    printf("Hello world\n");

    return 0;
}
```

最近有人在問，去掉`{` `}`你真的了解每一行嗎？我試著回答一下

[#include <stdio.h\>](#inc)<br>
[int main(int argc, char **argv)](#entr)<br>
[    printf("Hello world\n");](#printf)<br>
[    return 0;](#ret_st)<br>

* [同場加映gcc -v hello.c](#gcc-v)

---
<a name="inc"></a>

* `#include <stdio.h>`使用`cpp hello.c > hello.i` 可以看到

```c hello.i
# 1 "hello.c"
...
extern int printf (__const char *__restrict __format, ...);
extern int sprintf (char *__restrict __s,
      __const char *__restrict __format, ...) __attribute__ ((__nothrow__));

...
# 2 "hello.c" 2

int main(int argc, char **argv)
{
    printf("Hello world\n");

    return 1;
}

```

直接編譯hello.i並執行

```text 
$ gcc hello.i
$ ./a.out 
Hello world

```
<a name="entr"> </a>
* `int main(int argc, char **argv)`

從[前面文章](http://wen00072.github.io/blog/2014/03/14/study-on-the-linker-script)可以看到gcc編譯時除了link library和本身的object 檔案外，還會多link一些binary，可以從這邊找看看main在那邊。

說明一下，nm顯示的資料中
`U`：表示undefined symbol
`T`：表示symbol在Text section中

```text 尋找main
$ nm -A /usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu/crt1.o /usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu/crti.o /usr/lib/gcc/x86_64-linux-gnu/4.6/crtbegin.o /usr/lib/gcc/x86_64-linux-gnu/4.6/crtend.o /usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu/crtn.o 2> /dev/null |grep main 
/usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu/crt1.o:                 U __libc_start_main
/usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu/crt1.o:                 U main
```

可以看到在crt1.o有一個Undefined symbol叫main，在來看看我們的hello.c這邊
```text main和hello.c
$ nm hello.o 
0000000000000000 T main
                 U puts
```

所以可以看到crt1.o會用到main()，當然誰去呼叫main這件事我們就視而不見吧。
另外`crt1.o`可以從man gcc看到被稱為startup file。

<a name="printf"> </a>

* printf這邊其實要釐清的觀念是，OS透過system call提供服務。因此我們可以用`strace`去觀察hello呼叫了哪些system call
```text strace畫面
$ strace ./hello
execve("./hello", ["./hello"], [/* 39 vars */]) = 0
...
write(1, "Hello world\n", 12Hello world
...
exit_group(1)  
```
可以看到事實上printf使用write system call去讓OS印字串到螢幕上。而為什麼不直接用`write(1, "Hello world\n", strlen("Hello world\n"));`呢？看[TLPI(書）](http://man7.org/tlpi/)有提到主要原因是system call有代價的，而libc實作了buffer減少system call呼叫的次數。有興趣的可以使用`man setvbuf`看看buffer的設定方式。

<a name="ret_st"> </a>

* `return 0;`可以看下面的程式碼

```c return_status.c
/* This demos return status */
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char **argv)
{
    int rval = 0;

    if (argc != 2) {
        printf("usage: %s return_status_number(0~255)\n", argv[0]);
        printf("Then observe return status in $?\n");
        printf("ex: $ %s 121; echo $?\n", argv[0]);

        return 2;
    }

    /* Convert return status to 0~255*/
    rval = atoi(argv[1]);
    rval = rval % 255;

    return rval;
}
```

跑看看便知道
```text return_status
$ ./return_status ; echo $?
usage: ./return_status return_status_number(0~255)
Then observe return status in $?
ex: $ ./return_status 121; echo $?
2

$ ./return_status 219 ; echo $?
219

$ ./return_status 34 ; echo $?
34
```

`$?`：`man bash`找`?`可以看到是顯示上次執行回傳狀態。因此不難理解return的值是有人會接起來的。Makefile就使用這個特性判斷build code是否有問題，我們可以測試一下

```makefile Makefile
test:
	./return_status 147
```

```text Makefile執行結果
$ make test
./return_status 147
make: *** [test] Error 147
```
<a name="gcc-v"></a>
* `gcc -v hello.c`輸出

```text gcc -v hello.c 輸出節錄
$ gcc -v hello.c 
Using built-in specs.
COLLECT_GCC=gcc
...
/usr/lib/gcc/x86_64-linux-gnu/4.6/cc1 -quiet -v -imultilib . -imultiarch x86_64-linux-gnu hello.c -quiet -dumpbase hello.c -mtune=generic -march=x86-64 -auxbase hello -version -fstack-protector -o /tmp/cczq4fLe.s
...
as --64 -o /tmp/ccSFUZMm.o /tmp/cczq4fLe.s
...
/usr/lib/gcc/x86_64-linux-gnu/4.6/collect2 --sysroot=/ --build-id --no-add-needed --as-needed --eh-frame-hdr -m elf_x86_64 --hash-style=gnu -dynamic-linker /lib64/ld-linux-x86-64.so.2 -z relro /usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu/crt1.o /usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu/crti.o /usr/lib/gcc/x86_64-linux-gnu/4.6/crtbegin.o -L/usr/lib/gcc/x86_64-linux-gnu/4.6 -L/usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu -L/usr/lib/gcc/x86_64-linux-gnu/4.6/../../../../lib -L/lib/x86_64-linux-gnu -L/lib/../lib -L/usr/lib/x86_64-linux-gnu -L/usr/lib/../lib -L/usr/lib/gcc/x86_64-linux-gnu/4.6/../../.. /tmp/ccSFUZMm.o -lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed /usr/lib/gcc/x86_64-linux-gnu/4.6/crtend.o /usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu/crtn.o

```

## 參考資料：
* [nm manual](https://sourceware.org/binutils/docs/binutils/nm.html)
* [(簡體中文)Linux C編程一站式學習: 第 19 章 彙編與C之間的關係](http://learn.akae.cn/media/ch19s02.html)
* [TLPI(書）](http://man7.org/tlpi/)
