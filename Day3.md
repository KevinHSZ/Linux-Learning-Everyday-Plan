## ls /home/abc                             ( abc is a directory) 

task_struct → files_struct



```c
struct task_struct{
    struct fs_struct *fs; 
    struct files_struct *files;
}
```

​       fs_struct：文件系统上下文

它记录的是进程在文件系统树中的“位置”信息，这些信息通常与路径解析相关。

- **核心作用**：保存进程的**根目录**和**当前工作目录**。当你使用相对路径（如 `./test`）操作文件时，内核就是通过它来找到起点。

- **共享特性**：在创建线程时（使用 `CLONE_FS` 标志），多个线程可以共享同一个 `fs_struct`，这意味着在一个线程中调用 `chdir()` 改变工作目录，会影响到同进程的其他线程。

  

```c
struct path{
    struct vfsmount *mnt;
    struct dentry *dentry;    
}；
```



```
struct fs_struct{
    int umask;
    struct path root,pwd,altroot;
    
}
```

​      `fs_struct` 的重要成员

`fs_struct` 的定义在 `include/linux/fs_struct.h` 中清晰可见，主要成员包括：

- **`struct path root`**：进程的**根目录**。它决定了路径解析时，绝对路径（以 `/` 开头）的起点。
- **`struct path pwd`**：进程的**当前工作目录**。它决定了相对路径（如 `./file` 或 `../`）的解析起点。
- **`int umask`**：进程的**文件创建掩码**。在调用 `open()` 或 `mkdir()` 创建新文件/目录时，用于屏蔽掉权限位中不应设置的位。
- **`int users`**：**引用计数**。记录有多少个进程（或线程）共享这个 `fs_struct`。当计数归零时，该结构体会被释放。
- **`spinlock_t lock` / `seqcount_t seq`**：**同步机制**。用于保护结构体成员的并发访问，确保在多线程环境下读取 `root` 或 `pwd` 时的一致性。

```c
struct files_struct{
     struct fdtable *fdt;
     struct file *fd_array[NR_OPEN_DEFAULT];//通常是64在64位系统
     struct fdtable           fdtab;
     struct embedded_fd_set   close_on_exec_init;
     struct embedded_fd_set   open_fds_init;
   
    //标准输入        标准输出     标准错误
   // fd_array[0]  fd_array[1]  fd_array[2]
}
```

​       图中的 **`struct file \* fd_array[NR_OPEN_DEFAULT];`**
这就是那个**内嵌的初始指针数组**。

- `NR_OPEN_DEFAULT` 在 64 位系统下通常是 64。
- 当进程刚创建时，它指向的就是这个数组。
- 所以，当你打开标准输入、标准输出、标准错误时，**下标 0、1、2 对应的 `struct file` 指针，最初就保存在 `fd_array[0]`、`fd_array[1]`、`fd_array[2]` 中**。

图中这两行：
`struct embedded_fd_set close_on_exec_init;`
`struct embedded_fd_set open_fds_init;`

- 这是用来记录**位图（bitmap）**的初始空间。内核用一个 bit 代表一个 fd。
- 比如第 0 位是 1，代表 fd 0 被打开了；第 1 位是 1，代表 fd 1 被占用了。
- 同理，如果打开的文件很多，位图也会动态扩容，`fdtable` 里会有指针指向动态扩容后的新位图，这两个字段就是初始的内嵌空间。
- 

​          在 `files_struct` 中同时存在**默认表（内嵌的 `fdtab`）**和**正在使用的表（指针 `fdt`）**，这是 Linux 内核为了**极致优化性能**而采用的一种经典设计模式。

它们的核心区别在于：**一个是“出厂自带的初始装备”，另一个是“当前实际干活用的装备”**。

具体区别如下：

### 1. 默认表（`struct fdtable fdtab;` 以及内嵌的 `fd_array`）

- **物理位置**：它是直接嵌在 `files_struct` 结构体内部的，分配进程结构体时就一起分配了。
- **容量限制**：很小。在 64 位系统上，默认的 `fd_array` 只有 `NR_OPEN_DEFAULT`（通常是 64）个槽位。
- **作用**：作为**初始状态**。对于大多数普通进程来说，它们一辈子打开的文件可能都不超过 64 个。所以直接使用这个自带的表，**完全不需要额外去内存里申请空间**，既快又省内存。
- **生命周期**：只要进程在，它就一直在。哪怕后来扩容了，它依然占着坑（虽然不再被使用），因为它是结构体的一部分。

### 2. 正在使用的表（`struct fdtable *fdt;`）

- **物理位置**：它是一个**指针**。它既可以指向上面那个默认表，也可以指向后来动态申请的全新的、更大的 `fdtable`。
- **作用**：这是**内核代码实际读写文件描述符的唯一入口**。内核在 `open`、`read`、`close` 时，永远是顺着 `fdt` 指针去找当前的 `fd` 数组和位图。
- **动态变化**：
  - **初始时**：`fdt` 直接指向 `&fdtab`（默认表）。
  - **扩容时**：如果进程打开了超过 64 个文件，内核会 `kmalloc` 申请一个更大的 `fdtable` 结构体和更大的 `fd` 指针数组。初始化好之后，**原子地将 `fdt` 指针指向这个新的表**。

------

### 💡 为什么要这么设计？（小对象优化 + RCU无锁读）

1. **极致的性能**：90% 以上的进程，打开的文件都很少。让 `fdt` 直接指向内嵌的 `fdtab`，避免了分配内存的延迟，并且 `files_struct`、`fdtab`、`fd_array` 都在同一块连续内存中，**CPU Cache 命中率极高**。
2. **无锁扩容（RCU）**：当文件数爆满需要扩容时，内核不需要去“修改”那个内嵌的 `fdtab`。而是直接去外面新建一个大表，然后把 `fdt` 指针一换。
   - 由于指针替换是原子的，别的线程正在读取老表（`read` 操作）不会被打断。
   - 等所有读操作都退出了（RCU 宽限期结束），内核再去把老的大表或者旧资源给安全释放掉。

### 🎯 举个通俗的例子：

- **`fdtab`（默认表）**：就像你买书包时，**书包自带的一个小笔袋**。出门带两三支笔，用这个小笔袋足够了。
- **`fdt`（正在使用的表）**：这是你**手上实际拿着的装笔容器**。
  - 如果你的笔少，`fdt` 就指指那个小笔袋。
  - 如果你突然是个美术生，需要装 100 支笔，小笔袋装不下了，你就会去外面买个**大笔盒**，然后 `fdt` 变成指向那个大笔盒。那个小笔袋依然缝在书包上，只是你不再用它装笔了。

```
struct file{
     struct path f_path;
}
```



`ls` 是普通命令，用**绝对路径 /home/abc**，优先拿 `fs->root`：
`fs->root` 类型是 `struct path`

```
struct path {

    struct vfsmount *mnt;  

    struct dentry *dentry; 

};
```

`fs->root.dentry`        就是**根目录 / 的 dentry** 

`fs->root.mnt`              挂载实例，关联 `mnt->mnt_sb` → **super_block \*sb**

**一句话：task_struct →   fs_struct (root path) →   super_block + 根 dentry**







`struct fdtable` 是 Linux 内核中管理文件描述符的**真正核心结构**。它本身不包含“文件内容”，而是包含**指针数组**（用于寻址文件对象）和**位图**（用于管理分配状态）。

为了方便理解和记忆，可以将其关键成员分为以下三组：

## 一、 核心寻址：文件指针数组

- **`struct file __rcu \**fd;`**
  - **作用**：这就是那个**指针数组**！它指向一块内存，里面存放着所有的 `struct file *`。你的文件描述符（fd）就是这个数组的**下标**。例如 `fd[3]` 拿到的就是文件描述符 3 对应的文件对象指针。
  - **注意**：它并不一定固定指向 `files_struct` 里的初始 `fd_array`。如果打开文件较少，它指向内嵌的 `fd_array`；如果文件多扩容了，它指向动态申请的新数组。

## 二、 容量与状态：位图（Bitmaps）

- **`unsigned int max_fds;`**
  - **作用**：记录当前 `fd` 数组的总容量（最大能容纳多少个文件描述符）。
- **`unsigned long \*open_fds;`**
  - **作用**：**打开位图**。这是一个指针，指向一块内存。每一位（bit）对应一个 fd。如果第 N 位是 `1`，说明文件描述符 N 正在被使用；如果是 `0`，说明空闲。内核在 `open()` 分配新 fd 时，就是找这里面的第一个 `0`。
- **`unsigned long \*close_on_exec;`**
  - **作用**：**exec 关闭位图**。标记哪些 fd 在执行 `exec()` 系列函数（比如启动新程序）时，需要被自动关闭（通常在 `fcntl` 设置 `FD_CLOEXEC` 时标记）。
- **`unsigned long \*full_fds_bits;`**
  - **作用**：**满位图**（加速优化）。内核找空闲 fd 时是一个个找的，如果某一段（比如 64 个）全被占满了，就在这个位图里打个标记，下次找的时候直接跳过这一段，加快分配速度。

## 三、 生命周期管理

- **`struct rcu_head rcu;`**
  - **作用**：RCU 回收头。因为内核采用了无锁读（RCU）技术，当 `fdtable` 扩容需要释放旧表时，不能立刻 `free` 掉（因为可能还有别的线程在读旧表）。必须等一段“宽限期”后，通过这个 `rcu` 结构体安全地释放内存。

------

## 💡 补充：初始内嵌位图在哪里？

在 `struct files_struct` 中，你会看到下面两个内嵌的位图初始空间：

- `struct embedded_fd_set close_on_exec_init;`
- `struct embedded_fd_set open_fds_init;`

在 `files_struct` 初始化时，`fdtable` 的 `open_fds` 指针和 `close_on_exec` 指针会**指向**这两个内嵌的 `_init` 结构体。当打开的文件超过默认容量（通常是 64 个）触发扩容时，它们才会指向动态申请的新位图。

> **总结**：`fdtable` 就是一个**“文件目录本”**：
>
> - `fd` 告诉你**具体内容（文件指针）在哪页**；
> - `max_fds` 告诉你**这本目录本有多少页**；
> - `open_fds` 是一张**登记表**，告诉你哪页写了字，哪页还是空的。
