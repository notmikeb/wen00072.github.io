---
layout: post
title: 'GNU ld初探'
date: 2014-03-14 10:14
comments: true
categories: 
---
以前一直以為ld單純就是把*.a, *.o轉成binary，簡單測試一下發現完全不是這樣。

測試的檔案除了Makefile以外，其他的和[這邊](http://wen00072-blog.logdown.com/posts/184830-makefile-automatically-assign-a-file-to-a-specific-directory)一樣

將原本的Makefile部份

```makefile 原本的Makefile準備更動的部份
$(OUT_DIR)/$(TARGET): $(OUT_OBJS)
    $(CC) -o $@ $^
```

改成
```makefile $(CC)改成ld
$(OUT_DIR)/$(TARGET): $(OUT_OBJS)
    ld -o $@ $^
```

一跑make就出現錯誤
```text make log
$ make
mkdir -p build/libs/
cc -g -MMD -I ./include -c libs/liba.c -o build/libs/liba.o
mkdir -p build/libs/
cc -g -MMD -I ./include -c libs/libb.c -o build/libs/libb.o
mkdir -p build/
cc -g -MMD -I ./include -c test.c -o build/test.o
ld -o build/test build/libs/liba.o build/libs/libb.o build/test.o 
ld: warning: cannot find entry symbol _start; defaulting to 00000000004000b0
build/libs/liba.o: In function `from_liba':
./libs/liba.c:11: undefined reference to `puts'
build/libs/libb.o: In function `from_libb':
./libs/libb.c:11: undefined reference to `puts'
build/test.o: In function `main':
./test.c:10: undefined reference to `malloc'
make: *** [build/test] Error 1
```

ld 這邊再加上-lc又有其他的錯誤，看來的確是有東西隱藏在背面。因此需要有對照組，這時候`gcc -v`就可以出場了

```text gcc -v log節錄
$ gcc -v -o build/test build/libs/liba.o build/libs/libb.o build/test.o 
Using built-in specs.
...
 /usr/lib/gcc/x86_64-linux-gnu/4.6/collect2 --sysroot=/ --build-id --no-add-needed --as-needed --eh-frame-hdr -m elf_x86_64 --hash-style=gnu -dynamic-linker /lib64/ld-linux-x86-64.so.2 -z relro -o build/test /usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu/crt1.o /usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu/crti.o /usr/lib/gcc/x86_64-linux-gnu/4.6/crtbegin.o -L/usr/lib/gcc/x86_64-linux-gnu/4.6 -L/usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu -L/usr/lib/gcc/x86_64-linux-gnu/4.6/../../../../lib -L/lib/x86_64-linux-gnu -L/lib/../lib -L/usr/lib/x86_64-linux-gnu -L/usr/lib/../lib -L/usr/lib/gcc/x86_64-linux-gnu/4.6/../../.. build/libs/liba.o build/libs/libb.o build/test.o -lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed /usr/lib/gcc/x86_64-linux-gnu/4.6/crtend.o /usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu/crtn.o
```

可以看到gcc呼叫[collect2](http://www.slideshare.net/jserv/from-source-to-binary-how-gnu-toolchain-works/12)，而collect2會呼叫ld

```text strace collect2結果節錄
$ strace -e execve -f gcc -o build/test build/libs/liba.o build/libs/libb.o build/test.o 
execve("/usr/bin/gcc", ["gcc", "-o", "build/test",...
...
execve("/usr/lib/gcc/x86_64-linux-gnu/4.6/collect2",..., "--sysroot=/",...
...
execve("/usr/bin/ld", ["/usr/bin/ld", "--sysroot=/", ...
```

最後的Makefile版本變成
```makefile ld最後參數
# LD_FLAGS
LD_FLAGS= \
	--sysroot=/ \
	--build-id \
	--no-add-needed --as-needed --eh-frame-hdr \
	-m elf_x86_64 --hash-style=gnu \
	-dynamic-linker /lib64/ld-linux-x86-64.so.2 \
	-z relro /usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu/crt1.o\
	/usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu/crti.o \
	/usr/lib/gcc/x86_64-linux-gnu/4.6/crtbegin.o \
	-L/usr/lib/gcc/x86_64-linux-gnu/4.6 \
	-L/usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu \
	-L/usr/lib/gcc/x86_64-linux-gnu/4.6/../../../../lib \
	-L/lib/x86_64-linux-gnu \
	-L/lib/../lib -L/usr/lib/x86_64-linux-gnu \
	-L/usr/lib/../lib -L/usr/lib/gcc/x86_64-linux-gnu/4.6/../../..\
	-lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s \
	--no-as-needed \
	/usr/lib/gcc/x86_64-linux-gnu/4.6/crtend.o \
	/usr/lib/gcc/x86_64-linux-gnu/4.6/../../../x86_64-linux-gnu/crtn.o

# Build Rules
TARGET=test

$(OUT_DIR)/$(TARGET): $(OUT_OBJS)
	ld -o $@ $^ $(LD_FLAGS)
```

結論如下

1. gcc在build code默默處理掉很多細節
2. gcc -v可以協助顯示更多編譯細節

## 參考資料: 
[from Source to Binary: How GNU Toolchain Works](http://www.slideshare.net/jserv/from-source-to-binary-how-gnu-toolchain-works)
