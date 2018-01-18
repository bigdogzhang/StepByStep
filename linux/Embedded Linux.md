# What is Real-Time?

对于开发人员而言，实时处理本质上就是“does what you expect it to do when you expect it to do it.“。实时处理是对时间的要求。一般而言，实时处理分为hard real time，firm real time和soft real time。这个分类有很多诟病，但是流传的最广，其实它们之间并没有特别清晰的界限，而且一个系统中往往多种分类并存。具体定义可以参考StackOverflow:<https://stackoverflow.com/questions/17308956/differences-between-hard-real-time-soft-real-time-and-firm-real-time>

下面是部分内容节选：

## Hard Real-Time

> The **hard real-time** definition considers any missed deadline to be a system failure. This scheduling is used extensively in mission critical systems where failure to conform to timing constraints results in a loss of life or property. 

**Examples:**

- Air France Flight 447 crashed into the ocean after a sensor malfunction caused a series of system errors. The pilots stalled the aircraft while responding to outdated instrument readings. All 12 crew and 216 passengers were killed.
- Mars Pathfinder spacecraft was nearly lost when a priority inversion caused system restarts. A higher priority task was not completed on time due to being blocked by a lower priority task. The problem was corrected and the spacecraft landed successfully.
- An Inkjet printer has a print head with control software for depositing the correct amount of ink onto a specific part of the paper. If a deadline is missed then the print job is ruined.

## Firm Real-Time

> The **firm real-time** definition allows for infrequently missed deadlines. In these applications the system can survive task failures so long as they are adequately spaced, however the value of the task's completion drops to zero or becomes impossible. 

**Examples:**

- Manufacturing systems with robot assembly lines where missing a deadline results in improperly assembling a part. As long as ruined parts are infrequent enough to be caught by quality control and not too costly, then production continues.
- A digital cable set-top box decodes time stamps for when frames must appear on the screen. Since the frames are time order sensitive a missed deadline causes jitter, diminishing quality of service. If the missed frame later becomes available it will only cause more jitter to display it, so it's useless. The viewer can still enjoy the program if jitter doesn't occur too often.

## Soft Real-Time

> The **soft real-time** definition allows for frequently missed deadlines, and as long as tasks are timely executed their results continue to have value. Completed tasks may have increasing value up to the deadline and decreasing value past it. 

**Examples:**

- Weather stations have many sensors for reading temperature, humidity, wind speed, etc. The readings should be taken and transmitted at regular intervals, however the sensors are not synchronized. Even though a sensor reading may be early or late compared with the others it can still be relevant as long as it is close enough.
- A video game console runs software for a game engine. There are many resources that must be shared between its tasks. At the same time tasks need to be completed according to the schedule for the game to play correctly. As long as tasks are being completely relatively on time the game will be enjoyable, and if not it may only lag a little.

从本质上讲，任何操作系统都可以称为实时系统，只是实时等级不同而已。我们经常说的RTOS，是指能够实现hard real time层次的操作系统。RTOS会针对时间响应做特殊的设计，采用抢占式调度，尽可能缩短中断响应延迟，缩短任务切换的时间消耗等。RTOS绝大部分都属于嵌入式操作系统（Embedded OS），但是嵌入式操作系统未必都属于RTOS。对于RTOS的详细可以参考维基百科。<https://en.wikipedia.org/wiki/Real-time_operating_system>

# RTOS vs Embedded Linux

下面是StackOverflow上的摘要，很有参考价值。<https://stackoverflow.com/questions/25871579/what-is-the-difference-between-rtos-and-embedded-linux>

Linux is a general-purpose OS (GPOS); its application to embedded systems is usually motivated by the availability of device support, file-systems, network connectivity, and UI support. All these things can be available in an RTOS, but often with less broad support, or at additional cost or integration effort.

Many RTOS are not full OS in the sense that Linux is, in that they comprise of a static link library providing only task scheduling, IPC, synchronisation timing and interrupt services and little more - essentially the scheduling kernel only. Such a library is linked with your application code to produce a single executable that your system boots directly (or via a bootloader). Most RTOS do not directly support the loading and unloading of code dynamically from a file system as you would with Linux - it is all there at start-up and runs until power down.

Critically Linux is not real-time capable. An RTOS provides scheduling guarantees to ensure deterministic behaviour and timely response events and interrupts. In most cases this is through a priority based pre-emptive scheduling algorithm, where the highest priority task ready to run always runs - immediately - pre-empting any lower priority task without a specific yield or relinquishing of the CPU, or completion of a time-slice.

Linux has a number of scheduling options, including a real-time scheduler, but this is at best "soft" real-time - a term I dislike since it is ill-defined, and essentially means real-time, most of the time, but sometimes not. If your application has no need of "hard" real-time, that's fine, but typical latencies in real-time Linux will be in the order of tens or hundreds of microseconds, whereas a typical RTOS real-time kernel can achieve from latencies from zero to a few microseconds. 

Another issue with embedded Linux is that it needs significant CPU resources, perhaps >200MIPS, 32bit processor, ideally with an MMU, 4Mb of ROM and 16MB of RAM to just about boot (which may take several seconds). An RTOS on the other hand can be up in milliseconds, run in less than 10Kb, on microcontrollers from 8-bit up. This can have a significant impact on system cost for volume production despite being ostensibly "free".

There are larger RTOS products that exhibit some of the features of a GPOS such as dynamic loading, filesystems, networking, GUI (for example, in QNX), and many RTOS provide a POSIX API (usually secondary to their native real-time API) for example VxWorks and again QNX, so that a great deal of code developed for Linux and Unix can be ported relatively easily. These larger more comprehensive RTOS products remain scalable, so that functionality not required is not included. Linux in comparison has far more limited scalability.

OS的设计方案是基于OS的应用场景的，GPOS面向一般应用程序，不需要很强的实时性需求，但是却对广泛的硬件支持，应用程序的易开发性要求很高，所以GPOS往往会对底层进行很多的封装，尽可能抽象API，使应用程序脱离底层，更加容易开发。应用程序与底层离得越远，实时性就越差，因为底层事件到达应用程序需要花费的时间就越长。同时，GPOS为了增加用户体验，通常会采取更复杂的资源分配策略，比如，基于时间片轮询的调度方式使用CPU，虚拟内存等。这些都会以牺牲实时性为代价的，因为即使事件到达了应用程序，也未必能及时处理，即使开始处理，也可能随时被其他处理打断，事件什么时候开始处理，什么时候结束，是不可预知的（deterministic behaviour）。而RTOS则不同，它为了满足实时性要求，会使底层与应用程序尽可能接近，即：底层发生的事件，可以几乎无延时的传达到应用程序。同时，采用抢占式调度方法，保证事件可以及时被处理，且处理过程中不会被打断。通过测量事件处理函数，基本可以确定事件的响应时间，结果是可以完全可以测量的（scalable）。RTOS为了支持更多的硬件体系架构，增加应用程序的易开发性，也会增加封装，提供更抽象的API，设计更多的功能，但是这些都是与实时性相冲突的，理论上说，OS越简单实时性越高（没有OS的实时性应该是最高的），OS越抽象实时性越差。如何在支持更多硬件架构，和提供丰富易用功能的同时，尽可能保证实时性，是RTOS的设计难点，也是RTOS竞争的核心所在。当然，稳定性，第三方软件支持，价格，售后服务和技术支持，体积可裁剪性，规格认证等也是相当重要的因素。

# Linux的Real-Time解决方案

<https://www.linux.org/threads/achieving-realtime-linux.11667/#post-41145>

The vanilla (original and unmodified) Linux kernel (from kernel.org) does not support real-time abilities. However, there are ways to make Linux act like a real-time operating system. The easiest method is to use a Linux kernel that contains the "**PREEMPT_RT patch**" (also called the "-rt patch" or "RT patch"). A Linux kernel with the PREEMPT_RT patch supports preemptive multitasking which ensures that each process gets some CPU processing time. Also, the kernel behaves more like a real-time operating system. A Linux kernel with PREEMPT_RT enabled is called a PREEMPT_RT kernel. However, such a kernel is not 100% real-time.

Thankfully, alternatives and additional tools exist. 

Xenomai is a userland-level development framework that provides an interface between programs and the kernel. Xenomai acts as a second kernel executing beside the Linux kernel. However, installing Xenomai is not enough to achieve real-time processing. Developers must re-compile the program or programs that will be run as real-time applications.

- <http://xenomai.org/>
- <http://xenomai.org/introducing-xenomai-3/>
- <http://xenomai.org/2014/08/porting-a-linux-application-to-xenomai-dual-kernel/>

To make Linux operate better as a real-time system, it helps to configure and compile a custom kernel. It is best to disable kernel features that will cause additional latencies and enable features that reduce latencies. For instance, disable "CONFIG_CPU_FREQ" and "CONFIG_CPU_IDLE" so the CPU will not change its clock frequency or change states. The kernel should be configured to control its own power management since the BIOS does so in a way that causes latencies. Also, disable any drivers that are not essential, but make sure that needed drivers are included in the kernel. For instance, on systems lacking SMP, enabling and supporting SMP features will generate unneeded overhead that will cause latencies. Real-time kernels must be designed specifically for the user's hardware and not for a variety of hardware products. For more information on how to configure and compile a custom kernel, check out this Linux kernel guide - 

http://www.linux.org/threads/linux-kernel-reading-guide.5384/

Adeos (Adaptive Domain Environment for Operating Systems) is a nanokernel that runs under the Linux kernel, thus acting as a hypervisor. Adeos runs the Linux system as a real-time operating system. Running Adeos does not require recompiling programs or modifying the Linux kernel.

http://home.gna.org/adeos/

RTAI (Real-Time Application Interface) is a special kernel extension for Linux which offers real-time support. RTAI is POSIX-compliant and open-source. RTAI is a single patch for the Linux kernel that adds various services and a hardware abstraction layer.

https://www.rtai.org/

When selecting a method or a combination of methods to make Linux a real-time system, check the processor type and ensure that the selected methods support the system's CPU and hardware. Some processors and hardware have special features that enable then to function better in real-time environments. For instance, ARM processors in the Cortex-R series (R-Profile) support real-time execution and processing well.

If unable to achieve 100% real-time features in Linux using any of the above methods (alone or in combination), then there are other open-source alternatives. More can be learned about some of these alternative open-source RTOSes in the links below.

- ChibiOS/RT - <http://dcjtech.info/topic/chibiosrt/>
- FreeRTOS - <http://dcjtech.info/topic/freertos-safertos-and-family/>
- GNU eCos - <http://dcjtech.info/topic/gnu-ecos/>
- TRON - <http://dcjtech.info/topic/tron-the-operating-system/>


# 其他参考资料

<https://www.linuxfoundation.org/blog/intro-to-real-time-linux-for-embedded-developers/>

<https://www.embedded.com/design/operating-systems/4371651/Comparing-the-real-time-scheduling-policies-of-the-Linux-kernel-and-an-RTOS->

<https://www.electronicsweekly.com/market-sectors/embedded-systems/analysis-linux-versus-rtos-2008-08/>

