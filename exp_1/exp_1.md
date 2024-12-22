# 实验1：内核API
## 1.实验目的
通过本实验的学习，掌握信创操作系统内核定制中所常用的内核数据结构和函数，具体包
括：i.内核链表；ii. 内核内存的分配释放； iii.内核线程；iv.内核锁；   
## 2.实验内容
设计一个内核模块，并在此内核模块中创建一个内核链表以及两个内核线程。   
- 线程1需要遍历进程链表并将各个进程的pid、进程名加入到内核链表中。
- 线程2中需不断从内核链表中取出节点并打印该节点的元素。     
在卸载模块时停止内核线程并释放资源。  
## 3.代码解析
a. 创建内核链表：首先定义链表结构体，在thread1中创建链表，用for_each_process(task)将各个进程读入链表内，之后将链表链接起来，自旋锁进入互斥区，list_add_tail()内核链表添加函数。  

~~~C
static int thread1_func(void *data) {
    struct task_struct *task;

    for_each_process(task) {
        struct pid_node *my_node;
        my_node = kmalloc(sizeof(struct pid_node), GFP_KERNEL);
        if (!my_node) {
            printk(KERN_ALERT "Failed to allocate memory for my_node\n");
            continue;
        }

        my_node->pid = task->pid;
        strncpy(my_node->comm, task->comm, TASK_COMM_LEN);

        spin_lock(&lock);
        list_add_tail(&my_node->list, &my_list);
        spin_unlock(&lock);

        ssleep(2);
    }

    return 0;
}
~~~



b. 遍历内核链表：定义thread2函数，使用list_first_entry找到链表头，并读取链表用printk函数将PID和NAME显示出来

~~~C
static int thread2_func(void *data) {
    struct pid_node *current;

    while (!kthread_should_stop()) {
        spin_lock(&lock);

        if (!list_empty(&my_list)) {
            current = list_first_entry(&my_list, struct pid_node, list);
            list_del(&current->list);
            spin_unlock(&lock);

            printk(KERN_INFO "PID: %d, Name: %s\n", current->pid, current->comm);
            kfree(current);
        } else {
            spin_unlock(&lock);
        }

        ssleep(2);
    }

    return 0;
}

~~~

c. 初始化函数：定义kernel_Init函数，初始化函数
~~~C
int kernel_module_init(void) {
    printk(KERN_INFO "List and thread module init\n");
    INIT_LIST_HEAD(&my_list);
    spin_lock_init(&lock);

    thread1 = kthread_run(thread1_func, NULL, "thread1");
    if (IS_ERR(thread1)) {
        printk(KERN_ERR "Failed to create thread1\n");
        return PTR_ERR(thread1);
    }

    thread2 = kthread_run(thread2_func, NULL, "thread2");
    if (IS_ERR(thread2)) {
        printk(KERN_ERR "Failed to create thread2\n");
        return PTR_ERR(thread2);
    }

    return 0;
}
~~~

d. 退出进程函数：定义kernel_exit函数，使用kfree把链表删除，防止内存溢出
~~~C
void kernel_module_exit(void) {
    struct pid_node *current;

    // Stop thread1
    if (thread1)
        kthread_stop(thread1);

    // Stop thread2
    if (thread2)
        kthread_stop(thread2);

    // Clear the list
    while (!list_empty(&my_list)) {
        current = list_first_entry(&my_list, struct pid_node, list);
        list_del(&current->list);
        kfree(current);
    }

    printk(KERN_INFO "List and thread module exit\n");
}

module_init(kernel_module_init);
module_exit(kernel_module_exit);
~~~

## 4.实验操作
- 将本地编辑后的kernel_module.c文件上传到GDB-POD
~~~bash
scp -P 30573 D:/OS/kernel_module.c user22009200625b@10.168.59.90:/config 
~~~
该代码取得GDB-POD的权限后将本地文件传入/config文件夹内  
- 将GDB-POD中的kernel_module.c上传到虚拟机内
~~~bash
scp /config/kernel_module.c root@192.168.3.173:/root/lab/1 
~~~
该代码段取得虚拟机的root权限后将GDB-POD内的文件上传到虚拟机内  
- make内核模块
~~~bash
cd /root/lab/1
make
~~~
- 进入到虚拟机内，打开/root/lab/1文件夹内，创建并且安装内核模块，同时检查安装  
~~~bash
sudo insmod kernel_module.ko
lsmod | grep kernel_module
~~~
- 安装完成后显示当前运行的进程
~~~bash
dmesg
~~~
## 5.实验结果
我们发现dmesg显示出来了系统内目前正在运行的所有进程，这些有很多模块在运行到末端我们可以看到我们创建的模块正在运行。

![image](https://github.com/user-attachments/assets/6f9bfd37-986c-48a4-b123-e0bc0615bf87)

![image](https://github.com/user-attachments/assets/96f4d65f-f75a-4b2e-81c3-3050697f51c7)

![image](https://github.com/user-attachments/assets/6513c181-ad88-4312-bdaa-74548eca5610)

## 6.实验心得
通过本次实验，我学到了8个Linux的内核API函数，学会了GDB-POD的基本文件操作，能够将文件从本地上传到虚拟机和从虚拟机内下载到本地。本次实验还让我清晰地看到了线程的存储方法和创建方法，对通用操作系统有了更好的了解，为后续的实验和实验环境起了个好头。
