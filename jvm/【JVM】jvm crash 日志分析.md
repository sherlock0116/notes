# 【JVM】jvm crash 日志分析

## 一、生成

1. 生成 error 文件的路径：你可以通过参数设置 `-XX:ErrorFile=/path/hs_error%p.log`, 默认是在java 运行的当前目录 [default: ./hs_err_pid%p.log]

2. 参数 -XX: OnError 可以在 crash 退出的时候执行命令，格式是 `-XX:OnError=“string”`,  `<string>` 可以是命令的集合，用分号做分隔符, 可以用 "%p" 来取到当前进程的ID.

  ```
  例如:
  // -XX:OnError="pmap %p"                // show memory map
  // -XX:OnError="gcore %p; dbx - %p"     // dump core and launch debugger
  ```

  在 linux 中系统会 fork 出一个子进程去执行 shell 的命令，因为是用 fork 可能会内存不够的情况，注意修改你的 `/proc/sys/vm/overcommit_memory` 参数, 不清楚为什么这里不使用 vfork

3. `-XX:+ShowMessageBoxOnError` 参数，当 jvm crash 的时候在 linux 里会启动 gdb 去分析和调式，适合在测试环境中使用。

## 二、什么情况会生成 error 文件

linux 内核在发生 OOM 的时候会强制 kill 一些进程, 可以在 `/var/logs/messages` 中查找

## 三、Error crash 文件的几个重要部分

### 3.1 错误信息概要

```java
#
# A fatal error has been detected by the Java Runtime Environment:
#
#  SIGSEGV (0xb) at pc=0x0000000120cecd8b, pid=24103, tid=0x000000000000712f
#
# JRE version: Java(TM) SE Runtime Environment (8.0_251-b08) (build 1.8.0_251-b08)
# Java VM: Java HotSpot(TM) 64-Bit Server VM (25.251-b08 mixed mode bsd-amd64 compressed oops)
# Problematic frame:
# C  [librocksdbjni3847518070546690970.jnilib+0x19d8b]  _Z20key_may_exist_helperP7JNIEnv_PN7rocksdb2DBERKNS1_11ReadOptionsEPNS1_18ColumnFamilyHandleEP11_jbyteArrayiiP8_jobjectPb+0xdb
#
# Failed to write core dump. Core dumps have been disabled. To enable core dumping, try "ulimit -c unlimited" before starting Java again
#
# If you would like to submit a bug report, please visit:
#   http://bugreport.java.com/bugreport/crash.jsp
# The crash happened outside the Java Virtual Machine in native code.
# See problematic frame for where to report the bug.
#
```

1. ````
   SIGSEGV (0xb) at pc=0x0000000120cecd8b, pid=24103, tid=0x000000000000712f
   ````

   - SIGSEGV 错误的信号类型 
   - pc IP/PC寄存器值也就是执行指令的代码地址
   - pid 进程id

2. ```
   # Problematic frame:
   # C  [librocksdbjni3847518070546690970.jnilib+0x19d8b]  _Z20key_may_exist_helperP7JNIEnv_PN7rocksdb2DBERKNS1_11ReadOptionsEPNS1_18ColumnFamilyHandleEP11_jbyteArrayiiP8_jobjectPb+0xdb
   #
   ```

   就是导致问题的动态链接库函数的地址:pc=0x0000000120cecd8b 和 +0x19d8b 指的是同一个地址，只是一个是动态的偏移地址，一个是运行的虚拟地址

