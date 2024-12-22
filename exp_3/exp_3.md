# 实验3：edu设备驱动
## 1.实验目的
通过本实验的学习，掌握信创操作系统内核定制中所常见PCI设备驱动适配技术。
## 2.实验内容
本次实验旨在让学生深入理解并实践edu设备驱动的开发。实验中，我们将提供edu设备驱
动的框架代码，学生需在此基础上完成关键代码的实现。具体实验要求如下： 
1. 补全框架中的TODO位置的缺失代码，包含TODO的函数如下所示： 
◦ edu_driver_probe   
▪ 为edu_dev_info实例分配内存   
▪ 将BAR的总线地址映射到系统内存的虚拟地址   
◦ edu_driver_remove   
▪ 从设备结构体dev中提取edu_dev_info实例   
▪ 补全iounmap函数的调用的参数   
▪ 释放edu_dev_info实例   
◦ kthread_handler   
▪ 将用户传入的变量交给edv设备进行阶乘计算，并读出结果，注意加锁。结果
放入user_data中的data数据成员中时，需要确保读写原子性   
◦ edu_dev_open   
▪ 完成filp与用户进程上下文信息的绑定操作   
◦ edu_dev_release   
▪ 释放edu_dev_open中分配的内存   
◦ edu_dev_unlocked_ioctl  
▪ 用户通过ioctl传入要计算阶乘的数值，并读取最后阶乘的结果。计算阶乘使
用内核线程，线程代码放在kthread_handler中   
2. 实现驱动程序的ioctl调用处理功能。该调用需接收一个整型参数。当驱动程序接收
到用户的ioctl调用后，需创建一个内核线程。在该内核线程中，利用edu设备的阶乘
功能对传入的整型参数进行计算，并将计算结果存储于驱动程序中，以便用户进程后
续获取。   
3. 驱动程序需具备识别不同进程调用的能力，确保将计算结果正确返回给对应的调用进
程。   
4. 编写C语言应用程序，通过调用edu驱动的ioctl接口进行操作。首先，设置参数
cmd值为0，输入待计算的数值。等待一定时间后，将参数cmd值更改为1，再次调用
ioctl 接口，以获取设备计算完成的结果。

## 3.实验代码解析
1.edu设备发现函数"edu_driver_probe()"

<font size="4">

  - **关键技术1：设备使能函数**

  ~~~C
  int pci_enable_device(struct *device dev)
  ~~~
    利用返回值来判断设备是否挂载在pci总线上，0为使能不成功，1为使能成功  

    > from:https://www.kernel.org/doc/Documentation/PCI/pci.txt



  - **关键技术2：内存申请函数**
  
  ~~~C
  int kzalloc(size_t size, gfp_t flags)
  ~~~
    利用返回值判断内存申请与清理是否完成  

    > from:https://www.kernel.org/doc/html/latest/core-api/memory-allocation.html

  - **关键技术3："pci_resource"函数组**
  
  ~~~C
  unsigned long pci_resource_start(struct pci_dev *dev, int resno);
  unsigned long pci_resource_len(struct pci_dev *dev, int resno);
  unsigned long pci_resource_flags(struct pci_dev *dev, int resno);
  ~~~
  
  <p style="line-height: 2;">
    使用以上三个函数将pci资源分配给字符设备edu：  
    start函数将PCI总线号赋给io成员变量  
    len函数规定了寻址范围  
    flags函数将首地址赋给成员变量flag  
  </p>

    > from:https://www.kernel.org/doc/html/v4.12/driver-api/pci.html
  


  - **关键技术4：I/O内存申请函数**

  ~~~C
  int pci_request_regions(struct pci_dev *pdev, const char *res_name);
  ~~~

  <p style="line-height: 2;">
    申请PCI字符设备edu的I/O和内存资源，相当于在注册表内添加该设备  
    <div style="border: 1px solid red; padding: 10px;">
      <strong>注意:</strong> 不要在res_name字符串带有任何中文，在首次编写时因为乱码导致无法发现设备
    </div>
  </p>

    > from:https://docs.kernel.org/5.10/driver-api/pci/pci.html
  


  - **关键技术5：BAR总线地址映射到系统内存的虚拟地址**
  ~~~C
  void __iomem *ioremap(unsigned long offset, unsigned long size)
  ~~~

  <p style="line-height: 2;">
    将设备的物理地址映射到系统内存的虚拟地址，就是将设备虚拟化，保证设备与计算机本地系统的独立性
    <div style="border: 1px solid red; padding: 10px;">
      <strong>注意:</strong> 小心越界问题，由于本虚拟机有8GB的内存所以不用担心该问题，但是在开发嵌入式设备上的驱动中要格外注意需要调用iounmap来释放内存，避免内存泄漏
    </div>
  </p>  

    > from:https://www.kernel.org/doc/html/latest/driver-api/device-io.html

  - **关键技术6：设置驱动私有数据**
  ~~~C
  void *pci_set_drvdata(struct pci_dev *pdev, void *data)
  ~~~
  在驱动程序的生命周期内存储和访问特定的数据  
  
  > from:https://docs.kernel.org/5.10/driver-api/pci/pci.html  
</font>
2.edu设备移除函数"edu_driver_remove()"
<font size='4'>

  - **关键技术1：I/O内存释放函数**
  ~~~C
  void *pci_release_regions(struct pci_dev *pdev)
  ~~~
  将I/O设备所申请的I/O内存释放，将设备清除出/etc/proc和/etc/devices

  >from:https://docs.kernel.org/5.10/driver-api/pci/pci.html

  - **关键技术2：设备数据读取**
  ~~~C
  void *pci_get_drvdata(struct pci_dev *pdev)
  ~~~
  获取到设备已经读取到的数据，将数据地址返回

  >from:https://docs.kernel.org/5.10/driver-api/pci/pci.html
</font>

3.kthread句柄函数"kthread_handler()"
<font size='4'>

  - **关键技术：原子化存储**
  ~~~C
  void atomic64_set(atomic64_t *v, long long i)
  ~~~
  将i原子化存储到内存中
  
  <div style="border: 1px solid red; padding: 10px;">
      <strong>注意:</strong> 赋值时一定要上锁和关锁，不然会显示内存错误！
  </div>

  >from:https://www.kernel.org/doc/html/latest/core-api/wrappers/atomic_t.html
</font>

4.io控制函数"edu_dev_unlocked_ioctl()"
<font size='4'>

  - **关键技术：启动线程**
  ~~~C
  void kthread_run(int *function,void *data,char *name)
  ~~~
  启动线程

  >from:https://www.kernel.org/doc/html/v5.15/driver-api/basics.html?highlight=wake_up_process

  - **设计思路**
  使用结构体user_data与thread_data中的数据指向task_struct内，再给task分配内存，task生成线程然后调用"kthread_handler"完成阶乘
</font>

5.驱动程序初始化
<font size='4'>

  - **关键技术：注册设备**
  ~~~C
  int register_chrdev(char* major, char* name, static struct file_operations *fops)
  int pci_register_drive(struct pci_drive *pcid)
  ~~~
  在注册表上注册字符设备并且在pci表上注册设备，在/etc/proc中可以看到有major号和name名字的字符型设备会被添加，
  在/etc/drives里会看到edu设备的名字

  >from:https://www.kernel.org/doc/html/v5.15/driver-api/pci_drive

</font>

## 4.实验操作
完成.c文件之后执行make install将内核模块注册到/dev和/proc文件夹内，注册为字符设备  
然后调用gcc编译器编译user_space.c文件实例化之后加载内核模块，然后执行exe文件，得到结果  
## 5.实验结果
