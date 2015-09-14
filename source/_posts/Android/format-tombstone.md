title: 封装addr2line，从tombstone中解析出调用栈
date: 2015-09-14 21:12:30
tags: [android, debug]
---

本文介绍如何简单地封装addrline，从而能够从android native程序crash的墓碑日志中，解析出程序crash的调用栈。（如同Java程序crash一样一眼可以看出来）

## 例子

首先看一个例子,有如下 tombstone 日志:
```
*** *** *** *** *** *** *** *** *** *** *** *** *** *** *** ***
Build fingerprint: 'xxxxxx'
Revision: '0'
pid: 6936, tid: 6996, name: Binder_1 >>> /system/bin/mediaserver <<<
signal 11 (SIGSEGV), code 1 (SEGV_MAPERR), fault addr 00000024
r0 00000024 r1 00000004 r2 00000000 r3 00000001
r4 00000024 r5 00000000 r6 00000000 r7 41fa0010
r8 40dba7dc r9 00100000 sl 41fa0010 fp 00000016
ip 40dcbed8 sp 423c3e88 lr 40db9fb5 pc 40158598 cpsr 20000010
d0 000000000000001c d1 00000000000000f1
d2 00000008000003e2 d3 40e2f11400000040
d4 004600490044004f d5 00550041005f0059
d6 005f004f00490044 d7 0054005400450053
d8 0000000000000000 d9 0000000000000000
d10 0000000000000000 d11 0000000000000000
d12 0000000000000000 d13 0000000000000000
d14 0000000000000000 d15 0000000000000000
d16 3f801272163691fc d17 3ef5276650448ea9
d18 4000000000000000 d19 bf66c16b5f3aa8cf
d20 3fc55554dce6a713 d21 3e66376972bea4d0
d22 3f801272163691fc d23 bf7272163691fb82
d24 3ff01272163691fc d25 0000000000000000
d26 0000000000000000 d27 0000000000000000
d28 0000000000000000 d29 0000000000000000
d30 0000000000000000 d31 0000000000000000
scr 80000010
backtrace:
#00 pc 0000d598 /system/lib/libc.so
#01 pc 00003fb1 /system/lib/libCedarA.so
(android::CedarAAudioPlayer::getSpace()+10)
#02 pc 0000472d /system/lib/libCedarA.so
(android::CedarAPlayer::CedarAPlayerCallback(int, void*)+64)
#03 pc 00004814 /system/lib/libCedarA.so
#04 pc 0000e3d8 /system/lib/libc.so (__thread_entry+72)
#05 pc 0000dac4 /system/lib/libc.so (pthread_create+160)
stack:
423c3e48
423c3e4c
423c3e50
423c3e54
00000000
00000000
00000000
00000000
```

上述crash日志中，主要关心的是backtrace中的内容。我们可以轻易地使用 addr2line，将日志转化为程序调用栈。

经过转化后日志如下：

```
0. pthread_mutex_lock_impl
/home/gxzhang/grandstream/a20/android/bionic/libc/bionic/pthread.c:1187
1. android::Mutex::lock()
/home/gxzhang/grandstream/a20/android/frameworks/native/include/utils/Mutex.h:112
2. android::CedarAPlayer::StagefrightAudioRenderGetSpace()
/home/gxzhang/grandstream/a20/android/frameworks/av/media/CedarX-Projects/CedarA/CedarAPlayer.cpp:527
3. CDA_PlaybackThread
CedarAPlayer.cpp:0
4. __thread_entry
/home/gxzhang/grandstream/a20/android/bionic/libc/bionic/pthread.c:204
5. pthread_create
/home/gxzhang/grandstream/a20/android/bionic/libc/bionic/pthread.c:348
```

## 如何实现的呢?

### 前提条件
当然，有个前提条件：每个 .so 文件需要有对应的有符号文件。
例如：
```
addr2line -e -f libc.so 0000d598
```
其中，libc.so 必须是symbols文件，即未 striped。

### 反解命令格式如下:
``` bash
addr2line -C -f -e LIB ADRRESS
```
### 已知 tombstone 的格式如下:

```
......
backstrace:
//NUM pc  Address    LIB        *(function_name)
#00   pc  0000xxxx  libxxx.so
#01   pc  0000xxxx  libxxx.so   (function1()+10)
#02   pc  0000xxxx  libxxx.so   (function2(int, void*)+64)
#03   pc  0000xxxx  libxxx.so
#04   pc  0000xxxx  libxxx.so   (__thread_entry+72)
#05   pc  0000xxxx  libxxx.so   (pthread_create+160)
stack:
.......
```

### 实现算法:
1. 从 tombstone 中抽出 backtrace 的内容。这些内容是在 backtrace 和 stack 中间的。
2. 解析得到每一行的 NUM, ADDRESS, LIB 存到结构体 cmd 中
3. forall cmds
``` bash
exec `addr2line -C -f -e LIB ADDRESS`
```

### 代码如下:
``` c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#define TOMB_FILE "tombstone_00"
#define START_FLAG "backtrace:"
#define END_FLAG "stack:"
typedef struct Cmd {
    int num;
    char addr[10], lib[128];
    struct Cmd* next;
} Cmd;
typedef struct Cmd* PCmd;
static void printCmd(PCmd cmd);
static void printCmds(PCmd cmds);
// format:
// #NUM pc ADRRESS LIB *(function_name)
// example:
// #00 pc 0000xxxx lib111.so
static PCmd parse_cmd(char* line) {
    PCmd cmd = (PCmd) malloc(sizeof(Cmd));
    int find_num = 0, find_pc = 0, find_addr = 0, find_lib = 0;
    char str[128];
    while(*line != '\0') {
        if (*line == ' ') {
            line++;
            continue;
        }
        if (!find_num && *line == '#' && *(line+1) >= '0' && *(line+1) <= '9') {
            find_num = 1;
            sscanf(line, "%s", str);
            cmd->num = atoi(str+1);
            line+=strlen(str);
            continue;
        }
        if (find_num && !find_pc && *line == 'p' && *(line+1) == 'c') {
            find_pc = 1;
            line+=2;
            continue;
        }
        if (find_num && find_pc && !find_addr) {
            find_addr = 1;
            sscanf(line, "%s", cmd->addr);
            line+=strlen(cmd->addr);
            continue;
        }
        if (find_num && find_pc && find_addr && !find_lib) {
            find_lib = 1;
            sscanf(line, "%s", cmd->lib);
            line+=strlen(cmd->lib);
            continue;
        }
        line++;
    }
    if (find_num && find_pc && find_addr && find_lib) {
        return cmd;
    }
    return NULL;
}
static PCmd read_file(char* filename, int *count)
{
    FILE * fp;
    char * line = NULL;
    size_t len = 0, read;
    int i;
    PCmd head, tail, cmd;
    fp = fopen(filename, "r");
    if (fp == NULL)
        return;
    *count = 0;
    head = tail = NULL;
    int find_flag = 0;
    while ((read = getline(&line, &len, fp)) != -1) {
        if (read > 0) {
            for (i = read-1; i >= 0; i--) {
                if (line[i] == '\r' || line[i] == '\n') {
                    line[i] = '\0';
                }
            }
        }
        //printf(">>>>%s\n", line);
        if (strcmp(line, START_FLAG) == 0) {
            find_flag = 1;
        }
        if (strcmp(line, END_FLAG) == 0) {
            find_flag = 0;
        }
        if (find_flag == 0) {
            continue;
        }
        if (read > 0 && (cmd = parse_cmd(line))) {
            if (!head && !tail) {
                head = cmd;
                tail = cmd;
            } else {
                tail->next = cmd;
                tail = cmd;
            }
            cmd->next = NULL;
            *count++;
        }
    }
    return head;
}

static void printCmd(PCmd cmd) {
    if (cmd) {
        printf("%d,\t%s,\t%s\n", cmd->num, cmd->addr, cmd->lib);
    }
}
static void printCmds(PCmd cmds) {
    PCmd cmd = cmds;
    while(cmd) {
        printCmd(cmd);
        cmd = cmd->next;
    }
}
static void freeCmds(PCmd cmds) {
    int count = 0;
    PCmd cmd = cmds;
    while(cmd) {
        count++;
        cmds = cmd->next;
        free(cmd);
        cmd = cmds;
    }
    printf("%d object is free\n", count);
}
static void addr2line(PCmd cmds) {
    char cmd_str[256];
    PCmd cmd = cmds;
    while(cmd) {
        if (cmd->num == 0) {
            printf("\n=============================================================\n");
        }
        printf("%d. ", cmd->num);
        fflush(stdout);
        sprintf(cmd_str, "addr2line -C -f -e symbols%s %s", cmd->lib, cmd->addr);
        system(cmd_str);
        cmd = cmd->next;
    }
}

int main(int argc, char* argv[]) {
    int count = 0;
    if (argc != 2) {
        printf("need 1 arguement\n\n");
        return 0;
    }
    PCmd cmds = NULL;
    cmds = read_file(argv[1], &count);
    printCmds(cmds);
    addr2line(cmds);
    freeCmds(cmds);
    return 0;
}


```






