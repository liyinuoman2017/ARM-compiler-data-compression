# ARM-compiler-data-compression
ARM编译器数据压缩

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/fadde3198e034ad7b64142aff8908a73.png)

**调试一个平平无奇程序，居然引出了一个惊天大幂幂！可能有99%的单片机工程师都不知道ARM编译器的这个秘密！**

## 1、故事的开始

秋高气爽的某一天，我在调试一个STM32程序，不经意间我看到了一个变量，就是这一个回眸，牵扯出后面的故事。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/c57e0a0262fc49cf8afa43f397759fec.png)



它就是magic_number ，这个magic_number是一个全局静态变量，是一个32bit无符号数，它的数值为 0x20000037。

```bash
uint32_t magic_number = 0x20000037;
```


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/3545e0b6b0ca4926b2e255a328ab178b.png)




也不知道当时是抽什么疯，我想在编译出的HEX文件中找到这个变量的数值0x20000037，当时我打开HEX文件搜索20000037的时候，居然没有！难道是数据大端小端模式，然后我又搜索了37000020，**竟然还是没有！**

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/27e4ce2e4e6e40ce8c3096a1ddacfc19.png)

然后我**缩小搜索范围**，搜索0037和3700（防止出现20000037被分配到了两行），居然还是没有找到20000037！

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a145308f6f91421494faa3bb6cb4a862.png)



非常奇怪！**作为一个常量值0x20000037应该保存在可执行文件ELF中的.data**段（存放静态变量的区域），然后被存到HEX文件中。这涉及到一个重要概率：**分散加载**。

> **定义：**分散加载是ARM开发中一种重要的内存管理和映像文件生成方法，通过scatter文件（分散加载描述文件）来指定代码和数据在内存中的分布。
> **在程序执行过程中**，__scatterload()函数负责将RW/RO输出段从装载域地址复制到运行域地址，并完成ZI运行域的初始化工作。


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f85d1ff55c664f2eb54999acb989d147.png)


理论上来说，在HEX文件中肯定有0x20000037，要不然**如何始化变量magic_number**？（**程序中该变量被反复使用不可能被优化掉**）这一结果让我陷入迷茫，我在网上一通乱查，结果是一无所有。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f59e8460b9cc487b84d5f8f31bc3fee3.png)



## 2、寻找答案

既然在网上找不到正确答案，那么只能靠自己来解决问题，我又重新定义了另外一个变量liwei_number，它的值为0x13141790。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f2f6d008eedb49fda2b7562032f05ae6.png)


编程工程得到HEX文件，**查找变量13141790，居然找到了！**

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/65a48554038841ab9728d7005fa463da.png)



紧接着我又把变量liwei_number设置为了0x20000037 ,编译得到HEX文件，查找居然又找不到了！然后把变量liwei_number设置为了0x13141790 ,编译得到HEX文件,查找又能找到了！
**难道出现了量子效应？薛定谔的猫？**

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/490320cf26a040088777e34c6b541f62.png)


虽然很困惑，但是还是要继续定位问题。于是我将liwei_number改成了liwei_number_buff，包含了10个32bit数。

```bash
uint32_t magic_number = 0x20000037;
uint32_t liwei_number_buff[10] = 
{
0x13141790,0x13141780,0x13140000,0x13141760,0x13141700,
0x13140000,0x13140000,0x13140000,0x13140000,0x53131482,	
};
```


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b84158b90c4e41f0b8fe33c3c7c5110f.png)


编译编译得到HEX文件,查找数据，**只能找到2个变量能对应上数值**。虽然只有两个变量能对应上，但是从HEX数据中还是能发现一定的位置关系。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a98168eda36241f4a376b026d5f8ed79.png)



**难道编译器对某些数据进行了处理**？进行了什么处理？增加了校验？那为什么有些变量没有变化可以找到，有些变化被处理了无法找到？
至少目前我得出了一个结论或者判断：**编译器会根据一定的规则改变部分数据。**


接下来继续修改 liwei_number_buff 数值。

```bash
uint32_t magic_number = 0x20000037;
uint32_t liwei_number_buff[10] = 
{
0x13141790,0x11111111,0x2222222,0x13141760,0x33333333,
0x44444444,0x13141794,0x5555555,0x66666666,0x53131482,	
};
```

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7fbf3f20f63e46788c0c286d03394c26.png)



编译编译得到HEX文件,查找数据，**能找到3个变量能对应上数值**。 不难发现6个重复的数据都无法找到，而3个不重复的数据都可以找到。
目前我得出了一个判断：**编译器会根据一定的规则改变出行重复的数据。** 

> **改变重复的的数据，是不是对重复的数据进行了某种数据压缩？**
> 


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/1d9e752d97054f71b13610ba5b592280.png)



## 3、对比测试

keil5用的是ARM公司的编译器ARMCC，是不是只有ARMCC会对HEX中的静态变量初始值进行处理？**其它编译器也会有同样的操作吗？**
**用gcc编译器和makefile构建了一个STM32的编译环境**，不会的朋友可以参考我的另外一篇文章：

```bash
https://blog.csdn.net/li_man_man_man/article/details/144007577?spm=1001.2014.3001.5502
```

搭建好工程，在函数中定义了同样的变量liwei_number_buff，编译工程得到HEX文件。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/345c817205b341929219ca2cc9a521a1.png)



查找数值，居然找到了10个变量的数值。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/4852669d28fc4406b21b500b33725977.png)


通过对比实验，我得到了一个初步的结论：**armcc编译器对重复数据进行了压缩。**

## 4、证明结论

既然是armcc对数据进行了压缩，那么在armcc的用户手册中应该有相应的说明，打开arm编译器用户手册进行查找。


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a46be950f2e94a1a869149fdf35a7926.png)


打开《**Arm Compiler User Guide**》查找，翻了一遍，内容太多，没找到什么有效信息。于是我抱着试一试的心态搜索了“**压缩**”（**compress**），居然出现了好几处，仔细查看内容每一处的内容，其中有一处终于证明的我的想法！

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/9c4d3aa14e644698864594f8e35f2a97.png)



> For RW data overlays, it is necessary to disable RW data compression
> for the whole project. You can disable compression with the linker
> command-line option --datacompressor off, or you can mark the
> execution region with the attribute NOCOMPRESS.
> **译文：**
> 对于RW数据叠加，有必要在整个项目中禁用RW数据压缩。您可以使用链接器命令行选项--datacompressor-off禁用压缩，也可以使用属性NOCOMPRESS标记执行区域。
> 可以在keil工具中关闭压缩功能


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/938e7e9dbfac4e41859839251a77b0ea.png)


```bash
--datacompressor=off
```

在keil工程中设置中，关闭了压缩功能，重新编译工程得到HEX，查找数据10个数据的数值全部都能找到！说明就是编译器对全局静态变量进行了压缩。


![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2c15dadabe014585a689ff7f6922f365.png)



## 5、深入研究

前面证明了ARMCC编译器对数据进行了压缩，那么问题来了：

> 1、ARMCC如何对数据进行压缩
>  2、如何完成数据解压

#### 数据压缩

继续看文档《compiler_reference_guide》和《compiler_user_guide》

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/f40196d599064a068fbe7e35177f6d21.png)


搜索compress关键字，在《compiler_reference_guide》可以找到关于压缩数据的算法，根据手册可知：ARMCC压缩数据使用的是LZ77算法。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/bc9e70c3151246849aff122812d77e0c.png)

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/78123bcdea574a18970ff8786c64bbf3.png)



**那么LZ77算法是个什么？**



> **基本概念**
> LZ77算法是一种基于字典的编码方法，通过查找数据中重复出现的字符串，并用较短的标记来替代这些字符串，从而实现压缩。其核心思想是利用数据的重复结构信息来进行数据压缩，即如果字符串中的信息在之前出现过，则只需要指出其之前出现过的位置，便可以用相对索引来表示这些词。
> 

> **实现方式**
>  LZ77算法的实现依赖于一个滑动窗口（Sliding Window）和一个预读缓冲器（Read Ahead Buffer）。滑动窗口是一个历史缓冲器，用于存放输入流的前n个字节的有关信息。预读缓冲器则用于存放输入流中即将被编码的n个字节。算法在滑动窗口中查找与预读缓冲器中最匹配的数据，如果找到匹配的数据，则输出一个包含匹配长度和距离的数组（或三元组），然后滑动窗口和预读缓冲器继续向前移动；如果没有找到匹配的数据，则直接输出预读缓冲器中的字符，并滑动一个单位。

下面这篇文章可以帮助大家理解LZ77算法

```bash
https://www.zhihu.com/question/30112322/answer/3358054159
```

#### 数据解压

既然编译器使用了LZ77算法对数据进行了压缩，那数据解压是如何完成的？从《compiler_user_guide》手册中可以找到系统启动初始化流程图如下。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/cc53a79ebfdb4f3c8cd4bff615439599.png)


根据启动初始化流程图可知，处理器在启动后执行了cpoy/decompress RW data ，由此可见编译在生成可执行文件ELF的时候，链接进了一个LZZ解压算法的小程序，在处理器启动后执行RW段初始化，完成初始化之后跳转到mian函数。

## 6、对比GCC和ARMCC

最后对比GCC和ARMCC这两个编译器的差异。

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a6c6fd548d4c4942a8803a7cf9f85cb2.png)


GCC和ARMCC在嵌入式开发中有以下主要区别‌：

> ‌**开发环境‌：**
> ‌GCC‌：GCC是GNU项目的一部分，是一个开源的编译器，支持多种平台，包括Windows、Mac和Linux。它提供了丰富的文档和社区支持，适合需要跨平台开发的用户‌1。
> ‌ARMCC‌：ARMCC是ARM公司开发的嵌入式工具链，通常与Keil
> MDK集成开发环境（IDE）一起使用。ARMCC是闭源的商业产品，适合需要高效编译和优化但预算较高的用户‌。



> **编译效率‌： ‌**
> ARMCC‌：ARMCC在编译效率上通常优于GCC，编译出的代码更小，但这是以牺牲开源和社区支持为代价的‌。
> ‌GCC‌：虽然编译效率可能不如ARMCC，但其开源和跨平台的特性使得它在社区支持和文档方面更具优势‌。



**创作不易希望朋友们点赞，转发，评论，关注!**

**您的点赞，转发，评论，关注将是我持续更新的动力!**


**CSDN：https://blog.csdn.net/li_man_man_man**

**今日头条：https://www.toutiao.com/article/7149576260891443724**
