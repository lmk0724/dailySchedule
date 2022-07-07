# dailySchedule

### 7.2

今天了解到了这个项目，配了一下环境。采用的是wsl2下载ubuntu的方式。

### 7.3

今天创建了用于记录每日工作的github仓库，并且学习了一下github classroom相关的知识。

### 7.4

今天主要是看《计算机组成与设计 risc-v版》的一二章，和risc-v手册。

* 理解了一下risc-v支持的一些指令。
* 还看了一下risc-v指令集下的调用函数的汇编程序，处理终端的汇编程序。
* 看了一下risc-v设计的三种模式，用户模式，机器模式，监管者模式。机器模式是权限最高的，用户模式是权限最低的。机器模式可以处理异常，共有两种异常，一是同步异常，而是中断。加入了监管者模式之后，还可以将部分异常处理导向监管者模式。

### 7.5

今天最开始研究了一下OS2. 还是没有搞清楚这个实验的逻辑是什么。

出于好奇，看了一下ci-user目录下的内容，看到了/ci-user/check/ch2.py， 里面有temp数组，和expected数组，expected数组很显然就是本章节下面的多任务执行的输出，但是temp数组会更新expected数组。发现了这个之后，我甚至产生了一个想法，修改user下面的chapter2相关的任务程序的输出。

但是结合测试二的题目，try something in os2 in os2 DIR，显然仅仅是需要修改os2这个文件夹下面的文件，不需要修改user文件夹下的数据。

最后问了一下其他的同学，没想到仅仅是修改了一下os2文件夹下的Cargo.toml中的risc-v的地址，从github的地址修改为gitee的地址。这个还是存有一定的疑惑。

然后剩下的时间，就是一直在做rustlings。进度到了standard library types。

### 7.6

今天把剩下的rustlings做完了。

然后就开始看项目提供的文档了，一直对这个项目要完成什么，怎么完成感到十分疑惑，有一种云山雾罩的感觉。所以今天仔细看了一下项目的文档。通过看文档，大致理解了本项目的任务如下：

* 理解提供的框架代码
* 完成每章后面的练习

本项目的背景大致如下：是运行在一个裸机系统上面，没有任何的系统调用，为了实现某种目的，需要我们用rust手写系统调用。因为是在risc-v指令集条件下的系统，所以这中间还需要对risc-v和汇编语言有所了解。

### 7.7

今天上午结合着代码看第三章的文档。读着读着似乎有些理解了，一个特点是用汇编语言写一些函数，然后使用extern “C”就可以把这些函数给引入rust环境，rust可以调用他们。

今天下午就开始做chapter3的练习了。

首先在项目的根目录下运行```make test3```。出现了熟悉的错误。

![image-20220707194152951](pic/image-20220707194152951.png)

这个错误在第二章的时候出现过，将risc-v os的源从github改为gitee即可。

然后开始实现练习中的```sys_task_info```系统调用。此系统调用需要返回当前运行任务的状态，调用的系统调用的次数统计，和运行的时间。

通过观察代码，每个```task```有一个绑定的```TaskControlBlock```，包括```TaskStatus,TaskContext```，这两个属性都是和该```task```唯一绑定的，联想到，系统调用的两个信息，系统调用的次数和运行的时间也是和```Task```唯一绑定的。但是运行时间需要做一下处理，只需要在```TaskControlBlock```里面添加一个```task```开始运行的时间即可，调用该系统调用的时候，用当前时间减去```task```开始运行的时间就是该任务运行的时间。修改之后的```TaskControlBlock```如下：

```rust
// os/task/task.rs
pub struct TaskControlBlock {
    pub task_status: TaskStatus,
    pub task_cx: TaskContext,
    // LAB1: Add whatever you need about the Task.
    pub task_begin_time: usize,
    pub syscall_times: [u32; MAX_SYSCALL_NUM],
}

```

修改了结构体的属性，对应的需要修改```os/task/mod.rs```中的```lazy_static!```中相关的```TaskControlBlock```初始化代码。

然后需要思考的是新添加的两个属性什么时候更新。```task_begin_time```利用```timer.rs```中提供的```get_time()```进行初始化，初始化的时机就是该任务的状态由```UnInit```变为```Running```的时候，涉及任务状态改变的只有两个地方，```os/task/mod.rs```中```TaskManager```的两个方法，```run_first_task```和```run_next_task```。

```syscall_times```更新的时机，就是```task```调用系统调用的时候，也就是```os/syscall/mod.rs```里面的```syscall```函数。更新的时候有个难点，如何确定当前运行的是哪个任务。通过理解框架代码，发现有一个全局的静态对象```TASK_MANAGER```，保存着当前运行的是哪个任务，并且还有加载进来的所有任务的```TaskControlBlock```。直接按照框架的风格，为```TaskManager```实现一个私有方法，在外面提供一个公有函数接口。

```rust
// os/task/mod.rs
 fn update_syscall_times(&self, syscall_id: usize){
        let mut inner = self.inner.exclusive_access();
        let index = inner.current_task;
        inner.tasks[index].syscall_times[syscall_id] +=  1;
  }
pub fn update_syscall_times(syscall_id: usize){
    TASK_MANAGER.update_syscall_times(syscall_id);
}
```

最后就是系统调用的实现了，与上面的逻辑相似。

```rust
// os/task/mod.rs
fn set_task_info(&self, taskinfo: *mut TaskInfo){
        let inner = self.inner.exclusive_access();
        let t = (get_time() - inner.tasks[inner.current_task].task_begin_time)*1000/CLOCK_FREQ;
        // print!("{}",t);
        unsafe {
            *taskinfo = TaskInfo{
                // status: inner.tasks[inner.current_task].task_status,
                status: TaskStatus::Running,
                syscall_times: inner.tasks[inner.current_task].syscall_times,
                time: t,
            };
        }
       
    }
pub fn set_task_info(taskinfo: *mut TaskInfo){
    TASK_MANAGER.set_task_info(taskinfo);
}
// os/syscall/process.rs
pub fn sys_task_info(ti: *mut TaskInfo) -> isize {
    set_task_info(ti);
    0
}
```

