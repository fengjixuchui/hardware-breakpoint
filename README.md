<!--
 * @Author: zwf 240970521@qq.com
 * @Date: 2023-08-23 22:03:49
 * @LastEditors: zwf 240970521@qq.com
 * @LastEditTime: 2023-08-23 22:14:39
 * @FilePath: /hardware-breakpoint/README.md
 * @Description: 这是默认设置,请设置`customMade`, 打开koroFileHeader查看配置 进行设置: https://github.com/OBKoro1/koro1FileHeader/wiki/%E9%85%8D%E7%BD%AE
-->
# 硬件断点是什么

硬件断点是cpu自带的用于调试程序的断点，分为执行断点和内存断点。比如ARM64架构，它有4个内存断点和6个执行断点。执行断点是最常见的断点类型，就是在某个程序地址处加断点，CPU运行到这里时就会触发，而内存断点是，在某个内存地址处加断点，当CPU访问这块内存时就会停下来。硬件断点与软件断点的不同之处也在于此，硬件断点可以使用内存断点，而软件断点无法实现内存断点。Linux内核里也自带硬件断点模块，只需要开启如下选项即可。
`CONFIG_HAVE_HW_BREAKPOINT=y`
`CONFIG_PERF_EVENTS=y`

# 为什么要把硬件断点独立成一个驱动

Linux内核自带的硬件断点是跟很多业务捆绑的，开启硬件断点就要把这些业务都打开。这需要重新编译内核，重新编译所有的驱动。内核还好说，重新编译所有驱动，是挺痛苦的。而独立成驱动后就简单了，只需要把驱动插入就行。
还有就是独立成驱动后，我们可以很方便的对其进行定制修改，比如支持kgdb，触发硬件断点后，使其进入kgdb调试状态，便于我们调试跟踪程序运行过程。

## 硬件断点依赖的内核修改

硬件断点驱动的编译依赖一些内核的接口才能实现，所以内核也是有一些修改的，但是可以放心，修改量是很少的，只需要导出一些接口即可。
1. `unregister_step_hook`
2. `disable_debug_monitors`
3. `kernel_active_single_step`
4. `hook_debug_fault_code`
5. `kernel_enable_single_step`
6. `read_sanitised_ftr_reg`
7. `kernel_disable_single_step`
8. `register_step_hook`
9. `show_regs`
10. `enable_debug_monitors`

# 硬件断点驱动详解

驱动共分为3部分，分为底层实现，多核CPU的断点同步管理，API接口开放。
## 底层实现
底层实现包括设置对应的硬件断点寄存器，监控设置的地址，注册调试异常`hook`等，注册单步异常`hook`。

## 多核心CPU的断点同步管理

上述的底层实现只是实现单核`CPU`断点的设置。假如在一个多核心的`CPU`上要监视某个内存地址的读写行为，那肯定是要监控所有`CPU`对该地址的读写行为，只监控当前`CPU`的访问肯定是不够的。所以该部分是对上述底层的封装，对所有运行的核心进行遍历，通过多核同步机制(`smp_call_function_single`)，给所有核心打上/删除相同的断点。

## API接口开放

该部分集中管理所有通过API接口设置的断点，对外开放设置断点的接口，可选择和`KGDB`联动，实现断点触发后的源码级调试。分为如下几个部分：
1. 直接通过地址设置断点。
2. 如果内核支持符号查询功能的话，可以支持通过符号名字设置断点。
3. 在启用`kgdb`联动功能后，可以通过`kgdb`的设置断点功能设置硬件断点。
4. 在`shell`下通过`proc`调试文件来设置断点。

断点监测的虚拟地址范围最大为2Gb。当断点被触发时，会打印出当时操作这块内存的堆栈信息，便于问题定位，或是代码的深度理解。若是启用了`KGDB`的联动功能，则会在打出堆栈信息后，进入`KGDB`状态，等待远程主机连接调试。`KGDB`的原来及使用可以参考我的前一篇文章[KGDB原理分析及远程挂载调试ARM64内核](https://blog.csdn.net/qq_38384263/article/details/132290737?spm=1001.2014.3001.5502)。