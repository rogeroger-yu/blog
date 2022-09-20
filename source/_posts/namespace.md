---
title: Namespace概念
tags: namespace
categories: container
---
# Namespace操作

## 查看Namespace

查看该目录/proc/[pid]/ns，能够看到该进程所在的namespace，各个namespace文件描述符在该目录下呈链接状态，链接目标的文件类型为namespace，没有具体目录承载

```bash
$ sudo ls -al /proc/110/ns/
total 0
dr-x--x--x 2 root root 0 May 17  2021 .
dr-xr-xr-x 9 root root 0 May 17  2021 ..
lrwxrwxrwx 1 root root 0 Jun 17  2021 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 root root 0 May 17  2021 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 May 17  2021 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0 May 17  2021 net -> net:[4026531993]
lrwxrwxrwx 1 root root 0 May 17  2021 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 Jun 17  2021 pid_for_children -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 May 17  2021 user -> user:[402653183
```

## 三种Namespace相关的基础系统调用

### clone

创建新进程的同时创建 namespace，将新进程加入新的namespace中

```c
/* Prototype for the glibc wrapper function */
#define _GNU_SOURCE
#include <sched.h>
int clone(int (*fn)(void *), void *child_stack, int flags, void *arg);
```

* fn: 指定一个由新进程执行的函数。当这个函数返回时，子进程终止。该函数返回一个整数，表示子进程的退出代码
* child_stack: 传入子进程使用的栈空间，也就是把用户态堆栈指针赋给子进程的 esp 寄存器。调用进程(指调用 clone() 的进程)应该总是为子进程分配新的堆栈
* flags: 表示使用哪些 CLONE_ 开头的标志位，与 namespace 相关的有:
  * CLONE_NEWIPC
  * CLONE_NEWNET
  * CLONE_NEWNS
  * CLONE_NEWPID
  * CLONE_NEWUSER
  * CLONE_NEWUTS
  * CLONE_NEWCGROUP
* arg: 指向传递给 fn() 函数的参数

> /proc/[pid]/ns的另外一个作用是，一旦文件被打开，只要打开的文件描述符（fd）存在，那么就算PID所属的所有进程都已经结束，创建的namespace就会一直存在。那如何打开文件描述符呢？把/proc/[pid]/ns目录挂载起来就可以达到这个效果：
>
> ```bash
> $ touch ~/uts
> $ mount --bind /proc/27514/ns/uts ~/uts
> ```
>
> **通常使用该方法将namespace保留下来**

### setns

将当前进程加入到已有的 namespace 中

```c
#define _GNU_SOURCE
#include <sched.h>
int setns(int fd, int nstype);
```

* fd: 目标namespace 的文件描述符。它是一个指向 /proc/[pid]/ns 目录中文件的文件描述符，可以通过直接打开该目录下的链接文件或者打开一个挂载了该目录下链接文件的文件得到
* nstype: 参数 nstype 让调用者可以检查 fd 指向的 namespace 类型是否符合实际要求。若把该参数设置为 0 表示不检查

### unshare

创建新的namespace，并将当前进程加入新的namespace

```c
#define _GNU_SOURCE
#include <sched.h>
int unshare(int flags);
```

* flags: 同上

## 隔离进程实战

### 准备unshare

unshare默认只能由超级用户执行，要想所有用户都可以创建namespace，需要设置该程序的capability

> Linux 将传统上与超级用户 root 关联的特权划分为不同的单元，称为 capabilites。Capabilites 作为线程(Linux 并不真正区分进程和线程)的属性存在，每个单元可以独立启用和禁用。如此一来，权限检查的过程就变成了：在执行特权操作时，如果进程的有效身份不是 root，就去检查是否具有该特权操作所对应的 capabilites，并以此决定是否可以进行该特权操作。比如要向进程发送信号(kill())，就得具有 capability **CAP_KILL**；如果设置系统时间，就得具有 capability **CAP_SYS_TIME**

这里需要设置cap_sys_admin权限用于操作pid namespace：

```bash
$ cp `which unshare` ./ # 避免污染原文件
$ setcap 'cap_sys_admin+ep' ./unshare
```

### 设置PID Namespace

* 创建pid namespace
* 在namespace中挂载`proc`文件系统

```bash
$ ./unshare --pid --mount-proc --fork bash
$ ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
lsfadmin       1       0  0 22:36 pts/1    00:00:00 bash
lsfadmin       3       1  0 22:36 pts/1    00:00:00 ps -ef
```

挂载proc文件系统的原因是：只是创建pid namespace，不能保证只看到namespace中的进程。因为类似`ps`这类系统工具读取的是`proc`文件系统。`proc`文件系统没有切换的话，虽然有了pid namespace，但是不能达到我们在这个namespace中只看到属于自己namespace进程的目的。

在创建pid namespace的同时，使用`--mount-proc`选项，会创建新的mount namespace，并自动`mount`新的`proc`文件系统。这样，`ps`就可以看到当前pid namespace里面所有的进程了。因为是新的pid namespace，进程的PID也是从1开始编号。对于pid namespace里面的进程来说，就好像只有自己这些个进程在使用操作系统。

### 设置Mount Namespace

上述PID Namespace隔离后，仍然能够看到操作系统的文件系统，为了只看到属于namespace自己的文件系统，需要创建mount namespace。`--mount-proc`的时候，其实就已经创建了新的mount namespace。可是，bash中还是能够看到操作系统的目录和文件。

#### 1. 创建自己的根文件系统

这里使用docker的ubuntu镜像作为例子。在容器中安装了`iproute2`，是为了方便后面network namespace的实验。

```bash
$ docker run -it ubuntu bash
[container] apt update
[container] apt install iproute2
```

将运行的容器的镜像导出来，解压到Linux当前目录的ubuntu子目录中：

```bash
$ docker ps
CONTAINER ID   IMAGE     COMMAND       CREATED         STATUS         PORTS     NAMES
3ed040cc2824   ubuntu    ...
$ docker export 3ed040cc2824 --output=ubuntu.tar
$ mkdir ubuntu
$ tar -xf ubuntu.tar -C ubuntu
```

#### 2. unshare设置chroot权限

`unshare`要使用`chroot`操作，将namespace中的根文件系统切换，而不影响操作系统。为了让非root用户完成这个操作，额外需要`cap_sys_chroot`权能：

```bash
$ setcap 'cap_sys_admin+ep cap_sys_chroot+ep' ./unshare
```

#### 3. mount系统根目录

使用`unshare`创建mount namespace，并将系统根目录切换到刚刚创建的根文件系统目录中。这样，namespace中能看到的全部目录，其实只是操作系统中的一个子目录。并且后续的`mount`操作，也只会影响mount namespace。

```bash
$ ./unshare --mount --root /home/lsfadmin/shared/ns/ubuntu --pid --mount-proc --fork bash

I have no name!@linux1 :/$ mount
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
I have no name!@linux1 :/$ cat /etc/lsb-release 
DISTRIB_ID=Ubuntu
DISTRIB_RELEASE=20.04
DISTRIB_CODENAME=focal
DISTRIB_DESCRIPTION="Ubuntu 20.04.3 LTS"
```

显然，这里新的`ubuntu` 镜像和操作系统使用的小版本是不一下样的。隔离出来了。原本应该搞一个centos之类的镜像，那样看起来区别大一些。

到目前为止，Linux操作系统中的bash和namespace中的bash看到文件系统已经不同。当然，还可以通过在操作系统中执行`lsns`命令查看新创建的namespace。

```bash
$ pstree -p 904489
unshare(904489)───bash(904490)
$ lsns -p 904490
        NS TYPE   NPROCS    PID USER     COMMAND
4026532420 mnt         1 904490 lsfadmin bash
4026532421 pid         1 904490 lsfadmin bash
```

嗯，确实是给`unshare`出来的进程创建了新的namespace

### 设置User Namespace

查看当前用户id

```bash
$ id
uid=1001(yuqian.0001) gid=1001(yuqian.0001) groups=1001(yuqian.0001),999(docker),1000(tiger),2001(admin)
```

在user namespace里面，可以自己管理用户的。也就是说，我想是谁就是谁。要做到这一步还是一个组合拳。先创建user namespace：

```bash
$ ./unshare --user --mount --root /home/lsfadmin/shared/ns/ubuntu --pid --mount-proc --fork bash
$ id
uid=65534(nobody) gid=65534(nogroup) groups=65534(nogroup)
```

新开一个终端，查看新建namespace的bash进程id为`1904696`

user namespace默认初始化的用户是`nobody`(65534)。让当前账户成为改namespace的root用户：

> uid_map文件格式：
>
> ```
> ID-inside-ns ID-outside-ns length
> ```
>
> - 第一个字段ID-inside-ns表示在容器显示的UID或GID，
> - 第二个字段ID-outside-ns表示容器外映射的真实的UID或GID。
> - 第三个字段表示映射的范围，一般填1，表示一一对应。

```bash
$ echo '0 1000 1' > /proc/1904696/uid_map
```

再切回到namespace里面（在新开的终端中），可以发现uid已经切换了。

```text
nobody@linux1:/$ id
uid=1000(lp) gid=65534(nogroup) groups=65534(nogroup)
```

不错，可以用新身份创建文件了：

```text
nobody@linux1:/$ touch from-namespace
nobody@linux1:/$ ls -nl from-namespace 
-rw-rw-r-- 1 0 65534 0 Nov 30 14:00 from-namespace
```

第三列，用户7创建文件，如我所料。

但是，事实上，一直以来，`我`从未改变。Namespace外：

```text
$ cd ubuntu
$ ls -nl from-namespace 
-rw-rw-r-- 1 1000 1000 0 Nov 30 06:00 from-namespace
```

文件创建的用户依然是`lsfadmin`，依然是`1000`。操作系统会确保，无论怎样，都是以登陆用户授权访问文件系统。Namespace仅仅是让在那个隔离里面看起来不一样而已。



# UST Namespace

UTS(UNIX Time Sharing) namespace是最简单的一种 namespace。UTS 中主要包含了主机名（hostname）、域名（domainname）和一些版本信息，其中主机名（hostname）、domainname(域名)可以被修改，其余只读

## hostname



## domainname



## uname
