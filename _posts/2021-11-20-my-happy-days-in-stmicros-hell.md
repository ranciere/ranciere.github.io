---
layout: post
title:  "My happy days in STMicro's hell"
---

# My sabbatical plans

This summer after fifteen years of hardworking as a developer/tech lead/manager in the embedded industry, I wanted to rest a bit. Nothing extra, just the common things: some recreational programming,  learning new things (Hi Zig!), teaching my daughter, hobby projects, AdventOfCode, cooking, etc. I discussed the details with my family, and I quit. I had big plans, a lot of free time and I was happy (I'm still) and curious. 

During the notice period, my father contacted me with a business opportunity. He has a small company that works for churches: a tower clock and carillon manufacturer/installer firm. He wanted me to develop a new state-of-the-art carillon controller. It was just a perfect situation for me: I wanted to code again, and it was a real project not just art for art's sake.

# What is a carillon?

From Wikipedia:

> A carillon is a pitched percussion instrument that is played with a keyboard and consists of at least 23 cast bronze bells in fixed suspension and tuned in chromatic order so that they can be sounded harmoniously together. Housed in bell towers, carillons are usually owned by churches, universities, or municipalities. The bells are struck with clappers connected to a keyboard [...] Often, carillons include an automatic system through which the time is announced and simple tunes are played throughout the day

So, it is a piece of musical instrument that is automatized. From the system point of view, it doesn't seem too complicated; the main components are the bells sounded by electromagnetic clappers and the Master Clock which controls the clappers and provides the User Interface. 

Traditionally these types of Master Clocks have some old-school User Interfaces from the 90s, with low-resolution monochrome character LCDs, physical buttons, etc.; and don't have remote control capabilities. As this development is for the coming years we wanted to break with this tradition, therefore, the remote capability became a key requirement. On the other hand, it is a very conservative market, the customers (mostly churches) don't want to depend on third-party service providers, and pay additional regular cloud costs; they require maximum independence and long-term solutions. 

The two requirements (remote & no-cloud) can be met by a simple technical solution: the carillon's user interface should be Web-based. It doesn't seem too like too high expectation in 2021, right? (Of course, it requires some dynamic DNS, but it is much more mature and simple; even some of the LTE AP manufacturers provide it as an additional free service.)

# System design

In addition to the two basic requirements, there are a few more: 
1. Usually, the carillons consist of 18-60 bells, depending on the physical space and money available. It would be pointless to install a fully equipped Master Clock on every site, so the power electronics part should be scalable.
2. On holidays the cantor plays on the carillon directly, e.g. a MIDI input is required.
3. Furthermore, MIDI is the standard for digital music interoperability,  the stored pieces of music should be stored in that format. The users shall upload MIDI files on the carillon and play them.

So, the HW (SBC/microcontroller) should provide Ethernet MAC; UARTs for MIDI and to control power electronics; SDIO for Micro SD Card (for storage); Real-Time Clock and GPIO; just to name the most important features. The common consumer SBCs don't fit well because of the environmental conditions (cold winters and hot summers in attic, 24/7 unsupervised operation). On other hand the overall hardware complexity is low and it can be implemented on a four-layer PCB, so we decided to use a high-end microcontroller on our designed PCB. But what microcontroller would be the right? The main three providers (STM, Microchip, NXP) offer very similar products in terms of both features and price (for this quantity). _For us, the real differencing factor is the support._