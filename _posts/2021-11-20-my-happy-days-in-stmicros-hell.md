---
layout: post
title:  "My happy days in STMicro's hell"
image:
  path: /docs/assets/images/bell_image.jpg
  thumbnail: /docs/assets/images/bell_thumbnail.jpg
  caption: "Photos by Gefi (https://www.facebook.com/tamas.g.varga)"
---

This summer after fifteen years of hardworking as a developer/tech lead/manager in the embedded industry, I wanted to rest a bit. 

Nothing extra, just the common things: some recreational programming,  learning new things (Hi Zig!), teaching my daughter, hobby projects, AdventOfCode, cooking, etc. I discussed the details with my family, and I quit. I had big plans, a lot of free time and I was happy (I'm still) and curious. 

During the notice period, my father contacted me with a business opportunity. He has a small company that works for churches: a tower clock and carillon manufacturer/installer firm. He wanted me to develop a new state-of-the-art carillon controller. It was just a perfect situation for me: I wanted to code again, and it was a real project not just art for art's sake.

### What is a carillon?

From Wikipedia:

> A carillon is a pitched percussion instrument that is played with a keyboard and consists of at least 23 cast bronze bells in fixed suspension and tuned in chromatic order so that they can be sounded harmoniously together. Housed in bell towers, carillons are usually owned by churches, universities, or municipalities. The bells are struck with clappers connected to a keyboard [...] Often, carillons include an automatic system through which the time is announced and simple tunes are played throughout the day

So, it is a piece of musical instrument that is automatized. From the system point of view, it doesn't seem too complicated; the main components are the bells sounded by electromagnetic clappers and the Master Clock which controls the clappers and provides the User Interface. 

![System design](/docs/assets/images/carillon_system.png){: .align-left} Traditionally these types of Master Clocks have some old-school User Interfaces from the 90s, with low-resolution monochrome character LCDs, physical buttons, etc.; and don't have remote control capabilities. As this development is for the coming years we wanted to break with this tradition, therefore, the remote capability became a key requirement. On the other hand, it is a very conservative market, the customers (mostly churches) don't want to depend on third-party service providers, and pay additional regular cloud costs; they require maximum independence and long-term solutions. 

The two requirements (remote & no-cloud) can be met by a simple technical solution: _the carillon's user interface should be Web-based._ It doesn't seem too like too high expectation in 2021, right? (Of course, it requires some dynamic DNS, but it is much more mature and simple; even some of the LTE AP manufacturers provide it as an additional free service.)

### System design

In addition to the two basic requirements, there are a few more: 
1. Usually, the carillons consist of 18-60 bells, depending on the physical space and money available. It would be pointless to install a fully equipped Master Clock on every site, so the power electronics part should be scalable.
2. On holidays the cantor plays on the carillon directly, e.g. a MIDI input is required.
3. Furthermore, MIDI is the standard for digital music interoperability,  the stored pieces of music should be stored in that format. The users shall upload MIDI files on the carillon and play them.

So, the HW (SBC/microcontroller) should provide Ethernet MAC; UARTs for MIDI and to control power electronics; SDIO for Micro SD Card (for storage); Real-Time Clock and GPIO; just to name the most important features. The common consumer SBCs don't fit well because of the environmental conditions (cold winters and hot summers in attic, 24/7 unsupervised operation). On other hand the overall hardware complexity is low and it can be implemented on a four-layer PCB, so we decided to use a high-end microcontroller on our designed PCB. 

![HW architecture](/docs/assets/images/carillon_master_clock.png){: .align-center} 

In the figure, you can see the basic architecture. The `Controller` is the most complex part of the system, it provides the User Interface (in form of Single Page Application), synchronizes the RTC, plays the MIDI files at the appropriate time. The `PWR Out` peripherals are run by simple 8 bits microcontrollers. The cards control the electromagnetic hammers and are connected to the `Controller` via an RS485 bus. From the programmer's point of view, the key factor is the usefulness/applicability of the `Controller`'s MCU.

But what microcontroller would be the right? The main three providers (STM, Microchip, NXP) offer very similar products in terms of both features and price (for this quantity). _For us, the real differencing factor is the support._

### Choosing MCU ecosystem

**Processor architecture & development kits.** In recent years, the advance of ARM Cortex-M cores has been unstoppable; virtually all major manufacturers offer such microcontrollers. For maximum compatibility, I have chosen ARM. As the power consumption is not critical, but we would like to add some compute-intensive features later, the currently available best Cortex M7 cores have been selected. All of the major players offer Cortex M7 based, Ethernet-enabled MCUs (Microchip SAM V7, NXP i.MX series, STMicro STM32 F7 & H7) with a large amount of Embedded Flash. Also, they all provide dev kits, but *STMicro* is the best in terms of accessibility and selection range; not to mention the price. 
 
**Community.** At the beginning of the project, I asked some of my old friends and colleagues about the MCU selection. Without exception, they suggested the STM32 series. After that, I googled "STM32", "SAM" and "i.mx RT" on Reddit's Embedded channel and Stackoverflow and by far STM32 had the most hits. The clear winner is *STMicro*. _(Then I didn't know about STMicro's forum yet.)_

**IDE support.** I’m a Veteran Linux user, I started using Linux in high school in the ’90s. Even so, all my own computers run Linux. I want an IDE that is fully supported on Linux. Unfortunately, the Microchip (ex-Atmel) Studio is a Visual Studio-based IDE (not VSCode!), so it cannot be used. Both STMicro and NXP offer an Eclipse-based IDE with an integrated GNU C/C++ toolchain. Eclipse is absolutely old school, I'd be happier with some VSCode based solution; but in the end, it's okay. 

**All in all, the STM32 H7 series is the right choice for us.**

_In the next part, I will summarize my experiences with STMicroelectronics IDE and software libraries._
