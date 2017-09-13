---
id: 125
title: ioremap_nocache 函数分析(二)
date: 2011-11-29T21:07:15+00:00
author: bluezd
layout: post
guid: http://www.bluezd.info/?p=125
permalink: /archives/125
views:
  - "4877"
dsq_thread_id:
  - "6108891954"
categories:
  - kernel
  - PCI
tags:
  - 2.6.22.1
  - kernel
  - PCI
---
非连续映射地址空间
  
`</p>
<pre>
static struct vm_struct *__get_vm_area_node(unsigned long size, unsigned long flags,
                                             unsigned long start, unsigned long end,
                                             int node, gfp_t gfp_mask)
 {
         struct vm_struct **p, *tmp, *area;

         unsigned long align = 1;

         unsigned long addr;

         BUG_ON(in_interrupt());

         if (flags & VM_IOREMAP) {

                 int bit = fls(size);

                 if (bit > IOREMAP_MAX_ORDER)
                         bit = IOREMAP_MAX_ORDER;
                 else if (bit < PAGE_SHIFT)
                         bit = PAGE_SHIFT;

                 align = 1ul << bit;
         }

         addr = ALIGN(start, align);

         size = PAGE_ALIGN(size);

         if (unlikely(!size))
                 return NULL;

         area = kmalloc_node(sizeof(*area), gfp_mask &#038; GFP_LEVEL_MASK, node);
         分配虚拟页面结构

         if (unlikely(!area))
                 return NULL;

         /*
          * We always allocate a guard page.
          */
         size += PAGE_SIZE;
         作为隔离带

         write_lock(&#038;vmlist_lock);
         查找之前的 struct vmlist 链表(有序链表从小到大) 查找到一个合理的虚拟地址空间

         for (p = &vmlist; (tmp = *p) != NULL ;p = &#038;tmp->next) {
                 if ((unsigned long)tmp->addr < addr) {
                         if((unsigned long)tmp->addr + tmp->size >= addr)
                                 addr = ALIGN(tmp->size +
                                              (unsigned long)tmp->addr, align);
                         continue;
                 }
                 if ((size + addr) < addr)
                         goto out;

                 回绕
                 if (size + addr <= (unsigned long)tmp->addr)
                         goto found;

                 addr = ALIGN(tmp->size + (unsigned long)tmp->addr, align);

                 if (addr > end - size)
                         goto out;

         }

 found:

         area->next = *p;
         *p 可能为 NULL

         *p = area;
         新申请的 struct vm_struct 加入到 vmlist 链表中

         area->flags = flags;

         area->addr = (void *)addr;
         VMALLOC_START (线性地址)

         area->size = size;

         area->pages = NULL;

         area->nr_pages = 0;

         area->phys_addr = 0;

         write_unlock(&vmlist_lock);

         return area;

 out:

         write_unlock(&vmlist_lock);

         kfree(area);

         if (printk_ratelimit())
                 printk(KERN_WARNING "allocation failed: out of vmalloc space - use vmalloc=<size> to increase size.\n");

         return NULL;

}`</pre> 

注意：此时我们假设没有打开PAE!

(页目录)
  
`</p>
<pre>
int ioremap_page_range(unsigned long addr,
                       unsigned long end, unsigned long phys_addr, pgprot_t prot)
{

        pgd_t *pgd;

        unsigned long start;

        unsigned long next;

        int err; 

        BUG_ON(addr >= end);

        start = addr;
        线性地址

        phys_addr -= addr;

        pgd = pgd_offset_k(addr);
        求出线性地址 addr 在页目录表中的地址
      
        #define pgd_offset_k(address)     pgd_offset(&init_mm, address)

        #define pgd_offset(mm, address)   ((mm)->pgd+pgd_index(address))

        #define pgd_index(address)    (((address) >> PGDIR_SHIFT) & (PTRS_PER_PGD-1))

        页目录表中的偏移 

        #define PGDIR_SHIFT     22

        #define PTRS_PER_PGD    1024 

        do {

                next = pgd_addr_end(addr, end);
                一般情况下返回 end(线性地址末端)

                err = ioremap_pud_range(pgd, addr, next, phys_addr+addr, prot);

                if (err)
                        break;

        } while (pgd++, addr = next, addr != end);

        flush_cache_vmap(start, end);

        return err;
        返回 0

}`</pre> 

<pre class="brush: cpp; title: ; notranslate" title="">#define pgd_addr_end(addr, end)                                         \
({                                                                      \
        unsigned long __boundary = ((addr) + PGDIR_SIZE) & PGDIR_MASK;  \
                                                                        \
        (__boundary - 1 &lt; (end) - 1)? __boundary: (end);                \
})

PGDIR_SIZE = 1&lt;&lt;22

PGDIR_MASK = 3FFFFF
</pre>

> addr 与 end 之间大小不能超过4Mb,因为一个页目录项最多表示4Mb 内存 

(页上层目录)
  
`</p>
<pre>
static inline int ioremap_pud_range(pgd_t *pgd, unsigned long addr,
                unsigned long end, unsigned long phys_addr, pgprot_t prot)

{
        参数:
        pgd 线性地址 addr 所表示的页目录地址
        addr 线性地址
        end 线性地址末端
        phys_addr =  EHCI 总线地址
        prot 标志

 

        pud_t *pud;

        unsigned long next; 

        phys_addr -= addr;

        pud = pud_alloc(&init_mm, pgd, addr);
        返回的是pgd 页目录地址 

        if (!pud)
                return -ENOMEM;

        do {
                next = pud_addr_end(addr, end);

                #define pud_addr_end(addr, end)                 (end)

                if (ioremap_pmd_range(pud, addr, next, phys_addr + addr, prot))
                        return -ENOMEM;

        } while (pud++, addr = next, addr != end);

        return 0;

}`</pre> 

<pre class="brush: cpp; title: ; notranslate" title="">static inline pud_t *pud_alloc(struct mm_struct *mm, pgd_t *pgd, unsigned long address)
{

        return (unlikely(pgd_none(*pgd)) && __pud_alloc(mm, pgd, address))?
                NULL: pud_offset(pgd, address);
}

static inline pud_t * pud_offset(pgd_t * pgd, unsigned long address)
{

        return (pud_t *)pgd;

}
</pre>

(页中间目录)
  
`</p>
<pre>
static inline int ioremap_pmd_range(pud_t *pud, unsigned long addr,
                unsigned long end, unsigned long phys_addr, pgprot_t prot)

{
        pud 为pgd

        pmd_t *pmd;

        unsigned long next; 

        phys_addr -= addr;

        pmd = pmd_alloc(&init_mm, pud, addr);
        pmd 还是为 pgd
 
        if (!pmd)
                return -ENOMEM;

        do {

                next = pmd_addr_end(addr, end);

                #define pmd_addr_end(addr, end)                 (end) 

                if (ioremap_pte_range(pmd, addr, next, phys_addr + addr, prot))
                        return -ENOMEM;

        } while (pmd++, addr = next, addr != end);

        return 0;

}

static inline pmd_t *pmd_alloc(struct mm_struct *mm, pud_t *pud, unsigned long address)
{

        return (unlikely(pud_none(*pud)) && __pmd_alloc(mm, pud, address))?
                NULL: pmd_offset(pud, address);
}

static inline pmd_t * pmd_offset(pud_t * pud, unsigned long address)
{

        return (pmd_t *)pud;

}`</pre> 

(页表) 开始干正经事了
  
`</p>
<pre>
static int ioremap_pte_range(pmd_t *pmd, unsigned long addr,
                unsigned long end, unsigned long phys_addr, pgprot_t prot)
{

        pte_t *pte;

        unsigned long pfn; 

        pfn = phys_addr >> PAGE_SHIFT;
        EHCI 控制器总线地址的页框号

        pte = pte_alloc_kernel(pmd, addr);
        创建页表(如果不存在)
        pte 为线性地址addr 在页表中的地址
 
        if (!pte)
                return -ENOMEM;

        do {

                BUG_ON(!pte_none(*pte));

                set_pte_at(&init_mm, addr, pte, pfn_pte(pfn, prot));

                #define pfn_pte(pfn, prot)      __pte(((pfn) << PAGE_SHIFT) | pgprot_val(prot)) 

                很明显最后一个参数为 EHCI 的总线地址加上标志位(设置到页表中的地址为总线地址)               

                设置页表 

                pfn++; 

        } while (pte++, addr += PAGE_SIZE, addr != end);

        此时假设 EHCI映射到内存的 I/O MEM大小为 1024Kb,此处会循环设置

        return 0;

}`</pre> 

`</p>
<pre>
#define pte_alloc_kernel(pmd, address)                  \
        ((unlikely(!pmd_present(*(pmd))) && __pte_alloc_kernel(pmd, address))? \
                NULL: pte_offset_kernel(pmd, address))`</pre> 

`</p>
<pre>
int __pte_alloc_kernel(pmd_t *pmd, unsigned long address)
{

        pte_t *new = pte_alloc_one_kernel(&init_mm, address);
        申请一页页框作为页表 

        if (!new)
                return -ENOMEM; 

        spin_lock(&init_mm.page_table_lock);

        if (pmd_present(*pmd))          /* Another has populated it */
                pte_free_kernel(new);
        else
                pmd_populate_kernel(&init_mm, pmd, new);
                设置到页目录表中去(将页表地址填入到页目录中)

        spin_unlock(&init_mm.page_table_lock);

        return 0;

}`</pre> 

`</p>
<pre>
pte_t *pte_alloc_one_kernel(struct mm_struct *mm, unsigned long address)
{

        return (pte_t *)__get_free_page(GFP_KERNEL|__GFP_REPEAT|__GFP_ZERO);

}`</pre> 

`</p>
<pre>
#define pte_offset_kernel(dir, address) \
       ((pte_t *) pmd_page_vaddr(*(dir)) +  pte_index(address))`</pre> 

> 线性地址 addr 在页表中的地址 

`</p>
<pre>
#define pmd_page_vaddr(pmd) \
                 ((unsigned long) __va(pmd_val(pmd) & PAGE_MASK))

#define pte_index(address) \
                (((address) >> PAGE_SHIFT) & (PTRS_PER_PTE - 1))`</pre> 

取线性地址的中间10位
  
`</p>
<pre>#define PTRS_PER_PTE    1024`</pre> 

调用 ioremap_nocache() 函数之后，返回一个线性地址,此时 CPU 可以访问设备的内存(已经将其映射到了线性地址空间中了),此时 CPU 可以使用访问内存的指令访问设备的内存空间(host bridge 判断访问物理内存还是设备中的内存)，此时我们就可以像访问内存一样来访问设备的内存(寄存器)！

现在我们就可以使用 readl() 或 writel() 函数读取或写入 EHCI 中 Capability Registers Operational Registers，此时我们就可以对EHCI 编程了！

注意：
       
PCI 设备的 I/O 寄存器 与 配置寄存器 的区别哦！

只提一点,后续的文章会有一部分专门介绍 PCI
       
对于任何一个PCI 设备 其内部
       
都有 I/O 寄存器 \----> 由cpu 直接访问 (writel or readl)
       
PCI 配置寄存器 \----> host bridge 直接访问 (pci\_read\_config_word)

真相在此：
  
<a href="http://www.bluezd.info/wp-content/uploads/2011/11/23010930_1304601842tiIr.jpg" class="highslide-image" onclick="return hs.expand(this);"><img src="http://www.bluezd.info/wp-content/uploads/2011/11/23010930_1304601842tiIr-1024x911.jpg" alt="" title="ioremap" width="640" height="569" class="aligncenter size-large wp-image-126" srcset="http://www.bluezd.info/wp-content/uploads/2011/11/23010930_1304601842tiIr-1024x911.jpg 1024w, http://www.bluezd.info/wp-content/uploads/2011/11/23010930_1304601842tiIr-300x266.jpg 300w, http://www.bluezd.info/wp-content/uploads/2011/11/23010930_1304601842tiIr.jpg 1260w" sizes="(max-width: 640px) 100vw, 640px" /></a>