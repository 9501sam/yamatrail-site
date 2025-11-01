+++
date = '2025-11-01T13:01:44+08:00'
draft = false
title = '[xv6 學習紀錄 11] Lab net: Network driver'
series = ["xv6 學習紀錄"]
weight = 114
+++

## Background

> You'll use a network device called the E1000 to handle network communication. To xv6 (and the driver you write), the E1000 looks like a real piece of hardware connected to a real Ethernet local area network (LAN). In fact, the E1000 your driver will talk to is an emulation provided by qemu, connected to a LAN that is also emulated by qemu. On this emulated LAN, xv6 (the "guest") has an IP address of 10.0.2.15. Qemu also arranges for the computer running qemu to appear on the LAN with IP address 10.0.2.2. When xv6 uses the E1000 to send a packet to 10.0.2.2, qemu delivers the packet to the appropriate application on the (real) computer on which you're running qemu (the "host"). 

對於 xv6 來說，這就如同是連接上真正的 Ethernet

> You will use QEMU's "user-mode network stack". QEMU's documentation has more about the user-mode stack here. We've updated the Makefile to enable QEMU's user-mode network stack and the E1000 network card. 

需要先了解一些 QEMU 的文件

> The Makefile configures QEMU to record all incoming and outgoing packets to the file packets.pcap in your lab directory. It may be helpful to review these recordings to confirm that xv6 is transmitting and receiving the packets you expect. To display the recorded packets:
> 
> ```sh
> tcpdump -XXnr packets.pcap
> ```


> We've added some files to the xv6 repository for this lab. The file `kernel/e1000.c` contains initialization code for the E1000 as well as empty functions for transmitting and receiving packets, which you'll fill in. kernel/e1000_dev.h contains definitions for registers and flag bits defined by the E1000 and described in the Intel E1000 Software Developer's Manual. kernel/net.c and kernel/net.h contain a simple network stack that implements the IP, UDP, and ARP protocols. These files also contain code for a flexible data structure to hold packets, called an mbuf. Finally, kernel/pci.c contains code that searches for an E1000 card on the PCI bus when xv6 boots. 

`kernel/e1000.c` 中有 E1000 的初始話過程


## Lab: networking (hard)
> Your job is to complete `e1000_transmit()` and `e1000_recv()`, both in `kernel/e1000.c`, so that the driver can transmit and receive packets. You are done when `make grade` says your solution passes all the tests. 

這一題所需的背景知識真的很多

> Browse the E1000 Software Developer's Manual. This manual covers several closely related Ethernet controllers. QEMU emulates the 82540EM. Skim Chapter 2 now to get a feel for the device. To write your driver, you'll need to be familiar with Chapters 3 and 14, as well as 4.1 (though not 4.1's subsections). You'll also need to use Chapter 13 as a reference. The other chapters mostly cover components of the E1000 that your driver won't have to interact with. Don't worry about the details at first; just get a feel for how the document is structured so you can find things later. The E1000 has many advanced features, most of which you can ignore. Only a small set of basic features is needed to complete this lab. 

* skim Chapter 2 可以先稍微知道這個 device 的概念
* 想要寫出 driver 需要了解 Chapter 3 and 14 and 1.1
* Chapter 13 可以作為 reference

* 剛開始的時候不需要擔心細節，只需要先了解這個 document 的架構，有需要的時候後至少知道在哪裡
* 大多數的細節都可以忽略
* 比較 advanced 的 features 也可以忽略
* 想要做出這個 lab 只需要了解基本的 features

>  The `e1000_init()` function we provide you in `e1000.c` configures the E1000 to read packets to be transmitted from RAM, and to write received packets to RAM. This technique is called DMA, for direct memory access, referring to the fact that the E1000 hardware directly writes and reads packets to/from RAM. 

* 這裡使用 DMA 的技術，意思是我們都是透過 RAM 來跟 devices 做溝通

>  Because bursts of packets might arrive faster than the driver can process them, `e1000_init()` provides the E1000 with multiple buffers into which the E1000 can write packets. The E1000 requires these buffers to be described by an array of "descriptors" in RAM; each descriptor contains an address in RAM where the E1000 can write a received packet. struct rx_desc describes the descriptor format. The array of descriptors is called the receive ring, or receive queue. It's a circular ring in the sense that when the card or driver reaches the end of the array, it wraps back to the beginning. e1000_init() allocates mbuf packet buffers for the E1000 to DMA into, using mbufalloc(). There is also a transmit ring into which the driver should place packets it wants the E1000 to send. e1000_init() configures the two rings to have size RX_RING_SIZE and TX_RING_SIZE. 

因為度太快了，所以會有多個 buffer 需要注意

>  When the network stack in `net.c` needs to send a packet, it calls `e1000_transmit()` with an mbuf that holds the packet to be sent. Your transmit code must place a pointer to the packet data in a descriptor in the TX (transmit) ring. struct tx_desc describes the descriptor format. You will need to ensure that each mbuf is eventually freed, but only after the E1000 has finished transmitting the packet (the E1000 sets the E`1000_TXD_STAT_DD` bit in the descriptor to indicate this). 

在 `net.c` 中有 network stack

> In addition to reading and writing the descriptor rings in RAM, your driver will need to interact with the E1000 through its memory-mapped control registers, to detect when received packets are available and to inform the E1000 that the driver has filled in some TX descriptors with packets to send. The global variable regs holds a pointer to the E1000's first control register; your driver can get at the other registers by indexing regs as an array. You'll need to use indices E1000_RDT and E1000_TDT in particular. 

對於 descriptor rings 好像有一些印象

> To test your driver, run `make server` in one window, and in another window run `make qemu` and then run `nettests` in xv6. The first test in nettests tries to send a UDP packet to the host operating system, addressed to the program that `make server` runs. If you haven't completed the lab, the E1000 driver won't actually send the packet, and nothing much will happen. 

還蠻好奇 `make server` 在做什麼

> After you've completed the lab, the E1000 driver will send the packet, qemu will deliver it to your host computer, make server will see it, it will send a response packet, and the E1000 driver and then nettests will see the response packet. Before the host sends the reply, however, it sends an "ARP" request packet to xv6 to find out its 48-bit Ethernet address, and expects xv6 to respond with an ARP reply. kernel/net.c will take care of this once you have finished your work on the E1000 driver. If all goes well, nettests will print testing ping: OK, and make server will print a message from xv6!. 

在完成這個 lab 之後會 print `a message from xv6!`
