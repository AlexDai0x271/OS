# 实验2：deferred work
## 1.实验目的
通过本实验的学习，掌握信创操作系统内核定制中的workqueue技术，理解其与kernel thread 的区别。
## 2.实验内容
设计并实现一个内核模块，该模块旨在通过work queue和kernel thread两种不同的机制
来完成延迟工作（deferred work），并对比分析这两种实现方式的异同点。 
具体实验步骤如下： 
1. 分别采用work queue和kernel thread两种方式调用10个函数（函数内部打印学号后3
位依次加1的方式区分。例如函数1中打印315，函数2中打印316，以此类推），观察并
记录work queue与 kernel thread在执行函数时的顺序差异。请注意，每个函数应当对
应一个独立的kernel thread，即10个函数需由10个不同的kernel thread分别执行。 
2. 探究work queue中的delayed_work功能，要求在模块加载成功的5秒后打印一条预设
的信息，以验证delayed_work的延迟执行效果。
## 3.代码解析
- 定义数据结构：
  定义了 work_ctx 结构用于存放 work_struct 和当前ID，还定义了一个延迟工作 my_work、一个工作上下文数组 works[10] 和一个任务结构数组 threads[10]。
~~~C
struct work_ctx {
    struct work_struct work;
    int current_id;
};
struct delayed_work my_work;

struct work_ctx works[10];
struct task_struct *threads[10];
~~~
- kthread 执行函数
内核线程的处理函数，打印线程的ID信息。
~~~C
int kthread_handler(void *data) {
    int id = *(int *)data;
    printk(KERN_INFO "kthread : %d\n", 625 + id);
    return 0;
}
~~~
- work queue 执行函数
工作队列的处理函数，打印工作队列的ID信息。
~~~C
void work_queue_handler(struct work_struct *work) {
    struct work_ctx *ctx = container_of(work, struct work_ctx, work);
    printk("work queue : %d", ctx->current_id + 625);
}
~~~
- delayed work 执行函数
这是延迟工作的处理函数，打印 "delayed work!" 信息。以句柄函数的形式传递。
~~~C
void delayed_work_handler(struct work_struct *work) {
    printk("delayed work!\n");
}
~~~
- 内核模块初始化函数
deferred_work_init 函数用于初始化任务队列和内核线程。
INIT_WORK 函数用于创建任务队列
kthread_create, wake_up_process 用于创建线程并唤醒进程输出
~~~C
int deferred_work_init(void) {
    printk(KERN_INFO "deferred work module init\n");
    int i;
    // 初始化workqueue
    for (i = 0; i < 10; i++) {
        works[i].current_id = i;
        INIT_WORK(&works[i].work, work_queue_handler);
        schedule_work(&works[i].work);
    }
    // 初始化kthread
    for (i = 0; i < 10; i++) {
        int *arg = kmalloc(sizeof(*arg), GFP_KERNEL);
        *arg = i;
        threads[i] = kthread_create(kthread_handler, arg, "my_thread_%d", i);
        if (!IS_ERR(threads[i])) {
            wake_up_process(threads[i]);
        } else {
            printk(KERN_ERR "Failed to create thread %d\n", i);
        }
    }
    INIT_DELAYED_WORK(&my_work, delayed_work_handler);
    schedule_delayed_work(&my_work, 5 * HZ);
    return 0;
}
~~~
- 内核模块退出函数
deferred_work_exit 函数用于清理和退出模块。
~~~C
void deferred_work_exit(void) {
    int i;
    for (i = 0; i < 10; i++) {
        kthread_stop(threads[i]);
    }
    flush_scheduled_work();
    printk(KERN_INFO "deferred work module exit\n");
}

module_init(deferred_work_init);
module_exit(deferred_work_exit);
~~~
## 4.实验操作
- 按照实验1的相同方法进行内核模块编译
- 插入模块
~~~bash
insmod deferred_work.ok 
~~~
- 查看内核信息
~~~bash
dmesg
~~~
## 5.实验结果
看到倒数第一行内核在5s后延迟输出delayed work！
![image](https://github.com/user-attachments/assets/1936e55b-8e2c-453f-8715-73b6a3c389b2)

## 6.实验心得
通过这次实验，我归纳出work queue和kernel thread有以下特点与异同点：
work queue允许将工作排队，并在稍后由系统调度执行。
kernel thread用于执行长期运行的任务或需要独立线程来完成的复杂任务。
相同点：
目的：两者都用于在内核中延迟执行任务。

易用性：都提供了内核API，方便开发者使用。

安全性：都支持中断上下文中调度任务，确保系统稳定性。

不同点：
复杂度：工作队列简单，适合短时间的任务；内核线程适合复杂、长时间运行的任务。

资源开销：工作队列资源开销小，因为它们使用现有的内核线程；内核线程开销大，需要分配独立的线程资源。

灵活性：内核线程提供了更高的灵活性，可以处理复杂的同步和通信；工作队列相对简单，适合常见的延迟任务。

执行上下文：工作队列中的任务在现有的工作线程中执行；内核线程拥有自己的执行上下文。

得到结论：工作队列适用于简单、短时的延迟任务，而内核线程适用于复杂、长时间运行的任务。
