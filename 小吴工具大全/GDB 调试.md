# GDB 调试

## GDB 调试 coredump 文件

### coredump 文件

* `core`：进程号
* `coredump` 为核心转储文件
* 进程运行时，突然崩溃的一瞬间，为进程存储的一个快照

### 崩溃无法产生 coredump 的情况

* `core file size` 为 0，不能产生核心转储文件：通过 `ulimit -a`  **查看**，使用 `ulimit -c unlimited` **修改**，**默认会话及子进程**
* 不同情况的 `coredump` 产生的情况
  * coredump size <= core file size：可能产生
  * coredump size < 磁盘空间：产生
  * coredump size > 磁盘空间：不产生
* 此外还有使用系统指定差生核心转储文件，但是核心转储文件存放路径不存在，也不会创建，这需要你去将这里路径对应的文件夹进行创建
  * 查看核心转储文件存放为的代码：`cat /pro/sys/kernel/core_pattern`
  * 将这个文件中的路径（文件夹）创建出来即可

### 调试方法

* 命令：gdb \[binfile\]\[coredumpfile\]
* 调试思路：
  * 查看调用堆栈，寻找崩溃的原因
  * 根据崩溃点，查找代码，分析原因
  * 修复 bug
* gdb 命令：
  * bt（back trace）：显示当前程序的函数调用堆栈
  * f #number（#number 堆栈号）：切换到对应的堆栈号（#0）
* 通过跟进调用堆栈，就能找到对应的代码了

## GDB 调试多线程程序

### 基本指令

* 查看线程信息：info threads
* 切换到线程：t + 线程ID
* 切换堆栈：f + 堆栈号（bt 查看堆栈）
* 查看源代码：l + func
* 显示断点：i b （info breakpoint）

### 调度器锁

**调试时除了当前线程在运行，想要规定其他线程的运行情况，使用 `set scheduler-locking[mode]`**

* **mode**
  * **off**：不锁定任何线程，所有线程都可以继续执行
  * **on**：只有当前线程可以执行，其他线程都暂停运行
  * **step**：
    * 单步执行某一线程时，不会发生线程切换（跳转到其他线程），同时其他线程也会随着被调试线程的单步执行而执行
    * `continue，until，finish` 其他线程也执行，某一线程执行遇**断点**，GDB 会切换到这一线程（遇断点才切换）





