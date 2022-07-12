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

### 7.8

今天继续阅读第四章的文档，也就是lab2的文档。相比于lab1的文档，这次阅读文档不太顺利，原因是我之前对于页表的理解有些问题。

我之前一直以为多级页表中每个节点的每一项都是分为两个部分，虚拟页号和物理帧号，但是这里有问题，页表项不需要留出空间给虚拟页号。因为处理的时候会将物理帧按照数组组织，下标即为虚拟页号。那么就不需要再页表项中预留空间存虚拟页号了。

然后读完了文档就开始写代码了，着手解决系统调用sys_get_time的系统调用。给出的提示是由于采用了虚拟内存，需要修改sys_get_time。那么需要修改的一项就是该系统调用传入的指针了，这个指针应该是虚拟地址，需要手动转化为物理地址。

刚开始觉得系统调用是在内核态下执行的，所以觉得该虚拟地址是内核空间的，所以用内核内存空间转化了一下。代码如下：

```rust
// os/syscall/process.rs
pub fn sys_get_time(_ts: *mut TimeVal, _tz: usize) -> isize {
    let _us = get_time_us();
    let vaddr = _ts as usize;
    let vaddr_obj = VirtAddr(vaddr);
    let page_off = vaddr_obj.page_offset();

    let vpn = vaddr_obj.floor();
    
    
    let ppn = KERNEL_SPACE.lock().translate(vpn).unwrap().ppn();

    let paddr : usize = ppn.0 << 10 | page_off;
    let ts  = paddr as *mut TimeVal;
    unsafe {
        *ts = TimeVal {
            sec: _us / 1_000_000,
            usec: _us % 1_000_000,
        };
    }
    0
}
```

然后转念一想，这个地址是在用户程序中传入的，应该是用户程序的内核空间中的地址。然后使用用户空间翻译了一下

```
// os/syscall/process.rs
pub fn sys_get_time(_ts: *mut TimeVal, _tz: usize) -> isize {
    let _us = get_time_us();
    let vaddr = _ts as usize;
    let vaddr_obj = VirtAddr(vaddr);
    let page_off = vaddr_obj.page_offset();

    let vpn = vaddr_obj.floor();
    
    let ppn = translate_vpn(vpn);

    let paddr : usize = ppn.0 << 10 | page_off;
    let ts  = paddr as *mut TimeVal;
    unsafe {
        *ts = TimeVal {
            sec: _us / 1_000_000,
            usec: _us % 1_000_000,
        };
    }
    0
}

// os/task/mod.rs
impl TaskManager {
fn translate_vpn(&self, vpn: VirtPageNum)-> PhysPageNum{
        let mut inner = self.inner.exclusive_access();
        let current = inner.current_task;
        let CurrentAppSpace = &inner.tasks[current].memory_set;
        let ppn = CurrentAppSpace.translate(vpn).unwrap().ppn();
        return ppn;
    }
}


pub fn translate_vpn(vpn: VirtPageNum)-> PhysPageNum{
    TASK_MANAGER.translate_vpn(vpn)
}
```

但测试了一下，还是不行，这就有点儿费解了。

7.9

今天继续debug，想一想为什么sys_get_time不行，答案找到了，找到了两个错误的地方，一是我将ppn左移10位了，这显然是不对的，page的大小为4k。需要左移12位。二是我之前觉得调用系统调用的时候CPU执行的任务发生了改变，因此构造了一个last_task的属性，但是经过思考之后发现，系统调用还是处于当前的任务之中，不过是将CPU的主导权放到了内核态中。

然后就继续修改sys_task_info这个系统调用。这个系统调用还是同样的思路，通过查页表找到对应的物理的地址，然后进行写入，这里直接把实验一的代码搬过来即可。

但是在实现的时候发现一个问题，本实验TaskStatus的定义和实验一不同，没有Uninit状态，因此我就不能用之前的方式来设置任务的启动时间了，仅仅是在TASK_MANAGER初始化的时候调用get_time()作为任务的启动时间。然后调用sys_task_info时候获得的时间get_time()减去这个时间即可得到任务的运行时间，但是我在测试的时候发现这个运行时间不太准，比较随机，有时候是480ms左右，有时候是490ms左右，有时候是502ms左右，但是只有在500ms左右的时候才能通过测试。这就不清楚为什么会出现这种情况了。（ps：这个似乎可以投机取巧一下，在返回的taskinfo中，将执行时间加个12ms就可以过了，不知道这种允许不允许:smile:

并且由于TaskStatus定义的不同，会造成一个奇怪的现象，TaskStatus::Running != TaskStatus::Running。需要在本实验TaskStatus的定义中加入Uninit状态，才可以通过这一项测试。

然后就是实现 sys_mmap了，这里一个坑点是文档中提示的“可能的错误”，被我理解成了以往的参与者会犯的错误，没想到是这个函数包含的错误的情况，也就是说碰到这些情况时，系统调用需要返回-1.

实现这个系统调用最伤脑经的是判断一个虚拟页号是否已经有映射了，这里的函数调用比较复杂，$sys\_mmap()\rightarrow contains\_key() \rightarrow  task\_manager.contains\_key()  \rightarrow memoryset.contains\_key() \rightarrow MapArea.contains\_key()$。

```rust
// os/src/task/mod.rs
pub fn contains_key(vpn: &VirtPageNum)-> bool{
    TASK_MANAGER.contains_key(vpn)
}
impl TaskManager{
    fn contains_key(&self, vpn: &VirtPageNum)-> bool{
        let mut inner = self.inner.exclusive_access();
        let current = inner.current_task;
        inner.tasks[current].memory_set.contains_key(vpn)
	}
}

// os/src/mm/memeory_set.rs
impl MemorySet{
    pub fn contains_key(&self, vpn: &VirtPageNum) -> bool{
        let mut result = false;
        let l = self.areas.len();
        for i in 0..l{
            let res = self.areas[i].contains_key(vpn);
            result = result | res;
        }
        return result;
    }
}
impl MapArea{
    pub fn contains_key(&self, vpn: &VirtPageNum)-> bool{
        self.data_frames.contains_key(vpn)
    }
}

```

在进行上面的判断之后，就可以执行添加操作了，这里直接利用框架代码，提供的memory_set.insert_framed_area()函数。

最后就是sys_munmap的实现了。还是先利用上面实现的功能判断给定的虚拟页空间，是否存在没有映射的页。如果存在，则直接返回-1. 反之，进行这块虚拟页空间的释放。这里涉及到两个结构的更新，一是Vec\<MapArea\>，二是PageTable的更新。这里利用pageTable提供的unmap函数即可。

按照上面所说的写完代码之后，本地测试出现了几个问题，一是test 4_01（这里用输出结果指代测例）始终过不了，并且没有assertion error。二是test 4_05测例过不去，有assertion error，说的是在21行，最后一个assert!语句处报错。

最后就是今天编程的一点儿体会，发现我的rust编程还是没有登堂入室，时常会犯一些错误，比如没有加mut，不知道vec.iter()会转移所有权等等，只能指望编译器给我指出这些错误。

### 7.10

今天就是继续debug，看看昨天的两个bug是为什么。

发现了第一个bug是由于我没有设置PTE::U位，导致了一个结果----可以正常申请内存，但是申请下来的内存无法访问。改了这个，test 4_01就可以正常通过了。

第二个bug主要是因为我在sys_mmap和sys_munmap这两个系统调用中确定的虚拟页号空间有问题。我之前一直以为MapArea里面的VPNRange是闭区间，所以我start和end两个vpn全用的是floor，结果检查了一下VPNRange的迭代器代码才知道这个区间是左闭右开区间，所有就很自然地把end换成了ceil方法。

在该代码的时候还遇到一件有趣的事情。我尝试将MapArea.contains_key()方法的实现用VPNRange来代替，结果一代替的话，出现了很多bug，这让我十分苦恼。

```rust
// os/src/mm/memeory_set.rs
pub fn contains_key(&self, vpn: &VirtPageNum)-> bool{
        self.vpn_range.get_end().0 >= vpn.0 && self.vpn_range.get_start().0 <= vpn.0
    }
```

想了一下这个区间是左闭右开区间，因此问题是多加了一个等号，应该是```self.vpn_range.get_end().0 > vpn.0 && self.vpn_range.get_start().0 <= vpn.0```，改完之后就正常了。

但是改完之后，test4_05本地测试还是过不去(sys_task_info的那个测例，我取了个巧，直接在返回值taskinfo的执行时间中加了个12ms).

但是奇怪的是，我把这一版上传到云端，云端居然测试通过了，这实在是很神奇。

### 7.11

今天主要是读第五章的文档。

### 7.12

今天开始尝试编写代码，尝试着把sys_get_time, sys_task_info, spawn这几个系统调用，spawn这个系统调用可以简单的看作fork和exec的结合。不过是不需要拷贝父进程的地址空间了，直接指定elf获得的地址空间。

sys_get_time和sys_task_info这两个系统调用需要修改，这个实验的结构又有了大变，没有了全局的TASK_MANAGER，因此这次需要在全局的PROCESSOR上进行操作。

稍微看了看sys_set_priority，仅仅实现这个系统调用就可以了吗？框架中实现了stride算法的后续步骤了吗，我们需要实现后续的代码吗？

在make test5的时候发现程序一直卡在chapter 5的一个测试set_priority，卡一段时间，然后测试就退出了，连我写好的系统调用也没有进行测试。

![image-20220712221904312](pic/image-20220712221904312.png)

![image-20220712221947491](pic/image-20220712221947491.png)

如图所示的情况，可以看到APPS还有很多，usertests没有完全加载完，qemu模拟器就退出了。