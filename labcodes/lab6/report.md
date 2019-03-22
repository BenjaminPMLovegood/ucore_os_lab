# 练习0

时钟中断处：

```c
    case IRQ_OFFSET + IRQ_TIMER:
        ++ticks;
        sched_class_proc_tick(current);
        break;
```

`alloc_proc`处：

```c
        proc->rq = NULL;
        list_init(&(proc -> run_link));
        proc->time_slice = 0;
        proc->lab6_run_pool.left = proc->lab6_run_pool.right = proc->lab6_run_pool.parent = NULL;
        proc->lab6_stride = 0;
        proc->lab6_priority = 0;
```
# 练习1

`sched_class`中有五个函数指针：

`init`：初始化调度器。

`enqueue`：将进程加入rq。

`dequeue`：将进程移出rq。

`pick_next`：从rq中选择下一个要执行的进程。

`proc_tick`：每个时钟中断时被调用。



RR（Round Robin）下的调度过程：

通过调用`schedule`进行调度。`schedule`会关中断，调用`sched_class`中的函数将当前进程入队并选择进程出队执行，最后开中断。

RR算法的`enqueue`函数将进程放到队列尾并重置时间片。

RR算法的`pick_next`函数直接选择队首元素。

RR算法的`dequeue`函数直接删除相应元素。

RR算法的`proc_tick`函数将时间片计数减一，当时间片计数降至0时标记需要调度。



“多级反馈队列调度算法”的实现：

首先需要在`proc_struct`中加入优先级字段。调度器按优先级维护多个rq。

`enqueue`时：如果不是第一次入队，则降低一个优先级。然后根据优先级入队。注意低优先级的进程需要有更长的时间片，常见的实现设置相邻优先级的时间片长度比例为2。

`dequeue`时：直接删除即可。

`pick_next`时：从高到低检查各优先级队列，选择第一个非空队列的队首。有的实现中为了避免特殊情况下低优先级队列长期饥饿，会设定一个概率选择较低优先级的进程。

`proc_tick`同RR。

# 练习2

Stride Scheduling算法为每个进程维护两个值：stride和pass，rq则采用按stride升序的优先队列；每次调度器选择一个进程执行时会将它的stride加上pass。这里pass设置为`BIG_STRIDE / priority `。.

`enqueue`函数将进程加入优先队列。

`pick_next`函数直接选择队首元素，并且将其stride加上pass。

`dequeue`函数直接删除相应元素。

`proc_tick`函数同Round Robin。