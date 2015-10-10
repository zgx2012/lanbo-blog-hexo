title: ssh 使用
date: 2015-10-10 16:17:52
tags: [ssh]
---

## ssh 登录服务器
```
ssh root@XXX.XXX.XXX -i 私有key文件
```
按照后面的配置可以省略“私有key文件”。

## 同时管理多个ssh私钥
新增ssh的配置文件，并修改权限
```
touch ~/.ssh/config
chmod 600 ~/.ssh/config
```
修改config文件的内容
```
Host 192.168.130.88
    IdentityFile ~/.ssh/id_rsa.work
    User lee
 
Host github.com
    IdentityFile ~/.ssh/id_rsa.github
    User git
```
这样在登陆的时候，ssh会根据登陆不同的域来读取相应的私钥文件。


## 利用expect脚本执行ssh命令
```
#!/usr/bin/expect -f

set password admin

spawn ssh root@192.168.130.88 "telnetd -l /system/xbin/sh"
expect "*assword:*"  
  
send "\$password\r"  
expect eof

```

> ## Expect
> 上面脚本中的自动交互用到了expect，那么什么是expect呢？

> expect是一个基于Tcl的用于自动交互操作的工具语言，它适合用来编写需要交互的自动化脚本，比如上面提到的SSH输入用户名密码，自动FTP等等场景。
除了具有Tcl的语法，expect提供了几个常用的命令：

> ### send
用来发送一个字符串，比如 send "hello world"。
初始情况下，这个字符串会发送到标准输出。如果你用的是max OSX或者linux，可以在Terminal下直接输入expect命令并回车，就进入了expect交互环境，此时，输入send "hello world"就可以看到结果。
一旦你的程序已经与其他程序进行交互，字符串就会被发送到其他程序那里。如上面的例子脚本中，我们调用send "\$password\r"就是把密码发送给SSH连接的服务器端指定端口。

> ### expect
与send相反，expect用来等待你所期望的字符串。比如expect "hello"
在expect后面跟的字符串中，你可以指定一个正则表达式。
expect会一直等待下去，除非收到的字符串与预期的格式匹配，或者到了超期时间。

> ### spawn
spawn用来启动一个新的进程，比如上面的spawn ssh root@192.168.130.88 "telnetd -l /system/xbin/sh", Expect会执行命令ssh root@192.168.130.88 "telnetd -l /system/xbin/sh"。
在交互式的场景中，当你输入命令后，可能服务器端会返回一些操作提示符，以让你输入命令。Expect提供了这样三个常用的命令，spawn, expect和send，恰好满足这种需要。把它们结合起来使用，可以实现很多简单的自动化脚本。
 
> 其它常用的命令还有：interact，比如你通过脚本自动连接到了某个ftp，并输入了用户名密码，此时需要人工输入一些命令，就可以使用interact命令，它会把脚本的控制权交给用户；sleep，等待多少秒等等。
 
> 由于expect是从Tcl继承下来的，所以也支持Tcl的语法和命令，比如变量声明、流程控制等等。


