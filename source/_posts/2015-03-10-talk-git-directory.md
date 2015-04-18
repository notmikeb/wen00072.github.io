---
layout: post
title: '談談.git  目錄'
date: 2015-03-10 23:21
comments: true
categories: git 
---
如果有使用git的朋友，可能會知道git是一個分散式版本控制系統。這表示repository會放一份在本機上面。仔細或是有看書的人，會知道這些repository沒有意外會放在工作目錄的`.git`下面。今天無聊來這個目錄戳戳看。

因為愈寫愈多，還是弄一下目錄好了

* [測試環境](#g_env)
* [背景知識](#g_bkg)
* [.git目錄列表](#g_list)
* [.git第一層檔案](#g_1st_file)
  * [index](#gidx)
  * [packed-refs](#gpkref)
  * [config](#gconf)
  * [ORIG_HEAD](#gorg_head)
  * [description](#gdesc)
  * [HEAD](#ghead)
* [.git第一層目錄](#g_1st_dir)
	* [branches](#g_1st_dir_br)
	* [hooks](#g_1st_dir_hk)
	* [info](#g_1st_dir_inf)
	* [logs](#g_1st_dir_log)
	* [objects](#g_1st_dir_obj)
	* [refs](#g_1st_dir_ref)
* [參考資料](#g_ref)


<a name="g_env"></a>
## 測試環境
```
$ lsb_release -a
No LSB modules are available.
Distributor ID:	Ubuntu
Description:	Ubuntu 14.04.2 LTS
Release:	14.04
Codename:	trusty

$ git --version
git version 2.3.0

$ git remote show 
origin
$ git remote show origin 
* remote origin
  Fetch URL: https://github.com/pcman-bbs/pcmanx.git
  Push  URL: https://github.com/pcman-bbs/pcmanx.git
  HEAD branch: master
  Remote branches:
    gtk3                          tracked
    master                        tracked
    next-release                  tracked
    show_web_search_in_popup_menu tracked
    transparent_background        tracked
  Local branch configured for 'git pull':
    master merges with remote master
  Local ref configured for 'git push':
    master pushes to master (up to date)
```


<a name="g_list"></a>
## .git目錄列表
```
/tmp/pcmanx$ ls -gG --group-directories-first .git
total 56
drwxrwxr-x 2  4096 Mar  3 23:36 branches
drwxrwxr-x 2  4096 Mar  3 23:36 hooks
drwxrwxr-x 2  4096 Mar  3 23:36 info
drwxrwxr-x 3  4096 Mar  3 23:37 logs
drwxrwxr-x 4  4096 Mar  3 23:36 objects
drwxrwxr-x 5  4096 Mar  3 23:37 refs
-rw-rw-r-- 1   264 Mar  3 23:37 config
-rw-rw-r-- 1    73 Mar  3 23:36 description
-rw-rw-r-- 1    23 Mar  3 23:37 HEAD
-rw-rw-r-- 1 12256 Mar  4 23:11 index
-rw-rw-r-- 1    41 Mar  4 23:11 ORIG_HEAD
-rw-rw-r-- 1   829 Mar  3 23:37 packed-refs
```



<a name="g_bkg"></a>
## 背景知識
這邊只列出需要的背景知識，可能有錯。

* Git 的基本上資料有分blob, tree, commit, tag這四種型態
* Git 的branch可以視為放一個檔案，這個檔案存放某個commit的HASH資訊

多說無益，直接操作，先來看這四種型態是三小
```
$ git log
commit fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304
...
```


來看一下 fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304裏面是啥
```
$ git cat-file -p fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304
tree 2f6db7ea7db6b66cb4c07a98e78b580f67872db6
parent 3fb419f40c10d67bd1b9b18152fb43cca58ca411
...
```

可以看到有放

* tree型態的HASH
* 上一個commit的HASH
* commit的訊息

接下來來看tree吧

```
$ git cat-file -p 2f6db7ea7db6b66cb4c07a98e78b580f67872db6
...
100644 blob 5bc4b3f69d730535eb3a8aee28d7ef9273225d50	.gitignore
...
040000 tree b0b46378d75b1737257fef9d9eb433489b2d0ea0	build
```
有沒有和Linux的[File system](http://en.wikipedia.org/wiki/Ext2#Directories)很像？~~目錄~~tree資料放的是~~inode~~HASH值和檔案名稱的對應表。

我們可以看到HASH的型態有tree和blob，理所當然來看看blob吧。

```
$ git cat-file -p 5bc4b3f69d730535eb3a8aee28d7ef9273225d50
## autotools generated
Doxyfile
INSTALL
Makefile
Makefile.in
...
```

很明顯這是一個資料存放的地方。

然後我們來看branch是不是真的放在檔案，檔案內容是某個commit HASH吧，直接看範例。

```
$ git checkout -b br1
Switched to a new branch 'br1'
$ git checkout -b br2
Switched to a new branch 'br2'
$ tree .git/refs/
.git/refs/
├── heads
│   ├── br1
│   ├── br2
│   └── master
├── remotes
│   └── origin
│       └── HEAD
└── tags

$ git log
commit fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304
...
$ cat .git/refs/heads/br1 
fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304
$ cat .git/refs/heads/br2
fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304
$ cat .git/refs/heads/master 
fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304
```


<a name="g_1st_file"></a>
## .git第一層檔案
用file看一下，可以看到大部分都是純文字檔：

```
/tmp/pcmanx$ find .git -maxdepth 1 -type f -exec file {} \;
.git/index: Git index, version 2, 144 entries
.git/packed-refs: ASCII text
.git/config: ASCII text
.git/ORIG_HEAD: ASCII text
.git/description: ASCII text
.git/HEAD: ASCII text
```

接下來分別討論

* [index](#gidx)
* [packed-refs](#gpkref)
* [config](#gconf)
* [ORIG_HEAD](#gorg_head)
* [description](#gdesc)
* [HEAD](#ghead)


<a name="gidx"></a>
## index
先看下面的操作。

```
$ file .git/index 
.git/index: Git index, version 2, 144 entries

$ find -type f | grep -v .git\/ | cut -c 3- | sort | wc -l
144

$ git ls-files | wc -l
144
```
有興趣的人可以把`| wc -l`去掉玩看看，基本上這個是在描述repository的目錄結構。也就是說，哪個檔案要放在那個目錄。更精確的來說，協助那個blob hash要放在哪個tree中。根據[這邊](http://schacon.github.io/gitbook/7_the_git_index.html)的說法，這個檔案可以協助
* 提供產生tree object時需要的資料
* 提供比對working directory和tree的資訊
* merge產生衝突時，可以從index取得的資料協助解決衝突。不過這邊不是很懂，就單純字面翻譯了。


<a name="gpkref"></a>
## packed-refs
簡單來說，考量效率，git提供pack ref目錄的功能，ref的概念請看前面背景知識。

一樣，看例子比較快。

先用`git show-ref`看看我們有哪些reference
```
$ git show-ref
fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304 refs/heads/master
fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304 refs/remotes/origin/HEAD
c86b440bafe24caecfe0dbfbbd73031a1bcd1178 refs/remotes/origin/gtk3
fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304 refs/remotes/origin/master
22da092eb00b78946fd70de24427d702a77b925b refs/remotes/origin/next-release
98ff18f84fad4686012bf6183734eeb1ec5d7f46 refs/remotes/origin/show_web_search_in_popup_menu
9542889d17d276321d6da2de8d1f9d95f76fb483 refs/remotes/origin/transparent_background
7a3629ab3da7c609cf3698319ab606ebae0998e7 refs/tags/0.3.2@221
f57350c72987263c2b745690bdc8013fbbd6067c refs/tags/0.3.3@243
522d1dc2568b5c7c256e05ca69412480e1afcfb3 refs/tags/0.3.4
8e9d9520125f64a0b894a5d575f0c2d98c3ec06b refs/tags/0.3.4@301
7031115a8ea1151efc1536b43a509774ca2c0777 refs/tags/0.3.7
3f4dcbe6e1aa0045fbeb9f1f761ce035e65defea refs/tags/1.1
098d158c8d8a7ce7c6b2c5c47c967c6137beced2 refs/tags/1.2
```

看看`packed-refs`裏面是三小？有沒有發現和上面很像啊？
```
$ cat .git/packed-refs 
## pack-refs with: peeled fully-peeled 
c86b440bafe24caecfe0dbfbbd73031a1bcd1178 refs/remotes/origin/gtk3
fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304 refs/remotes/origin/master
22da092eb00b78946fd70de24427d702a77b925b refs/remotes/origin/next-release
98ff18f84fad4686012bf6183734eeb1ec5d7f46 refs/remotes/origin/show_web_search_in_popup_menu
9542889d17d276321d6da2de8d1f9d95f76fb483 refs/remotes/origin/transparent_background
7a3629ab3da7c609cf3698319ab606ebae0998e7 refs/tags/0.3.2@221
f57350c72987263c2b745690bdc8013fbbd6067c refs/tags/0.3.3@243
522d1dc2568b5c7c256e05ca69412480e1afcfb3 refs/tags/0.3.4
8e9d9520125f64a0b894a5d575f0c2d98c3ec06b refs/tags/0.3.4@301
7031115a8ea1151efc1536b43a509774ca2c0777 refs/tags/0.3.7
3f4dcbe6e1aa0045fbeb9f1f761ce035e65defea refs/tags/1.1
098d158c8d8a7ce7c6b2c5c47c967c6137beced2 refs/tags/1.2
```

看一下ref目錄，上面的那些reference並不存在在`.git/ref`裏面
```
$ tree .git/refs/
.git/refs/
├── heads
│   └── master
├── remotes
│   └── origin
│       └── HEAD
└── tags

4 directories, 2 files
```

好，生一個branch看看。
```
$ git checkout -b I_got_a_new_ref
Switched to a new branch 'I_got_a_new_ref'
```

喔喔！`.git/refs`出現了剛才產生的ref
```
$ tree .git/refs/
.git/refs/
├── heads
│   ├── I_got_a_new_ref
│   └── master
├── remotes
│   └── origin
│       └── HEAD
└── tags

4 directories, 3 files
```

跑一下`git gc`整理一下
```
$ git gc
Counting objects: 5002, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (1068/1068), done.
Writing objects: 100% (5002/5002), done.
Total 5002 (delta 3913), reused 5002 (delta 3913)
```

疑？剛才的branch不見了。
```
$ tree .git/refs/
.git/refs/
├── heads
├── remotes
│   └── origin
│       └── HEAD
└── tags

4 directories, 1 file
```

跑去pack-ref了
```
$ cat .git/packed-refs 
## pack-refs with: peeled fully-peeled 
fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304 refs/heads/I_got_a_new_ref
fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304 refs/heads/master
c86b440bafe24caecfe0dbfbbd73031a1bcd1178 refs/remotes/origin/gtk3
fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304 refs/remotes/origin/master
22da092eb00b78946fd70de24427d702a77b925b refs/remotes/origin/next-release
98ff18f84fad4686012bf6183734eeb1ec5d7f46 refs/remotes/origin/show_web_search_in_popup_menu
9542889d17d276321d6da2de8d1f9d95f76fb483 refs/remotes/origin/transparent_background
7a3629ab3da7c609cf3698319ab606ebae0998e7 refs/tags/0.3.2@221
f57350c72987263c2b745690bdc8013fbbd6067c refs/tags/0.3.3@243
522d1dc2568b5c7c256e05ca69412480e1afcfb3 refs/tags/0.3.4
8e9d9520125f64a0b894a5d575f0c2d98c3ec06b refs/tags/0.3.4@301
7031115a8ea1151efc1536b43a509774ca2c0777 refs/tags/0.3.7
3f4dcbe6e1aa0045fbeb9f1f761ce035e65defea refs/tags/1.1
098d158c8d8a7ce7c6b2c5c47c967c6137beced2 refs/tags/1.2
```


<a name="gconf"></a>
## config
其實這個是個大哉問，就值得專門開一個章節討論。所以我整理到[這個頁面](http://wen00072-blog.logdown.com/posts/256871-introduction-to-git-config)，請自行前往閱讀。


<a name="gorg_head"></a>
## ORIG_HEAD
這個檔案很有趣，一開始其實不存在。後來不知道怎麼突然跑出來，問了估狗和男人(man git reset)後得到的答案就是當reset的時候會把舊的HEAD hash放到這邊。為什麼要這樣做呢，當然是反悔用的。你可以`man git commit`找`ORIG_HEAD`就可以看看`git commit --amend`同效果的範例了。


<a name="gdesc"></a>
## description
懶得解釋，跳過


<a name="ghead"></a>
## HEAD
望文生義，當然是指向目前所在的commit。

等等！

指向目前所在的commit是三小？
一樣，來看例子。

目前clone下來，裏面放的是一個檔案，檔案裏面是什麼呢？一樣是個commit HASH
```
$ cat .git/HEAD 
ref: refs/heads/master

$ cat .git/refs/heads/master
fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304

$ git cat-file -t fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304
commit
```

那我直接checkout一個不是branch的可以嘛？當然沒問題。
可以看到git會把你checkout的commit HASH寫入.git/HEAD

```
$ strace git checkout 93f0addca5b2723ff0f1b744b032f359
...
open("/tmp/pcmanx/.git/HEAD.lock", O_RDWR|O_CREAT|O_EXCL, 0666) = 3
write(3, "93f0addca5b2723ff0f1b744b032f359"..., 40) = 40
write(3, "\n", 1)                       = 1
close(3)                                = 0
...
rename("/tmp/pcmanx/.git/HEAD.lock", "/tmp/pcmanx/.git/HEAD") = 0
...

$ cat .git/HEAD 
93f0addca5b2723ff0f1b744b032f359938a286d
```


<a name="g_1st_dir"></a>
## 第一層目錄

* [branches](#g_1st_dir_br)
* [hooks](#g_1st_dir_hk)
* [info](#g_1st_dir_inf)
* [logs](#g_1st_dir_log)
* [objects](#g_1st_dir_obj)
* [refs](#g_1st_dir_ref)


<a name="g_1st_dir_br"></a>
## branches
快過期的東西([出處](http://git-scm.com/docs/gitrepository-layout))，跳過。


<a name="g_1st_dir_hk"></a>
## hooks
hook，這個翻成中文很鳥，保留原文。基本上就是一組callback，特定的時候會去呼叫。我目前生意沒有做很大，有需要再去看。列出預設的檔案如下

```
$ ll .git/hooks/
total 48
drwxrwxr-x 2 wen wen 4096 Mar  9 23:10 ./
drwxrwxr-x 8 wen wen 4096 Mar  9 23:21 ../
-rwxrwxr-x 1 wen wen  452 Mar  9 23:10 applypatch-msg.sample*
-rwxrwxr-x 1 wen wen  896 Mar  9 23:10 commit-msg.sample*
-rwxrwxr-x 1 wen wen  189 Mar  9 23:10 post-update.sample*
-rwxrwxr-x 1 wen wen  398 Mar  9 23:10 pre-applypatch.sample*
-rwxrwxr-x 1 wen wen 1642 Mar  9 23:10 pre-commit.sample*
-rwxrwxr-x 1 wen wen 1239 Mar  9 23:10 prepare-commit-msg.sample*
-rwxrwxr-x 1 wen wen 1348 Mar  9 23:10 pre-push.sample*
-rwxrwxr-x 1 wen wen 4898 Mar  9 23:10 pre-rebase.sample*
-rwxrwxr-x 1 wen wen 3611 Mar  9 23:10 update.sample*
```


<a name="g_1st_dir_inf"></a>
## info
儲存資訊，廢話。目前裏面只有一個exclude，而且檔案內容還都是註解。但是[這邊](http://git-scm.com/docs/gitrepository-layout)有說還有其他檔案。另外值得提的是exclude說明有提到`.gitignore`針對的是`git status`, `git rm`, `git add`, `git clean`。其他的操作還是得在`.git/log/exclude`中設定。


<a name="g_1st_dir_log"></a>
## logs
這是存放log的地方。log是什麼呢？直接看個例子。

我們先新增一個測試檔案
```
$ touch test2
$ git add test2
$ git commit test2 -m "test2"
[master 09580e4] test2
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 test2
```

執行`git reflog`，可以看到你剛才的動作以及有當次HEAD的commit HASH已經被記起來。
```
$ git reflog 
09580e4 HEAD@{0}: commit: test2
fd5d6ab HEAD@{1}: checkout: moving from 93f0addca5b2723ff0f1b744b032f359938a286d to master
93f0add HEAD@{2}: checkout: moving from master to 93f0addca5b2723ff0f1b744b032f359938a286d
...
```

這東西有什麼用處呢？當然是讓你反悔用的。再來看個例子

首先開一個branch
```
$ git checkout -b demo_reflog
Switched to a new branch 'demo_reflog'
```

新增並commit file1
```
$ touch file1
$ git add file1
$ git commit file1 -m "file 1"
[demo_reflog c982c70] file 1
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 file1
```

然後新增並commit file2
```
$ touch file2
$ git add file2
$ git commit file2 -m "file 2"
[demo_reflog b85f4d4] file 2
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 file2
```

切回master並且宰掉剛才的branch
```
$ git checkout master 
Switched to branch 'master'
Your branch is up-to-date with 'origin/master'.

$ git branch -D demo_reflog  
Deleted branch demo_reflog (was b85f4d4).
```

假設上一個動作是手滑誤刪，能不能反悔。
YES YOU CAN!

先看看log
```
$ git reflog 
fd5d6ab HEAD@{0}: checkout: moving from demo_reflog to master
b85f4d4 HEAD@{1}: commit: file 2
c982c70 HEAD@{2}: commit: file 1
...
```

可以知道上次branch最後commit的HASH，那麼我們就可以切過去然後開另外一個branch了。
```
$ git checkout -b i_am_back b85f4d4
Switched to a new branch 'i_am_back'
```

可以看到資料回來了。
```
$ git log --pretty=oneline --abbrev-commit
b85f4d4 file 2
c982c70 file 1
fd5d6ab Fix invisible cairo caret issue.
...
```

當然這還是險招，git本身保留是有保存期限的。有興趣問男人。`man git gc` 

不知道有沒有會問：「啊不是要介紹`.git/logs`嘛？」

問得好！猜猜剛才git reflogs檔案從哪裡讀出來？

一樣請出strace大大
```
$ strace -e open git reflog 
...
open("/tmp/pcmanx/.git/logs/HEAD", O_RDONLY) = 3
...
```

有看到`.git/logs/HEAD`吧？有沒有想看一下裏面是啥？至少我想看。

```
$ cat .git/logs/HEAD
...
fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304 c982c70f00875b83e4845ca35ef0ca4c70f73e64 Wen.Liao <censored> 1425997414 +0800	commit: file 1
c982c70f00875b83e4845ca35ef0ca4c70f73e64 b85f4d4dca4012a2d1785750061fae9e03e817e1 Wen.Liao <censored> 1425997438 +0800	commit: file 2
b85f4d4dca4012a2d1785750061fae9e03e817e1 fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304 Wen.Liao <censored> 1425997673 +0800	checkout: moving from demo_reflog to master
fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304 b85f4d4dca4012a2d1785750061fae9e03e817e1 Wen.Liao <censored> 1425997732 +0800	checkout: moving from master to b85f4d4
b85f4d4dca4012a2d1785750061fae9e03e817e1 b85f4d4dca4012a2d1785750061fae9e03e817e1 Wen.Liao <censored> 1425997746 +0800	checkout: moving from b85f4d4dca4012a2d1785750061fae9e03e817e1 to i_am_back
```

精確的來說，`.git/logs`下面不只紀錄HEAD的log，還有各種ref的log。有興趣的朋友可以自行參考[這邊](http://git-scm.com/docs/gitrepository-layout)，並且進去目錄逛逛。


<a name="g_1st_dir_obj"></a>
## objects
先看剛clone下來的目錄結構
```
$ tree .git/objects/
.git/objects/
├── info
└── pack
    ├── pack-6042320de285747900cc3263a47f7bab429cabf8.idx
    └── pack-6042320de285747900cc3263a47f7bab429cabf8.pack
```

來生一個目錄和並且commit一個東西進去。
```
$ mkdir tree
$ echo test > tree/file
$ git add tree/file 
$ git commit tree/file -m "test tree and file"
[master 2ba01ef] test tree and file
 1 file changed, 1 insertion(+)
 create mode 100644 tree/file
```

然後看看這個目錄變化吧
```
$ tree .git/objects/
.git/objects/
├── 2b
│   └── a01ef5583f35dd57b5fb5bde732f73bc315b6d
├── 9d
│   └── aeafb9864cf43055ae93beb0afd6c7d144bfa4
├── de
│   └── f69c2c316574aaf328e546486fa750eb9c53a0
├── e1
│   └── b8ecbb1f19709f3a4867a0ffe08bb2e07acf19
├── info
└── pack
    ├── pack-6042320de285747900cc3263a47f7bab429cabf8.idx
    └── pack-6042320de285747900cc3263a47f7bab429cabf8.pack

6 directories, 6 files
```

接下來把目錄和檔案混起來，用`git cat-file -p`看看裏面是啥
```
$ git cat-file -p 2ba01ef5583f35dd57b5fb5bde732f73bc315b6d
tree def69c2c316574aaf328e546486fa750eb9c53a0
parent fd5d6abe34f41f8687c1fc5e2ab0a2f65c570304
author Wen.Liao <censored> 1425999641 +0800
committer Wen.Liao <censored> 1425999641 +0800

test tree and file
```
```
$ git cat-file -p 9daeafb9864cf43055ae93beb0afd6c7d144bfa4
test
```
```
$ git cat-file -p def69c2c316574aaf328e546486fa750eb9c53a0
100644 blob 5bc4b3f69d730535eb3a8aee28d7ef9273225d50	.gitignore
...
040000 tree e1b8ecbb1f19709f3a4867a0ffe08bb2e07acf19	tree
```
```
$ git cat-file -p e1b8ecbb1f19709f3a4867a0ffe08bb2e07acf19
100644 blob 9daeafb9864cf43055ae93beb0afd6c7d144bfa4	file
```

其實不難猜，你local commit的東西會被處理後變成object，並且依照hash分門別類地存放。

接下來我們來做個測試
```
$ git gc
Counting objects: 5006, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (1070/1070), done.
Writing objects: 100% (5006/5006), done.
Total 5006 (delta 3914), reused 5001 (delta 3913)
```

可以看到，剛才的目錄全部消失，而pack目錄的檔案名稱也和前面的不同了。對照上面`git gc`的訊息，可以知道`git gc`做了壓縮，並且將這些object一起壓到pack目錄的檔案中了。
```
$ tree .git/objects/
.git/objects/
├── info
│   └── packs
└── pack
    ├── pack-57d163604c430e1919a18e97c5b1312291b62721.idx
    └── pack-57d163604c430e1919a18e97c5b1312291b62721.pack

2 directories, 3 files
```

所以我們可以從觀察中做出結論。Git object的存放有兩種型式

* loose object：就是剛才看到用[HASH前兩碼]/[剩下HASH] 的目錄檔案結構
* packed object: 把object壓縮成兩個檔案

而looose object型式經過`git gc`後會有可能轉換成packed object，反之亦然。詳細規則問男人。
由於`git gc`是用來**清除**和**整理**local repository，也就是說做了`git gc`是有可能刪除用不到的資料。詳細情況一樣請問男人`man git gc`


<a name="g_1st_dir_ref"></a>
## refs

* [背景知識](#g_bkg)和[packed-ref](#gpkref)有提到，就不要拖台錢了。


<a name="g_ref"></a>
## 參考資料

* [Stackoverflow: What does the git index contain EXACTLY?](http://stackoverflow.com/questions/4084921/what-does-the-git-index-contain-exactly)
* [Git Community Book: The Git Index](http://schacon.github.io/gitbook/7_the_git_index.html)
* [Stackoverflow: HEAD and ORIG_HEAD in Git](http://stackoverflow.com/questions/964876/head-and-orig-head-in-git)
* [ProGit: 10.2 Git Internals - Git Objects](http://git-scm.com/book/en/v2/Git-Internals-Git-Objects)
* [what's inside your .git directory](http://gitready.com/advanced/2009/03/23/whats-inside-your-git-directory.html)
* [git-pack-refs - Pack heads and tags for efficient repository access](http://git-scm.com/docs/git-pack-refs)
* [ProGit: 8.1 Customizing Git - Git Configuration](http://git-scm.com/book/en/v2/Customizing-Git-Git-Configuration#_git_config)
* [gitrepository-layout - Git Repository Layout](http://git-scm.com/docs/gitrepository-layout)
* [Git Community Book: How Git Stores Objects](http://schacon.github.io/gitbook/7_how_git_stores_objects.html)
