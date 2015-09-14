title: NDK 开发 Native可执行程序
date: 2015-09-14 20:04:39
tags: [android, ndk]
---

我们都知道通过 NDK 可以开发 Native 的 .so 库，然后通过 JNI 的方式给 Java 层调用。

但如何开发一个直接可以运行的 C 程序呢？

其实很简单，首先得有一部有 ROOT 权限的 Android 手机。（开发步骤，基本上同开发一个.so一样）

1. 下载并解压 [NDK](http://developer.android.com/sdk/ndk/index.html) 

        $ chmod a+x android-ndk-r10c-darwin-x86_64.bin
        $ ./android-ndk-r10c-darwin-x86_64.bin

2. 设置 NDK 路径和 PATH
修改 ~/.bashrc 文件

        $ NDK=～/soft/ndk/android-ndk-r10d
        $ PATH=$PATH:$NDK

3. 新建 hello_exe 工程

        $ cd ~/soft/ndk/workspace
        $ mkdir -p hello_exe/jni && cd hello_exe
        $ ls
        jni

4. 在 jni 目录下创建 hello_exe.c

        #include <stdio.h>

        int main(int argc, char **argv)
        {
            printf("Hello, world!\n");
            return 0;
        }

5. 在 jni 目录下创建 Android.mk

        LOCAL_PATH := $(call my-dir)
        
        include $(CLEAR_VARS)
        
        LOCAL_MODULE    := hello_exe
        LOCAL_SRC_FILES := hello_exe.c
        
        include $(BUILD_EXECUTABLE)

6. 回到 hello_exe 工程目录，编译

        $ cd ..
        $ source ~/.bashrc   # 这一步是使之前的PATH设置生效，也可以重启生效。
        $ ndk_build
        [armeabi] Compile thumb  : hello_exe <= hello_exe.c
        [armeabi] Executable     : hello_exe
        [armeabi] Install        : hello_exe => libs/armeabi/hello_exe

7. 将 hello_exe push 到手机上。当然需要有 ROOT 权限。

        $ adb shell mkdir -p /data/hello_exe
        $ adb push libs/armeabi/hello_exe /data/hello_exe/
        $ adb shell
        # cd /data/hello_exe
        # chmod 777 hello_exe
        # ls -l
        -rwxrwxrwx    1 root     root          9464 Mar 19 14:56 hello_exe
        
8. 执行程序

        # ./hello_exe
        Hello, world!
