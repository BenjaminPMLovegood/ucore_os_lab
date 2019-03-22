# 练习0

diff & patch之后，需要修改的地方只有一处，在时钟中断处，改为：

```c
    case IRQ_OFFSET + IRQ_TIMER:
        ++ticks;
        run_timer_list();
        break;
```

`make grade`此时已经能全部通过。

# 练习1

## 内核级信号量的设计描述

内核级信号量的实现参见`kern/sync/sem.h`：

```c
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;

void sem_init(semaphore_t *sem, int value);
void up(semaphore_t *sem);
void down(semaphore_t *sem);
bool try_down(semaphore_t *sem);
```

`semaphore_t`结构体保存了信号量的值和等待队列。`__up`和`__down`则是核心实现。

```c
static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        wait_t *wait;
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
            sem->value ++;
        }
        else {
            assert(wait->proc->wait_state == wait_state);
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
        }
    }
    local_intr_restore(intr_flag);
}

static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    if (sem->value > 0) {
        sem->value --;
        local_intr_restore(intr_flag);
        return 0;
    }
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);
    local_intr_restore(intr_flag);

    schedule();

    local_intr_save(intr_flag);
    wait_current_del(&(sem->wait_queue), wait);
    local_intr_restore(intr_flag);

    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}
```

`__up`完成了V操作。首先关中断，然后判断，如果等待队列为空，则直接将value加一；若不为空则唤醒第一个等待的进程并将其移除，最后开中断。

`__down`完成了P操作。首先关中断，然后判断，如果value大于0，则将value减一直接返回；否则将进程加入等待队列，标记为等待，开中断并触发调度。在被唤醒之后，则和value大于0时行为一样。

用户态信号量机制可以直接把内核的信号量机制通过系统调用封装实现。不同之处是用户态信号量需要系统调用进入内核态。

# 练习2

实现如下：

```c
// monitor.c
// Unlock one of threads waiting on the condition variable. 
void 
cond_signal (condvar_t *cvp) {
   //LAB7 EXERCISE1: 2016011349
   cprintf("cond_signal begin: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);  
  /*
   *      cond_signal(cv) {
   *          if(cv.count>0) {
   *             mt.next_count ++;
   *             signal(cv.sem);
   *             wait(mt.next);
   *             mt.next_count--;
   *          }
   *       }
   */   
    if (cvp->count > 0) {
        cvp->owner->next_count++;
        up(&(cvp->sem));
        down(&(cvp->owner->next));
        cvp->owner->next_count--;
    }
   cprintf("cond_signal end: cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}

// Suspend calling thread on a condition variable waiting for condition Atomically unlocks 
// mutex and suspends calling thread on conditional variable after waking up locks mutex. Notice: mp is mutex semaphore for monitor's procedures
void
cond_wait (condvar_t *cvp) {
    //LAB7 EXERCISE1: 2016011349
    cprintf("cond_wait begin:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
   /*
    *         cv.count ++;
    *         if(mt.next_count>0)
    *            signal(mt.next)
    *         else
    *            signal(mt.mutex);
    *         wait(cv.sem);
    *         cv.count --;
    */
    cvp->count++;
    if (cvp->owner->next_count > 0) {
        up(&(cvp->owner->next));
    } else {
        up(&(cvp->owner->mutex));
    }
    down(&cvp->sem);
    cvp->count--;
    cprintf("cond_wait end:  cvp %x, cvp->count %d, cvp->owner->next_count %d\n", cvp, cvp->count, cvp->owner->next_count);
}
```



```c
// check_sync.c
// implemented by semophore
void phi_take_forks_condvar(int i) {
     down(&(mtp->mutex));
//--------into routine in monitor--------------
     // LAB7 EXERCISE1: 2016011349
     // I am hungry
     // try to get fork
     state_condvar[i] = HUNGRY;
     phi_test_condvar(i);
     if (state_condvar[i] != EATING) {
         cond_wait(&mtp->cv[i]);
     }
//--------leave routine in monitor--------------
      if(mtp->next_count>0)
         up(&(mtp->next));
      else
         up(&(mtp->mutex));
}

void phi_put_forks_condvar(int i) {
     down(&(mtp->mutex));

//--------into routine in monitor--------------
     // LAB7 EXERCISE1: 2016011349
     // I ate over
     // test left and right neighbors
     state_condvar[i] = THINKING;
     phi_test_condvar(LEFT);
     phi_test_condvar(RIGHT);
//--------leave routine in monitor--------------
     if(mtp->next_count>0)
        up(&(mtp->next));
     else
        up(&(mtp->mutex));
}
```

用户态条件变量机制也可以直接把内核的条件变量机制通过系统调用封装实现。不同之处也是用户态条件变量需要系统调用进入内核态。

可以。~~直接将信号量的代码人工内联到条件变量的代码中。~~