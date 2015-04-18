---
layout: post
title: 'Linux 檔案的hole'
date: 2013-12-31 06:45
comments: true
categories: [C, Linux, hole, lseek]
---
## 大綱
檔案的hole檔案中連續的null 字元。

當lseek指定的offset (假設叫o_lseek)超過目前檔案最後的offset(o_end)時，多出來的delta系統並不會寫入到實體空件，而是有人要讀取這段空間時會讀到連續的null字元。也就是說會有檔案大小遠大於實際佔用的儲存空間的特性。

一般來說，hole會應在sparse的檔案，最典型的的例子就是Virtual machine的image檔案。

- [範例](#範例)
    - [執行結果](#執行結果)
    - [Makefile](#Makefile)
    - [程式](#程式)
- [參考資料](#參考資料)

<a name="範例"></a>
## 範例

測試程式行為描述

1. 讀入任意字串
2. 將字串寫入檔案，每個字元間隔1G的空間

<a name="執行結果"></a>

## 執行結果

請注意檔案大小和目錄大小的差異

```text result
$ ./create_hole_file 
Enter any text: test111
Get test111
, used to create test hole file

$ ls -lh file_in_the_hole 
-rw------- 1 nobody nobody 7.1G Dec 31 14:24 file_in_the_hole

$ du -sh .
128K	.

```

<a name="Makefile"></a>
## Makefile
```makefile Makefile
CFLAGS=-g -Wall -pedantic
LDFLAGS=

TARGET=create_hole_file
HOLE_FILE_NAME=file_in_the_hole

all: $(TARGET)
create_hole_file: create_hole_file.c
	gcc -o $@ $(CFLAGS) $^ -DHOLE_FILE_NAME="\"$(HOLE_FILE_NAME)\""

clean:
	rm -rf *.o *~ $(TARGET) $(HOLE_FILE_NAME)
```

<a name="程式"></a>
## 程式

```c create_hole_file.c
#include <stdio.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>
#include <unistd.h>

#ifndef HOLE_FILE_NAME
#error You need to define HOLE_FILE_NAME
#endif

#define MAX_CHAR_LINE  (256)
#define HOLE_GAP       (1024*1024*1024) /* 1G */
int main(void)
{
    char line[MAX_CHAR_LINE] = {0};
    int rval = 0;
    int fd   = 0;
    int i    = 0;

    off_t   lseek_offset = 0;
    ssize_t write_size   = 0;

    /* Open file */
    fd = open(HOLE_FILE_NAME, O_CREAT | O_WRONLY, S_IRUSR  | S_IWUSR);
    if (fd == -1) {
        fprintf(stderr, "%d: %s\n", __LINE__, strerror(errno));
        return -1;
    }

    /* Get string from stdin */
    fprintf(stderr, "Enter any text: ");
    fgets(line, MAX_CHAR_LINE, stdin);

    fprintf(stderr, "Get %s, used to create test hole file\n", line);

    /* Implent hole */
    for (i = 0; i < strlen(line); i++) {
        write_size = write(fd, &line[i], sizeof(char));
        if (write_size == -1) {
            fprintf(stderr, "%d: %s\n", __LINE__, strerror(errno));
            return -1;
        }

        /* file in the hole! */
        lseek_offset = lseek(fd, HOLE_GAP, SEEK_CUR);
        if (lseek_offset == -1) {
            fprintf(stderr, "%d: %s\n", __LINE__, strerror(errno));
            return -1;
        }

    }

    /* Close */
    rval = close(fd);
    if (rval == -1) {
        fprintf(stderr, "%d: %s\n", __LINE__, strerror(errno));
        return -1;
    }
    return 0;
}
```

<a name="參考資料"></a>
## 參考資料

- man lseek
- The Linux Programming Interface: Chapter 4
