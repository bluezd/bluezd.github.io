---
id: 118
title: ioremap_nocache 函数分析(一)
date: 2011-11-28T22:00:35+00:00
author: bluezd
layout: post
guid: http://www.bluezd.info/?p=118
permalink: /archives/118
views:
  - "8378"
dsq_thread_id:
  - "6103392872"
categories:
  - kernel
  - PCI
tags:
  - 2.6.22.1
  - PCI
---
最近一直在研究 USB 看到了 EHCI 部分，在 usb\_hcd\_pci_probe () 函数中：
  
`</p>
<pre>hcd->regs = ioremap_nocache (hcd->rsrc_start, hcd->rsrc_len);`</pre> 

ioremap_nocache() 函数我想大家都不陌生，现在我就把此函数分析一下，不当之处请大家谅解！

对于 EHCI 来说它把它本身的寄存器和内存映射到内存中区！但是站在 CPU 的角度来说，我们无法直接访问这块内存空间，需要将设备的总线地址映射成一个 cpu 可访问的线性地址！

调用 ioremap_nocache()函数之后，返回一个线性地址,此时 CPU 可以访问设备的内存(已经将其映射到了线性地址空间中了),此时CPU 可以使用访问内存的指令访问设备的内存空间(host bridge 判断访问物理内存还是设备中的内存)，此时我们就可以像访问内存一样来访问设备的内存(寄存器)！

内核版本 2.6.22.1

cat /proc/iomem
  
此时我们就以此区间(0xd8426800 – 0xd8426bff)表示 EHCI 总线地址区间(机器内存3G)

`</p>
<pre>
/**
  * ioremap_nocache     -   map bus memory into CPU space
  * @offset:    bus address of the memory
  * @size:      size of the resource to map
  *
  * ioremap_nocache performs a platform specific sequence of operations to
  * make bus memory CPU accessible via the readb/readw/readl/writeb/
  * writew/writel functions and the other mmio helpers. The returned
  * address is not guaranteed to be usable directly as a virtual
  * address.
  *
  * This version of ioremap ensures that the memory is marked uncachable
  * on the CPU as well as honouring existing caching rules from things like
  * the PCI bus. Note that there are other caches and buffers on many
  * busses. In particular driver authors should read up on PCI writes
  *
  * It's useful if some control registers are in such an area and
  * write combining or read caching is not desirable:
  *
  * Must be freed with iounmap.
  */
 void __iomem *ioremap_nocache (unsigned long phys_addr, unsigned long size)
 {
         phys_addr EHCI 总线地址

         size 区间大小(1023)

         unsigned long last_addr;

         void __iomem *p = __ioremap(phys_addr, size, _PAGE_PCD);

         if (!p)
                 return p;

         /* Guaranteed to be > phys_addr, as per __ioremap() */

         last_addr = phys_addr + size - 1;

         if (last_addr < virt_to_phys(high_memory) - 1) {

                 struct page *ppage = virt_to_page(__va(phys_addr));            

                 unsigned long npages;

                 phys_addr &#038;= PAGE_MASK;

                 /* This might overflow and become zero.. */

                 last_addr = PAGE_ALIGN(last_addr);

                 /* .. but that's ok, because modulo-2**n arithmetic will make
                 * the page-aligned "last - first" come out right.
                 */
                 npages = (last_addr - phys_addr) >> PAGE_SHIFT;

                 if (change_page_attr(ppage, npages, PAGE_KERNEL_NOCACHE) < 0) {
                         iounmap(p);
                         p = NULL;
                 }
                 global_flush_tlb();

         }

         return p;                                      
}`</pre> 

`</p>
<pre>#define _PAGE_PCD       0x010`</pre> 

非连续映射地址空间
  
非连续映射地址空间用于将连续的线性地址映射到不连续的物理地址！同时它也提供了一种访问高物理内存的方法！

内核主要在一下三种情况使用非连续映射地址空间

映射设备的I/O空间

为内核模块分配空间

为交换分区分配空间

非连续映射地址空间的起始地址在常规映射地址空间的结束地址后8MB-16MB之间，而且保证8MB对齐(地址的低24位为0)

`</p>
<pre>
/*
 * Remap an arbitrary physical address space into the kernel virtual
 * address space. Needed when the kernel wants to access high addresses
 * directly.
 *
 * NOTE! We need to allow non-page-aligned mappings too: we will obviously
 * have to convert them into an offset in a page-aligned mapping, but the
 * caller shouldn't need to know that small detail.
 */
void __iomem * __ioremap(unsigned long phys_addr, unsigned long size, unsigned long flags)
{
        void __iomem * addr;

        struct vm_struct * area;

        unsigned long offset, last_addr;

        pgprot_t prot;

        /* Don't allow wraparound or zero size */

        last_addr = phys_addr + size - 1;
        总线地址末端

        if (!size || last_addr < phys_addr)
                return NULL;

        /*
         * Don't remap the low PCI/ISA area, it's always mapped..
         */

        if (phys_addr >= ISA_START_ADDRESS && last_addr < ISA_END_ADDRESS)
                return (void __iomem *) phys_to_virt(phys_addr); 

        #define ISA_START_ADDRESS       0xa0000

        #define ISA_END_ADDRESS         0x100000

        640kb-1Mb 之间(此空洞用于连接到ISA总线上的设备)

        /*
         * Don't allow anybody to remap normal RAM that we're using..
         */

        if (phys_addr <= virt_to_phys(high_memory - 1)) {

                high_memory 为 896Mb 对应线性地址

                phys_addr 在小于896Mb的常规内存空间中

                char *t_addr, *t_end;

                struct page *page;

                t_addr = __va(phys_addr);
                转化成线性地址

                t_end = t_addr + (size - 1);
                若小于896MB 则此页框应该被设置为保留

                for(page = virt_to_page(t_addr); page <= virt_to_page(t_end); page++)

                        if(!PageReserved(page))
                                return NULL;
        }

        prot = __pgprot(_PAGE_PRESENT | _PAGE_RW | _PAGE_DIRTY
                        | _PAGE_ACCESSED | flags);

        #define __pgprot(x)     ((pgprot_t) { (x) }

        /*
         * Mappings have to be page-aligned
         */

        offset = phys_addr &#038; ~PAGE_MASK;
        取一页页框的偏移

        phys_addr &#038;= PAGE_MASK;
        总线地址按4KB 对齐

        #define PAGE_SHIFT      12

        #define PAGE_SIZE       (1UL << PAGE_SHIFT)

        #define PAGE_MASK       (~(PAGE_SIZE-1))

 
        size = PAGE_ALIGN(last_addr+1) - phys_addr;

        #define PAGE_ALIGN(addr)        (((addr)+PAGE_SIZE-1)&#038;PAGE_MASK)

        /*
         * Ok, go for it..
         */
        area = get_vm_area(size, VM_IOREMAP | (flags << 20));

        #define VM_IOREMAP      0x00000001      /* ioremap() and friends */

        申请一个非连续映射节点描述符

        if (!area)
                return NULL;

        area->phys_addr = phys_addr;
        总线地址

        addr = (void __iomem *) area->addr;
        起始线性地址

        if (ioremap_page_range((unsigned long) addr,
                        (unsigned long) addr + size, phys_addr, prot)) {

                vunmap((void __force *) addr);

                return NULL;
        }

        return (void __iomem *) (offset + (char __iomem *)addr);

        offset + addr
        offset 为 0

        addr 为线性地址，此地址被 CPU 用于读写 EHCI I/O mem 空间

        这也验证那句话:在X86平台上总线地址就是物理地址
}`</pre> 

`</p>
<pre>
/**
  *      get_vm_area  -  reserve a contingous kernel virtual area
  *      @size:          size of the area
  *      @flags:         %VM_IOREMAP for I/O mappings or VM_ALLOC
  *
  *      Search an area of @size in the kernel virtual mapping area,
  *      and reserved it for out purposes.  Returns the area descriptor
  *      on success or %NULL on failure.
  */
 struct vm_struct *get_vm_area(unsigned long size, unsigned long flags)
 {
         return __get_vm_area(size, flags, VMALLOC_START, VMALLOC_END);
 }

struct vm_struct *__get_vm_area(unsigned long size, unsigned long flags,
                                 unsigned long start, unsigned long end)
 {
         return __get_vm_area_node(size, flags, start, end, -1, GFP_KERNEL);
 }`</pre>