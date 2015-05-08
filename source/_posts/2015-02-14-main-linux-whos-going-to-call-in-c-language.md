---
layout: post
title: 'Linux中誰來呼叫C語言中的main?'
date: 2015-02-14 15:56
comments: true
categories: [C, Linux, binutil]
---
記得很久以前聽說在Linux執行檔案時，真正的起始點並不是main，加上[之前](http://wen00072-blog.logdown.com/posts/188339-study-on-the-gnu-ld)有看到單純ld會幫你偷偷link一些沒看過的object檔案，所以這次就來看到底真相為何？

## 測試環境
因為~~很假掰~~想要順便接觸一下ARM的組語，所以這次測試就使用Qemu跑ARM的Debian。

```
$ lsb_release -a
No LSB modules are available.
Distributor ID:	Debian
Description:	Debian GNU/Linux 8.0 (jessie)
Release:	8.0
Codename:	jessie

$ file /bin/ls
/bin/ls: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, BuildID[sha1]=571db48d9c9e4625b7da206e748e41c237f2b202, stripped
```

## 測試原始碼，一樣是大家熟悉的Hellow world

```c hello1.c
#include <stdio.h>

int main()
{
	printf("Hello World\n");

	return 0;
}
```

不知道各位還記得前面有提過，執行檔中有`.text`的section。要執行的機械碼會放在這邊。我們先來看看hello1執行檔會從那邊開始？

```
$ readelf -h hello1
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00 
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           ARM
  Version:                           0x1
  Entry point address:               0x102f0
  Start of program headers:          52 (bytes into file)
...
Section header string table index: 33
```

從readelf可以看到起始點為0x102f0，那麼0x102f0是在那邊呢？我們再去看symbol table可以看到很巧的就是`.text`的起始點。

```
$ objdump -t hello1

hello1:     file format elf32-littlearm

SYMBOL TABLE:
00010134 l    d  .interp	00000000              .interp
...
000102f0 l    d  .text	00000000              .text
```

好了，那麼`.text`這邊起始的程式是什麼？

```asm
Disassembly of section .text:

000102f0 <_start>:
   102f0:       e3a0b000        mov     fp, #0
   102f4:       e3a0e000        mov     lr, #0
   102f8:       e49d1004        pop     {r1}            ; (ldr r1, [sp], #4)
   102fc:       e1a0200d        mov     r2, sp
   10300:       e52d2004        push    {r2}            ; (str r2, [sp, #-4]!)
   10304:       e52d0004        push    {r0}            ; (str r0, [sp, #-4]!)
   10308:       e59fc010        ldr     ip, [pc, #16]   ; 10320 <_start+0x30>
   1030c:       e52dc004        push    {ip}            ; (str ip, [sp, #-4]!)
   10310:       e59f000c        ldr     r0, [pc, #12]   ; 10324 <_start+0x34>
   10314:       e59f300c        ldr     r3, [pc, #12]   ; 10328 <_start+0x38>
   10318:       ebffffeb        bl      102cc <__libc_start_main@plt>
   1031c:       ebfffff0        bl      102e4 <abort@plt>
   10320:       000104b4        .word   0x000104b4
   10324:       00010420        .word   0x00010420
   10328:       00010448        .word   0x00010448
```

很有趣，沒看到`main()`，反而看到`_start`。到底是`_start`是什麼呢？還記得Linker script嗎？裏面有一個`ENTRY`指令，可以指定程式從那邊開始跑，先來看一下預設的`ENTRY`是不是也是`_start`? 

```
$ ld --verbose | grep ENTRY
ENTRY(_start)
```
目前我們只知道執行檔起始點是`_start`，而不是`main`，那顯然有人幫你把執行檔加碼，以至於你的執行檔出現了`_start`。最偷懶的方式就是去找binary看看是不是有這樣的symbol。

```
user@host:/usr/lib$ find -name "*.[ao]" -exec nm -A {} \;  2> /dev/null | grep " _start$"
./arm-linux-gnueabi/crt1.o:00000000 T _start
./arm-linux-gnueabi/gcrt1.o:00000000 T _start
./arm-linux-gnueabi/Scrt1.o:00000000 T _start
./debug/usr/lib/arm-linux-gnueabi/crt1.o:00000000 T _start
./debug/usr/lib/arm-linux-gnueabi/gcrt1.o:00000000 T _start
./debug/usr/lib/arm-linux-gnueabi/Scrt1.o:00000000 T _start
```

OK，的確有object檔案裡面有`_start`，我們再來確認編譯的時候會不會link這些檔案。

```
$ gcc -v hello1.c 
Using built-in specs.
COLLECT_GCC=gcc
...
COLLECT_GCC_OPTIONS='-v' '-march=armv4t' '-mfloat-abi=soft' 
...
-X --hash-style=gnu -m armelf_linux_eabi
...
/usr/lib/gcc/arm-linux-gnueabi/4.9/../../../arm-linux-gnueabi/crt1.o 
...
```


而`_start`會呼叫外部函數`__libc_start_main`，我們透過`LD_DEBUG`來看一下。

```
$ LD_DEBUG=all ./hello1 2>&1 |grep __libc_start_main
       890:	symbol=__libc_start_main;  lookup in file=./hello1 [0]
       890:	symbol=__libc_start_main;  lookup in file=/lib/arm-linux-gnueabi/libc.so.6 [0]
       890:	binding file ./hello1 [0] to /lib/arm-linux-gnueabi/libc.so.6 [0]: normal symbol `__libc_start_main' [GLIBC_2.4]
```

可以看到，在./hello1中有去找`__libc_start_main`，最後去`libc.so.6`找，並且找出`libc.so.6`中`__libc_start_main`的位址(即binding)。而`__libc_start_main`的[prototype](http://refspecs.linuxbase.org/LSB_4.0.0/LSB-Core-generic/LSB-Core-generic/baselib---libc-start-main-.html)如下

```c
int __libc_start_main(int (*main) (int, char **, char **), int argc, char ** ubp_av, void (*init) (void), void (*fini) (void), void (*rtld_fini) (void), void (*stack_end));
```

看到有趣的東西嘛？我有看到

* main函數當作function pointer傳入
* main函數的參數
* 其他不知道三小的function pointer
	* init
  * fini
  * rtld_fini

從這邊我可以猜測這個函數就是呼叫一堆callback function，這些callback function就是上面列的死人骨頭。 

從[手冊](http://refspecs.linuxbase.org/LSB_4.0.0/LSB-Core-generic/LSB-Core-generic/baselib---libc-start-main-.html)的說明可以看到`__libc_start_main()`是用來執行環境的初始化、呼叫`main`函數並且傳遞參數，當`main`函數結束後處理回傳值。手冊提到的範例詳細行為有

* 檢查權限，確保安全性
* thread subsystem初始化 (我可不知道什麼thread subsystem唷)
* 向`rtld_fini`註冊release callback function，當shared object結束時使用該callback釋放資源
* 呼叫init callback function
* 呼叫main callback function並且帶入參數
* 當main callback function結束後，將回傳值作為參數呼叫exit


我們再回頭看看`_start`的組合語言：

```
000102f0 <_start>:
   102f0:       e3a0b000        mov     fp, #0
   102f4:       e3a0e000        mov     lr, #0
   102f8:       e49d1004        pop     {r1}            ; (ldr r1, [sp], #4)
   102fc:       e1a0200d        mov     r2, sp
   10300:       e52d2004        push    {r2}            ; (str r2, [sp, #-4]!)
   10304:       e52d0004        push    {r0}            ; (str r0, [sp, #-4]!)
   10308:       e59fc010        ldr     ip, [pc, #16]   ; 10320 <_start+0x30>
   1030c:       e52dc004        push    {ip}            ; (str ip, [sp, #-4]!)
   10310:       e59f000c        ldr     r0, [pc, #12]   ; 10324 <_start+0x34>
   10314:       e59f300c        ldr     r3, [pc, #12]   ; 10328 <_start+0x38>
   10318:       ebffffeb        bl      102cc <__libc_start_main@plt>
   1031c:       ebfffff0        bl      102e4 <abort@plt>
   10320:       000104b4        .word   0x000104b4
   10324:       00010420        .word   0x00010420
   10328:       00010448        .word   0x00010448
```

有趣的地方是這3個位址
```
   10320:       000104b4        .word   0x000104b4
   10324:       00010420        .word   0x00010420
   10328:       00010448        .word   0x00010448
```
從[這邊](#ful-asm)可以看到這3個位址分別是

* 10320:       000104b4        .word   0x000104b4
	* `__libc_csu_fini`
* 10324:       00010420        .word   0x00010420
  * `main`
* 10328:       00010448        .word   0x00010448
  * `__libc_csu_init`
  
也就是說，`main`和`__libc_csu_init`分別當作第一和第四參數傳給`__libc_start_main`，而`__libc_csu_fini`則被丟到stack，一樣傳給`__libc_start_main`了。

## 結論
Linux執行程式的起始點並不是`main`，而是glibc  binary中`crt1.o`準備的`_start`。這個start主要將你的`main`，還有一些hook函數丟給`__libc_start_main`，接下來libc的`__libc_start_main`樵好事情後才真正執行你的`main`，並且還要在`main`結束後清理戰場。

## 延伸閱讀

* Hook function
	* [GCCINT: 17.20.5 How Initialization Functions Are Handled](https://gcc.gnu.org/onlinedocs/gccint/Initialization.html#Initialization)
  * [Stackoverflow: How exactly does __attribute__((constructor)) work?](http://stackoverflow.com/questions/2053029/how-exactly-does-attribute-constructor-work)

## 參考資料

* [Linux Standard Base Core Specification 4.0: __libc_start_main](http://refspecs.linuxbase.org/LSB_4.0.0/LSB-Core-generic/LSB-Core-generic/baselib---libc-start-main-.html)
	* 不知道是不是過期，請小心服用
* [Computer Science from the Bottom Up: Chapter 8. Behind the process	(大推!)](http://bottomupcs.sourceforge.net/csbu/x3564.htm)
* [Mini FAQ about the misc libc/gcc crt files](http://dev.gentoo.org/~vapier/crt.txt)
* [allenlsy: C Essense (2) - C and Assembly](http://allenlsy.com/c-essense-2-assembly-and-c/)
	* 全系列都推！


<a name="ful_asm"></a>
## 完整反組譯程式碼
```asm Hello.dis
$ cat hello1.dis 

hello1:     file format elf32-littlearm


Disassembly of section .init:

0001029c <_init>:
   1029c:	e92d4008 	push	{r3, lr}
   102a0:	eb000021 	bl	1032c <call_weak_fn>
   102a4:	e8bd4008 	pop	{r3, lr}
   102a8:	e12fff1e 	bx	lr

Disassembly of section .plt:

000102ac <puts@plt-0x14>:
   102ac:	e52de004 	push	{lr}		; (str lr, [sp, #-4]!)
   102b0:	e59fe004 	ldr	lr, [pc, #4]	; 102bc <_init+0x20>
   102b4:	e08fe00e 	add	lr, pc, lr
   102b8:	e5bef008 	ldr	pc, [lr, #8]!
   102bc:	00010318 	.word	0x00010318

000102c0 <puts@plt>:
   102c0:	e28fc600 	add	ip, pc, #0, 12
   102c4:	e28cca10 	add	ip, ip, #16, 20	; 0x10000
   102c8:	e5bcf318 	ldr	pc, [ip, #792]!	; 0x318

000102cc <__libc_start_main@plt>:
   102cc:	e28fc600 	add	ip, pc, #0, 12
   102d0:	e28cca10 	add	ip, ip, #16, 20	; 0x10000
   102d4:	e5bcf310 	ldr	pc, [ip, #784]!	; 0x310

000102d8 <__gmon_start__@plt>:
   102d8:	e28fc600 	add	ip, pc, #0, 12
   102dc:	e28cca10 	add	ip, ip, #16, 20	; 0x10000
   102e0:	e5bcf308 	ldr	pc, [ip, #776]!	; 0x308

000102e4 <abort@plt>:
   102e4:	e28fc600 	add	ip, pc, #0, 12
   102e8:	e28cca10 	add	ip, ip, #16, 20	; 0x10000
   102ec:	e5bcf300 	ldr	pc, [ip, #768]!	; 0x300

Disassembly of section .text:

000102f0 <_start>:
   102f0:	e3a0b000 	mov	fp, #0
   102f4:	e3a0e000 	mov	lr, #0
   102f8:	e49d1004 	pop	{r1}		; (ldr r1, [sp], #4)
   102fc:	e1a0200d 	mov	r2, sp
   10300:	e52d2004 	push	{r2}		; (str r2, [sp, #-4]!)
   10304:	e52d0004 	push	{r0}		; (str r0, [sp, #-4]!)
   10308:	e59fc010 	ldr	ip, [pc, #16]	; 10320 <_start+0x30>
   1030c:	e52dc004 	push	{ip}		; (str ip, [sp, #-4]!)
   10310:	e59f000c 	ldr	r0, [pc, #12]	; 10324 <_start+0x34>
   10314:	e59f300c 	ldr	r3, [pc, #12]	; 10328 <_start+0x38>
   10318:	ebffffeb 	bl	102cc <__libc_start_main@plt>
   1031c:	ebfffff0 	bl	102e4 <abort@plt>
   10320:	000104b4 	.word	0x000104b4
   10324:	00010420 	.word	0x00010420
   10328:	00010448 	.word	0x00010448

0001032c <call_weak_fn>:
   1032c:	e59f3014 	ldr	r3, [pc, #20]	; 10348 <call_weak_fn+0x1c>
   10330:	e59f2014 	ldr	r2, [pc, #20]	; 1034c <call_weak_fn+0x20>
   10334:	e08f3003 	add	r3, pc, r3
   10338:	e7932002 	ldr	r2, [r3, r2]
   1033c:	e3520000 	cmp	r2, #0
   10340:	012fff1e 	bxeq	lr
   10344:	eaffffe3 	b	102d8 <__gmon_start__@plt>
   10348:	00010298 	.word	0x00010298
   1034c:	0000001c 	.word	0x0000001c

00010350 <deregister_tm_clones>:
   10350:	e59f301c 	ldr	r3, [pc, #28]	; 10374 <deregister_tm_clones+0x24>
   10354:	e59f001c 	ldr	r0, [pc, #28]	; 10378 <deregister_tm_clones+0x28>
   10358:	e0603003 	rsb	r3, r0, r3
   1035c:	e3530006 	cmp	r3, #6
   10360:	912fff1e 	bxls	lr
   10364:	e59f3010 	ldr	r3, [pc, #16]	; 1037c <deregister_tm_clones+0x2c>
   10368:	e3530000 	cmp	r3, #0
   1036c:	012fff1e 	bxeq	lr
   10370:	e12fff13 	bx	r3
   10374:	000205ff 	.word	0x000205ff
   10378:	000205fc 	.word	0x000205fc
   1037c:	00000000 	.word	0x00000000

00010380 <register_tm_clones>:
   10380:	e59f1024 	ldr	r1, [pc, #36]	; 103ac <register_tm_clones+0x2c>
   10384:	e59f0024 	ldr	r0, [pc, #36]	; 103b0 <register_tm_clones+0x30>
   10388:	e0601001 	rsb	r1, r0, r1
   1038c:	e1a01141 	asr	r1, r1, #2
   10390:	e0811fa1 	add	r1, r1, r1, lsr #31
   10394:	e1b010c1 	asrs	r1, r1, #1
   10398:	012fff1e 	bxeq	lr
   1039c:	e59f3010 	ldr	r3, [pc, #16]	; 103b4 <register_tm_clones+0x34>
   103a0:	e3530000 	cmp	r3, #0
   103a4:	012fff1e 	bxeq	lr
   103a8:	e12fff13 	bx	r3
   103ac:	000205fc 	.word	0x000205fc
   103b0:	000205fc 	.word	0x000205fc
   103b4:	00000000 	.word	0x00000000

000103b8 <__do_global_dtors_aux>:
   103b8:	e92d4010 	push	{r4, lr}
   103bc:	e59f401c 	ldr	r4, [pc, #28]	; 103e0 <__do_global_dtors_aux+0x28>
   103c0:	e5d43000 	ldrb	r3, [r4]
   103c4:	e3530000 	cmp	r3, #0
   103c8:	1a000002 	bne	103d8 <__do_global_dtors_aux+0x20>
   103cc:	ebffffdf 	bl	10350 <deregister_tm_clones>
   103d0:	e3a03001 	mov	r3, #1
   103d4:	e5c43000 	strb	r3, [r4]
   103d8:	e8bd4010 	pop	{r4, lr}
   103dc:	e12fff1e 	bx	lr
   103e0:	000205fc 	.word	0x000205fc

000103e4 <frame_dummy>:
   103e4:	e92d4008 	push	{r3, lr}
   103e8:	e59f0028 	ldr	r0, [pc, #40]	; 10418 <frame_dummy+0x34>
   103ec:	e5903000 	ldr	r3, [r0]
   103f0:	e3530000 	cmp	r3, #0
   103f4:	1a000001 	bne	10400 <frame_dummy+0x1c>
   103f8:	e8bd4008 	pop	{r3, lr}
   103fc:	eaffffdf 	b	10380 <register_tm_clones>
   10400:	e59f3014 	ldr	r3, [pc, #20]	; 1041c <frame_dummy+0x38>
   10404:	e3530000 	cmp	r3, #0
   10408:	0afffffa 	beq	103f8 <frame_dummy+0x14>
   1040c:	e1a0e00f 	mov	lr, pc
   10410:	e12fff13 	bx	r3
   10414:	eafffff7 	b	103f8 <frame_dummy+0x14>
   10418:	000204e8 	.word	0x000204e8
   1041c:	00000000 	.word	0x00000000

00010420 <main>:
   10420:	e92d4800 	push	{fp, lr}
   10424:	e28db004 	add	fp, sp, #4
   10428:	e59f0014 	ldr	r0, [pc, #20]	; 10444 <main+0x24>
   1042c:	ebffffa3 	bl	102c0 <puts@plt>
   10430:	e3a03000 	mov	r3, #0
   10434:	e1a00003 	mov	r0, r3
   10438:	e24bd004 	sub	sp, fp, #4
   1043c:	e8bd4800 	pop	{fp, lr}
   10440:	e12fff1e 	bx	lr
   10444:	000104c8 	.word	0x000104c8

00010448 <__libc_csu_init>:
   10448:	e92d43f8 	push	{r3, r4, r5, r6, r7, r8, r9, lr}
   1044c:	e59f6058 	ldr	r6, [pc, #88]	; 104ac <__libc_csu_init+0x64>
   10450:	e59f5058 	ldr	r5, [pc, #88]	; 104b0 <__libc_csu_init+0x68>
   10454:	e08f6006 	add	r6, pc, r6
   10458:	e08f5005 	add	r5, pc, r5
   1045c:	e0656006 	rsb	r6, r5, r6
   10460:	e1a07000 	mov	r7, r0
   10464:	e1a08001 	mov	r8, r1
   10468:	e1a09002 	mov	r9, r2
   1046c:	ebffff8a 	bl	1029c <_init>
   10470:	e1b06146 	asrs	r6, r6, #2
   10474:	0a00000a 	beq	104a4 <__libc_csu_init+0x5c>
   10478:	e2455004 	sub	r5, r5, #4
   1047c:	e3a04000 	mov	r4, #0
   10480:	e2844001 	add	r4, r4, #1
   10484:	e5b53004 	ldr	r3, [r5, #4]!
   10488:	e1a00007 	mov	r0, r7
   1048c:	e1a01008 	mov	r1, r8
   10490:	e1a02009 	mov	r2, r9
   10494:	e1a0e00f 	mov	lr, pc
   10498:	e12fff13 	bx	r3
   1049c:	e1540006 	cmp	r4, r6
   104a0:	1afffff6 	bne	10480 <__libc_csu_init+0x38>
   104a4:	e8bd43f8 	pop	{r3, r4, r5, r6, r7, r8, r9, lr}
   104a8:	e12fff1e 	bx	lr
   104ac:	00010088 	.word	0x00010088
   104b0:	00010080 	.word	0x00010080

000104b4 <__libc_csu_fini>:
   104b4:	e12fff1e 	bx	lr

Disassembly of section .fini:

000104b8 <_fini>:
   104b8:	e92d4008 	push	{r3, lr}
   104bc:	e8bd4008 	pop	{r3, lr}
   104c0:	e12fff1e 	bx	lr

```
