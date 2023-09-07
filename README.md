# QEMU 项目剖析

## 0 部署 QEMU

1. 使用`git`下载QEMU源码：

```shell
https://github.com/qemu/qemu.git
```

2. 构筑项目：

```shell
cd qemu/
mkdir build
cd build/
../configure
make
```

3. `./configure`报错：
```shell
../meson.build:836:2: ERROR: Dependency "pixman-1" not found, tried pkgconfig

A full log can be found at /tmp/qemu/build/meson-logs/meson-log.txt
```

4. 下载`sudo apt-get install libpixman-1-dev && ../configure`

5. 编译构筑`make`


## 1 QEMU (User Mode) 框架

### 1.0 相关术语

- `HOST/source`：运行 QEMU 的主机体系结构
- `GUEST/target`：目标可执行程序的体系结构

### 1.1 主程序

- `linux-user/main.c`为用户态 QEMU 入口：

```txt
main ---- error_init    # 初始化错误信息
       |
       |- ac->init_machine -- tcg_init -- [tcg_exec_init] # tcg初始化 >>
       |
       |- cpu_create    # 
       |
       |- load_exec     # 加载可执行二进制程序
       |         |- prepare_binprm  # 读取目标二进制文件
       |         |- load_elf_binary # 加载elf文件
       |         |              |- load_elf_image
       |         |              |- setup_arg_pages
       |         |              |- copy_elf_strings
       |         |              |- load_elf_interp
       |         |              |- target_mmap
       |         |              |- create_elf_tables
       |         |              |- target_munmap
       |         |
       |         |- load_fit_binary # 加载flt文件
       |         |- do_init_thread  # 初始化重要寄存器
       |
       |- cpu_loop      # 翻译，执行
                |- cpu_exec_start   # 代码块预处理
                |- [cpu_exec]       # 主操作 >>
                |- cpu_exec_end     # 后处理
                |- process_queued_cpu_work  # CPU队列任务处理
                |- switch(...)      # 处理异常或系统调用
                |- process_pending_signals  # 

```



```txt
# 主函数
main ----- error_init
        |- module_call_init
        |- qemu_init_cpu_list
        |- ac->init_machine -- tcg_init
        |                   |- tcg_exec_init  
        |
        |- cpu_create
        |- tcg_prologue_init

# 调用翻译块


```

### 1.2 寄存器映射

- `cpu_create()` 是创建`CPUState`对象的操作，该对象用于描述CPU架构的相关数据，包括寄存器数据、CPU特性、GUEST端CPU架构相关数据等。
- `CPU{GUEST_ARCH}State`是一个结构体，用于描述目标CPU架构的相关数据，主要存储GUEST端的寄存器数据。GUEST端寄存器对应为HOST端的内存。

```txt
// 寄存器生命周期
memset(...);                    // regs初始化为0
loader_exec(...);               // 加载目标二进制可执行文件
    |- do_init_thread(regs, infop);    // 初始化一些寄存器
    |- init_thread(regs, infop);       // 初始化rax, rsp, rip
target_cpi_copy_regs(env, regs);// 将regs值赋到env里
```

- `env`来保存虚拟寄存器的首地址

```txt
// env生命周期
env = cpu -> env_ptr;
target_cpu_copy_regs(env, regs);    // 将regs值赋到env里
cpu_loop(env);                      // 动态翻译执行
    |- cpu_exec(cs) -- tb_find -- tb_gen_code -- tcg_gen_code
                    |- cpu_loop_exec_tb -- cpu_tb_exec -- tcg_qemu_tb_exec

```

- TCG使用变量来表示Guest架构中的寄存器, 因此Guest寄存器到Host寄存器的整个映射过程为: 
```
GUEST寄存器--(TCG前端)-->IR变量--(TCG后端)-->HOST寄存器
```
- TCG把GUEST中的寄存器等都抽象为统一的变量，只关注于他们的存活周期, 从而指导TCG后端分配寄存器，可以留给后端充足的优化空间。


### 1.2 TCG 变量
- TCGContext中变量如下：
    - TCGTemp对象数组：TCG使用数组索引表示变量, 指令中的操作数也只有变量与立即数。
    - 后端翻译时会把这个数组映射到寄存器或者内存, 再按照IR指令中的idx进行分配, 就可以完成空间分配的任务。

```cpp
struct TCGContext {  
    ...
    int nb_globals; //多少个全局变量
    int nb_temps;   //一共多少个变量
    ...;
    TCGTempSet free_temps[TCG_TYPE_COUNT * 2];  //空闲的临时变量, TCG_TYPE_COUNT=2
    TCGTemp temps[TCG_MAX_TEMPS]; /* 首先是全局变量, 之后是临时变量, TCG_MAX_TEMPS=512 */
    ...
};
```

- 变量的本质就是一段时间内对于空间的占用。因此TCG只关注变量的存活期, 通过空闲位图记录空间占用情况。
- 在生成IR时, 如果需要空间保存数据, 则手动申请, 会得到一个变量索引idx, 使用完毕后手动释放. 后续翻译时会以索引idx为依据给变量在Host中分配内存和寄存器。
- TCG变量分为三种：临时(TCG BB)、局部(TCG func)和全局变量(All TCG func)。
- 变量相关操作定义在`tcg/tcg.c`中，TCG变量分配函数如下, `tcg_temp_new_internal()`会先检查空闲位图`TCGTempSet`再从temps中分配。
- `cpu_R`是定义在`target/arm/translate.c`中的静态变量, 保存arm中寄存器到TCG变量的映射。


---


## 2 读写模块

### 2.1 读入ELF文件

- `load_elf_binary`检查仿真文件的magic number，如果是0x7fELF的话，说明该文件为ELF格式，调用`load_elf_binary`函数进行加载。



---

## 3 中间码 TCG IR

- `Linux-user`只支持模拟应用程序运行,入口在`linux-user/main.c`， 主要是支持ELF格式的可执行文件。



- 对于`loader_exec()`参数：
    - execfd 可执行文件的文件句柄
    - exec_path 可执行文件的文件路径
    - target_argv 目标二进制的参数
    - target_environ 目标相关环境列表
    - regs 寄存器
    - info 仿真镜像的信息
    - &bprm 二进制信息

### 3.1 执行

- qemu执行目标代码是以翻译块为基本单位整体处理之后再执行，一组连续执行的代码。
- `cpu_loop` 位于 `linux-user/i386/cpu_loop.c`，对于x86_64，无论32位还是64位，都使用这个文件。

#### 3.1.1 cpu_exec_start
- 目标：用于设置进入翻译执行状态时的相关参数。
- 过程：啥都没干，只是把一部分代码读入。

```cpp
void cpu_exec_start() {
    ...
    qatomic_set(&cpu->running, true);   // 上锁
    smp_mb();                           // 控制内存一致性
    qatomic_read();                     // 原子读
    ...
}
```

### 3.1.2 cpu_exec
- 目标：用于设置进入翻译执行状态时的相关参数。
- 过程：寻找代码块并执行，返回一个int值，值代表处理过程中遇到的异常。

```cpp
int cpu_exec() {
    ...
    CPUClass *cc = CPU_GET_CLASS(cpu);  // 获取cpu的相关类型信息
    cpu_handle_halt(cpu);               // 处理halt exception
    set_clocks();           // 设置时钟
    sigsetjmp();            // 保存当时的CPU环境, 当发生异常时跳回
    // 只要不发生处理不了的中断、不发生处理不了的意外，能找到代码块并执行
    while(!cpu_handle_exception()) {
        while(!cpu_handle_interrupt()) {
            tb_find();      // 寻找代码块
            cpu_loop_exec_tb();     // 执行代码块
        }
    }
    align_clocks();         // 如果guest代码提前送到，尝试对齐主机和虚拟时钟
    ...
}
```




### 3.2 tcg 前端

- 目标：GUEST -> TCG IR（解码）

```txt
tcg_exec_init -- cpu_gen_init -- tcg_context_init   # 初始化tcg上下文
              |- page_init                # 页表初始化
              |- tb_htable_init           # 初始化Helper函数hash表
              |- alloc_code_gen_buffer    # 获取Host指令缓冲区
              |- size_code_gen_buffer     # Host指令缓冲区大小
              |- tcg_prologue_init        # 初始化序言
```


#### 3.2.1 tcg_exec_init
- 目标：
- 过程：


#### 3.2.2 tb_context_init
- 目标：tcg上下文初始化。
- 过程：生成tcg序言和收尾。


#### 3.2.3 gen_intermediate_code



### 3.3 tcg 后端

- 目标：TCG IR -> HOST
- IR翻译至HOST代码的过程主要存放在`tcg/tcg.c`，其中起主要作用的是`tcg_gen_code()`函数。

```txt
cpu_exec -- tb_find -- tb_gen_code -- tcg_gen_code -- tcg_reg_alloc_start

```

#### 3.3.1 tb_alloc
- 目标：tcg上下文初始化。
- 过程：分配`tb`对象并添加到tcg上下文中，从tcg上下文中的生成指令缓冲区划分出`tb`对象。

#### 3.3.2 tb_gen_code
- 目标：用于设置进入翻译执行状态时的相关参数。
- 过程：tcg上下文初始化，生成`TranslationBlock`对象`tb`，翻译IR为Host指令。
- 存储：对于每一段 Guest 的`TranslationBlock`, QEMU 都会创建一个`TranslationBlock`。通过 `tcg_region_tree` 可以实现从`retaddr`到 `TranslationBlock`的映射。



### 3.4 中断

- linux-user是通过生成syscall意外，中断guest代码，转到host代码执行syscall的。

---

## 4 控制模块

### 4.1 输入参数

- `parse_args(argc, argv)`解析qemu的参数：

```shell
qemu-... [options] program [arguments ...]
```


