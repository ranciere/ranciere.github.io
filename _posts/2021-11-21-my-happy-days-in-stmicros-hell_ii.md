---
layout: post
title:  "My happy days in STMicro's hell: Part 2"
image:
  path: /docs/assets/images/power_electronics_image.jpg
  thumbnail: /docs/assets/images/power_electronics_thumbnail.jpg
  caption: "Photos by Gefi (https://www.facebook.com/tamas.g.varga)"
---

In the previous [part](/2021/11/20/my-happy-days-in-stmicros-hell.html), I've explained why I chose the STM32 microcontroller ecosystem for our brand new carillon Master Clock. In this post, I share my experiences (spoiler alert: horror).

The easiest way to get experiences with a microcontroller and ecosystem is by buying a development kit and trying out the examples. Fortunately, the STMicro offer a wide range of devkits, so I bought an [STM32H747I-DISCO](https://www.st.com/en/evaluation-tools/stm32h747i-disco.html) and a [NUCLEO-H743ZI](https://www.st.com/en/evaluation-tools/nucleo-h743zi.html) board and started experimenting. It was very exciting at first, but it slowly turned into a desperate struggle. 

### First impressions with the IDE
Ten years ago I worked for a security camera developer/manufacturer startup, and used Eclipse for developing embedded Linux applications; so STMicro's Eclipse-based development environment, STM32CubeIDE, was not unfamiliar for me. It wasn't so unknown that virtually nothing had changed in the recent period, which is quite shocking given other open source IDEs that have emerged in the meantime. 

The only unknown part was the STM32Cube, which is a configurator and code generator tool. There are two ways to create a new project: either you start from a sample program or you can go through the whole configuration process from the ground up. If you choose the configuration path, you can set many parameters, but not all of them: the remainder will need to be set in headers (I haven't figured out what the STMicro’s decision principle was). Furthermore, there are software components that cannot be selected because the prerequisites for adding them are not met, but the UI does not tell you what these prerequisites are; you have to figure them out on a trial and error basis. 

![lwIP: Where is the config?](/docs/assets/images/eclipse_lwip_noconfig_brd.png){: .align-center}

After a few hours of trying, I gave up and chose the other option: I started with an example application. The example obviously violated the principle of [eating your own dog food](https://en.wikipedia.org/wiki/Eating_your_own_dog_food): the examples are not created with the project configurator; ergo _you have no chance to migrate the configuration of the example to your real application_.

### STM32H747I-DISCO and the adventure with Mbed OS

My first - naive - idea was to start development on a discovery board, take advantage of the more advanced hardware peripherals, and then migrate the finished application to a custom board. But I immediately ran into the problem that although the board has an Ethernet PHY IC and connector no example application uses them, as the control signals are used for something else and you have to solder to use Ethernet. 

Of course, another devkit (NUCLEO series, with a very similar controller) is supported, but as I showed earlier, there is no easy way to migrate between examples. In practice, this means that the entire init configuration must be reviewed/rewritten manually (in addition, the software stack seems to be over-engineered). As a beginner on the platform, I tried to migrate a Nucleo example with little success 😢. But I didn’t want to give up the benefits of the richer discovery kit (SDRAM, eMMC, audio, LCD, etc.); and when evaluating possible platforms, I noticed that Mbed OS supported this board so I gave last chance. 

To my surprise, the Mbed OS' example code worked right away: it reached the network and VSCode displayed the debug logs. Unfortunately, the OS supports few peripherals and one can't debug with it, only Serial Wire Viewer is available. It isn't an option for me: I don't want to rewrite/port the drivers and need the debugger.

![STM32H747I-DISCO](/docs/assets/images/STM32H747I-DISCO.jpg){: .align-center}

After a few weeks of futile attempts, I gave up using the discovery board: I took out the Nucleo kit and started with the network example program in Eclipse. This was the initial state of my project, _I never touched the STM32Cube code generator again_. Of course, I couldn't avoid fighting with Eclipse: I had to convert the example application written in C to a C++ project. It meant a manual `.*project` files refactoring.

Did I mention I have STM32H747 discovery boards for sale?

### Some 'easy' bugs
After I refactored the project to support C++, I checked the stability and performance of the Ethernet connection. I wasn't satisfied with the result: the performance was poor, and I also had some stability issues. By that time, I was much more experienced and started searching in the STMicro's forum. I came across some interesting things: there were a lot of posts about the malfunctioning of the Ethernet driver, and a lot of developers had similar problems. After reading a few hours, I found two posts ([How to make Ethernet and lwIP working on STM32](https://community.st.com/s/question/0D50X0000BOtfhnSQB/how-to-make-ethernet-and-lwip-working-on-stm32) and [[bug fixes] STM32H7 Ethernet](https://community.st.com/s/question/0D50X0000C6eNNSSQ2/bug-fixes-stm32h7-ethernet)) that collected all of the significant bugs with links and suggested solutions. Just for fun, I picked some comments and links out:

- **Code made by ST is not thread-safe. When used with RTOS, it generally ignores lwIP requirements described in [Common pitfalls](https://www.nongnu.org/lwip/2_1_x/pitfalls.html) and [Multithreading](https://www.nongnu.org/lwip/2_1_x/multithreading.html). IP stack initialization, Ethernet link status and DHCP processing code are all broken in this regard.** (!!!)
- [Overlapping memory regions in linker script files](https://community.st.com/s/question/0D53W00000VjTwcSAF/linker-file-bugs-in-lwip-http-server-examples-for-gcc)
- [lwIP driver Rx buffers released while still in use](https://community.st.com/s/question/0D50X00009q5WkDSAU/i-may-have-found-an-error-in-the-stm32h7-ethernet-driver-when-receiving-multiple-frames)

After finding the fixes, I overwrote the files from the example and fixed some compilation errors. And voilà, everything worked as it should. 

Learning from thread security cases, I started to look into possible heap implementation issues; since I wanted to develop an essential part of the project in C++, and the heap usage in C++ is much more significant than in C. The result was no longer a surprise. Not only were there posts on the STMicro forum, but a developer (Dave Nadler) dedicated a separate [website](https://nadler.com/embedded/newlibAndFreeRTOS.html) to the topic, summarizing the issues with the official implementation. Some thoughts from the site:
- Cube-generated projects using FreeRTOS do not properly support malloc/free/etc and family, nor general newlib RTL reentrancy. 
- Your application will corrupt memory if it calls malloc/free/etc:
  - directly
  - via a newlib C-RTL function called by your application (for example sprintf %f), or
  - via STM-provided HAL-LL code (for example STMicro's USB stack)

The bug has since been fixed in [v1.7.0](https://community.st.com/s/question/0D53W00000zH4CZSA0/stm32cubeide-170-released) (in July of 2021; two years after reporting!!), but it is not fixed in the example codes (they are not generated with their toolchain). Fortunately, I found this problem sooner and could integrate Dave Nadler's solution before it caused a problem in my project.

### SDCard driver bug
Happiness and sunshine, everything was beautiful, I was doing well with my tasks. But I noticed a confusing phenomenon: files uploaded to the SD card sometimes contained a few (1-3) bytes of junk. At first, I suspected another network error, but after a day of debugging, I excluded this. The error was clearly in the FAT file system or the SD card driver. It always occurred at the boundary of the 512-byte page, the contents of the bad pages were shifted by a few bytes; but the pages before and after were flawless. This clearly indicated some sort of memory handling problem. I went to the forums again to see if I could find any useful information on this, and all pointed to some kind of DMA error.

So I reviewed the low-level interface and I found a macro `ENABLE_SCRATCH_BUFFER` in `sd_diskio.c` which was **disabled** in the FatFS example for NUCLEO-H743I board. If this macro is disabled, it will not be guaranteed that memory buffers are aligned to 4 bytes, but the DMA controller is not able to transfer misaligned data. _It meant that this example never worked correctly._
The bug fixing was not straightforward also: defining the macro caused a compilation error (!); followed by a semantic error (a wrong placed curly bracket among `ifdef` logic, what a cliché!) and hours of debugging. But after all, I could fix it! Of course, it had been already reported (`ENABLE_SCRATCH_BUFFER` was the magic keyword) but it hasn't been fixed since on the H7 platform. 

### Summary and lessons learned

The Carillon Master Clock was completed in mid-November. I spent a lot of time on this project, learned a lot from it, and am happy with the result. However I am not happy with my efficiency: I spent at least half of the time doing work that wasn’t part of my job: fighting with CubeMX and Eclipse; testing, debugging and fixing STMicro libraries; hunting for information in forums. It is a very annoying and upsetting experience. I am sure that these bugs - found and reported by the community but not corrected by STMicro - cause many thousand of wasted engineering hours worldwide. My activity can still be interpreted as a hobby, but I’m sure the strict management of a company wouldn’t have allowed the project's budget to run out so much. 

**This quality is unsatisfactory in 2021. If I had known this at the beginning of the project, I would definitely not have chosen this platform.**