# 实验四 操作系统裁减以及操作系统

## 0.1 编译内核    
在进入虚拟机的/root/lab/4/kernel之后，我们要进行内核编译  
~~~shell
  make menuconfig
~~~
等待编译完成，会进入编辑界面
在编辑界面向下移动会看到“filesystem”按下回车进入filesystem界面  
将光标移动到ext4处，按下空格取消选择，再将光标移动到到btrfs项，同样按下空格将其选择  
此时，按下键盘的右箭头，选择Save然后按下回车，将其保存为.config文件  
再按下Exit推出到Terminal，输入以下代码：
~~~shell
    make -j $(nproc) Image.gz
~~~
**参数解释：**
> -j: 这个选项用于指定可以并行运行的任务数量。并行编译可以显著加快构建过程。  
> $(nproc): 用于返回当前系统中的处理器核心数。将它与 -j 选项结合使用，make工具将尽可能多地并行执行任务，最大限度地利用所有可用的处理器核心。
> Image.gz: 是我们需要的编译好的内核镜像文件名称
完成内核编译之后可以观察到/root/lab/4文件夹内的“Image.gz”文件变为绿色，证明镜像编译完成  

## 0.2 编译文件系统
**在编译文件系统之前，我们需要查看虚拟机的架构，避免编译错误**
使用到指令
~~~shell
uname -r
~~~

![图片](https://github.com/user-attachments/assets/85b59899-b661-4f5a-aea4-7522343a3f8b)

不难看出我们的虚拟机是arm64架构的虚拟机，因此在接下来的操作中需要选择**AArch64**选项  
进入/root/lab/4/buildroot-2024.02.6，同样键入  
~~~shell
make menuconfig
~~~
进入到配置界面

![图片](https://github.com/user-attachments/assets/3f8fe83e-a52f-4be9-87ac-a6d0d4f8585d)

选择到“Target Options”，进入如下界面

![图片](https://github.com/user-attachments/assets/297e834b-a02c-400f-8737-b7ea4f406587)

此时，我们需要选择第一个选项，并且选择到(AAch64(little endian))如果对编译时间没有要求可以选择(AAch64(big endian))选项

![图片](https://github.com/user-attachments/assets/7120e5c9-0b3b-4c67-abcb-44bf6ffab6ed)

进入Filesystem Images选项内

![图片](https://github.com/user-attachments/assets/77575a84-8e2d-4574-9134-9f540ab3bac0)

完成如下配置，其中filesystem size选项建议选择1024m以上，这样在运行时将会流畅一些
> 理论上btrfs格式要求文件系统大小不低于 109MB

![图片](https://github.com/user-attachments/assets/edda5b6f-52f6-4424-bbca-3d6eb8b8b9e3)

接下来我们需要查找到bash和vim并将其添加到文件目录内  
建议使用“/”来查找

bash在如下路径：

![图片](https://github.com/user-attachments/assets/f638948a-8859-4d09-a795-b836fb6cbcda)

找到并勾选上

![图片](https://github.com/user-attachments/assets/05514334-23ac-4bb1-bc34-cd7cc8bee760)

vim在如下路径：

![图片](https://github.com/user-attachments/assets/6bf918ad-b18e-4964-9492-558d9e7738b7)

找到并且勾选上，然后保存并退出
老方法全速编译
~~~shell
  make -j $(nproc)
~~~






