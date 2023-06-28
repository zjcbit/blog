---
title: "bpf系统调用"
date: 2023-06-24T13:24:55+08:00
draft: false
tag: ["技术", "ebpf"]
categories: ["技术", "ebpf", "翻译"]
---
# bpf系统调用
## 1. 前言
- 正如您在第1章中看到的，当用户空间应用程序希望在内核空间做一些事情时，它们使用系统调用API发出请求。因此，如果用户空间应用程序想要将eBPF程序加载到内核中，则必须涉及一些系统调用。事实上，有一个名为bpf()的系统调用，在本章中，我将向您展示如何使用bpf()的系统调用来加载eBPF，并与ebpf程序和map交互。
- 需要注意的是，内核中运行的eBPF代码并不使用系统调用来访问Map。系统调用接口仅由用户空间应用程序使用。相反，内核中运行的eBPF程序使用helper函数来读取和写入Map;在前两章中，您已经看到了这方面的示例。
- 如果你自己编写eBPF程序，很有可能你不会直接使用bpf()这个系统调用。而是有很多封装好的库来进行使用，这些库是更高级的抽象，会是我们编写程序变得更加简单，我将在后面的章节来介绍。也就是说，这些抽象通常非常直接地映射到底层syscall命令。但是，无论您使用什么库，您都需要掌握：底层操作与加载ebpf程序、创建和访问Map等；您将在本章中看到这些操作。
- 在我开始展示如何使用bpf()系统调用之前，让我们先看下bpf()系统调用到底是什么，下面是bpf()系统调用的定义:
  ```c
  int bpf(int cmd, union bpf_attr *attr, unsigned int size)
  ```
  * cmd: 指定要执行的命令，bpf()并不是只做一件事，有许多不同的命令可用于操作eBPF程序和映射。图4-1展示了用户空间程序可能用于加载eBPF程序、创建Map、将程序attach到事件上、以及访问映射中的键值对的常用命令。
  ![4-1](/assets/images/bpf/4-1.png "Figure 4-1")
   图4-1 用户空间程序与eBPF程序交互，并使用syscall映射到内核中
  * attr: 这个参数用于保存所需执行命令cmd的所有参数，并指定了attr中需要多少字节。
- 我第一章使用过**strace**这个命令，用来演示一个用户空间的程序会使用syscalls发起多少次请求，在这一章中我将用这个命令来演示bpf()系统调用是如何使用的。为了避免演示过程过于混乱，我将忽略参数attr的一些细节，除非是有需要特别注意的地方。
- 本章源码在 https://github.com/zjcbit/learning-ebpf/tree/main/chapter4 可以找到
## 2. 示例
对于此示例，我将使用名为hello-buffer-config.py的BCC程序，此程序在运行时向perfbuffer缓冲区发送一条消息，并向将execve()系统调用的信息发送到用户空间。并允许为每个用户ID发送不同的消息。
```c
// 声明一个结构体用于特定的12个字符的消息
struct user_msg_t {
   char message[12];
};

// BCC的宏定义用于声明一个HasTable，名字为config，键类型为u32(其大小即为UserID，如果你没有指定type，则默认K、V都是u64)，值为user_msg_t
BPF_HASH(config, u32, struct user_msg_t);

// 您可以将任意数据提交到perfbuffer，无需在此处指定任何数据类型
BPF_RINGBUF_OUTPUT(output, 1); 

// 在此示例中，程序始终提交的是data_t 这个数据结构
struct data_t {     
   int pid;
   int uid;
   char command[16];
   char message[12];
};

int hello(void *ctx) {
   struct data_t data = {}; 
   char message[12] = "Hello World";
   struct user_msg_t *p;

   data.pid = bpf_get_current_pid_tgid() >> 32;
   data.uid = bpf_get_current_uid_gid() & 0xFFFFFFFF;

   bpf_get_current_comm(&data.command, sizeof(data.command));
   // 在使用lookup函数获取用户ID后，代码在以该用户ID作为键的hashtable中查找消息, 如果有匹配的消息则发送此消息，否则使用Hello World
   p = config.lookup(&data.uid);
   if (p != 0) {
      bpf_probe_read_kernel(&data.message, sizeof(data.message), p->message);       
   } else {
      bpf_probe_read_kernel(&data.message, sizeof(data.message), message); 
   }

   output.ringbuf_output(&data, sizeof(data), 0); 
 
   return 0;
}
```
Python 代码还有两行附加行：
```python
b["config"][ct.c_int(0)] = ct.create_string_buffer(b"Hey root!")
b["config"][ct.c_int(501)] = ct.create_string_buffer(b"Hi user 501!")
```
这两个消息用于在config中定义0和501用户ID的消息。对应于转发到此虚拟机上的根用户和我的用户ID。此代码使用Python中的ctypes,来确保与C语言中定义的user_msg_t有相同的类型。

下面是此示例的一些说明性输出，我们需要用两个终端，一个执行编写的程序，一个来捕获它：
```bash
Terminal 1                          Terminal 2
$ ./hello-buffer-config.py
37926 501 bash Hi user 501!         ls
37927 501 bash Hi user 501!         sudo ls
37929 0 sudo Hey root!
37931 501 bash Hi user 501!         sudo -u daemon ls
37933 1 sudo Hello World
```
现在你了解了这个程序是用来做什么的了，我想向你展示当我们运行我们的程序时bpf()的系统调用。我将使用**strace**指定-e来展示bpf()的系统调用。
``` bash
$ strace -e bpf ./hello-buffer-config.py
```
如果你亲自尝试运行，就会发现有很多的系统调用，每个系统调用都能看到bpf()在做什么。输出类似如下：
```
bpf(BPF_BTF_LOAD, ...) = 3
bpf(BPF_MAP_CREATE, {map_type=BPF_MAP_TYPE_PERF_EVENT_ARRAY...) = 4
bpf(BPF_MAP_CREATE, {map_type=BPF_MAP_TYPE_HASH...) = 5
bpf(BPF_PROG_LOAD, {prog_type=BPF_PROG_TYPE_KPROBE,...prog_name="hello",...) = 6
bpf(BPF_MAP_UPDATE_ELEM, ...}
...
```