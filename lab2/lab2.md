## Lab2  Report

##### 计42  王欣然  2014011300

#### 练习0：填写已有实验
将Lab1中的代码填入Lab2文件的对应位置。

#### 练习1：实现 first-fit 连续物理内存分配算法（需要编程）
已有的代码能够基本实现内存分配和释放，但没有正确地维护free list，没有按照地址排序，不是first-fit分配。
在default_init_memmap中，要让新的内存块插入到队列末尾而非开头，因此要用list_add_before：

    static void
    default_init_memmap(struct Page *base, size_t n) {
        assert(n > 0);
        struct Page *p = base;
        for (; p != base + n; p ++) {
            assert(PageReserved(p));
            p->flags = p->property = 0;
            set_page_ref(p, 0);
        }
        base->property = n;
        SetPageProperty(base);
        nr_free += n;
        list_add_before(&free_list, &(base->page_link));//new block should be inserted at the end
    }
   
   default_alloc_pages中要先记录内存块之前所在的队列位置，然后将内存块插入回这个位置：
	
	        if (page != NULL) {
    	list_entry_t *pre_page = list_prev(&(page->page_link));//Get the pre-page
            list_del(&(page->page_link));
            if (page->property > n) {
                struct Page *p = page + n;
                p->property = page->property - n;
                list_add(pre_page, &(p->page_link));//Add this part after the pre-page
        }
            nr_free -= n;
            ClearPageProperty(page);
        }
default_free_pages中释放的空间不应插入队列开头，而是第一个地址大于base且无法合并的块之前：

      while (le != &free_list) {
            p = le2page(le, page_link);
    	if (base + base->property < p)  //The address is bigger than base and cannot be combined
                break;
            le = list_next(le);
            if (base + base->property == p) {
                base->property += p->property;
                ClearPageProperty(p);
                list_del(&(p->page_link));
            }
            else if (p + p->property == base) {
                p->property += base->property;
                ClearPageProperty(base);
                base = p;
                list_del(&(p->page_link));
            }
        }
        nr_free += n;
        list_add_before(le, &(base->page_link));//Add before the first block whose addr is bigger than base and cannot be combined


**实现与参考答案的区别**:
参考答案中，将每一页都放入了链表中，而实际上并不需要这样做，将空闲块的头页放进链表即可。



#### 练习2：实现寻找虚拟地址对应的页表项（需要编程）
按照注释进行实现。
先根据地址找到对应的目录表项，若该目录表项对应的页没有分配，则分配物理内存页。分配时将对应页的ref值置1，并将物理地址转为虚拟地址后memset。
然后将物理地址和权限位更新至页目录表中。
然后根据中间页表索的引找到对应页表项位置，返回指针。

函数中填充的代码具体如下：

        pde_t pde = pgdir[PDX(la)]; // (1) find page directory entry
        if ((pde & PTE_P) != 1) {   // (2) check if entry is not present
            if (!create)            // (3) check if creating is needed, then alloc page for page table
                return NULL;
            struct Page *page = alloc_page();
            if (page == NULL)       // alloc page failed
                return NULL;
            set_page_ref(page, 1);  // (4) set page reference
            uintptr_t pa = page2pa(page);   // (5) get linear address of page
            uintptr_t va = (uintptr_t) KADDR(pa);   // virtual address of the physical address
            memset((void *)va, 0, PGSIZE);  // (6) clear page content using memset
            pde = pa | PTE_P | PTE_W | PTE_U;   // (7) set page directory entry's permission
            pgdir[PDX(la)] = pde;       // update index
        }
        pte_t *pte_base = (pte_t *)KADDR(PDE_ADDR(pde));    //get the base
        return &pte_base[PTX(la)];  // return pte

**描述页目录项（Pag Director Entry）和页表（Page Table Entry）中每个组成部分的含义和以及对ucore而言的潜在用处**
PDE是一级页表，首位为valid位，后面的位存储PTE的基地址。
PTE是二级页表，首位为valid位，后面的位存储物理内存中相应的基地址。
通过PDE、PTE二级页表的机制可以节省页表本身占据的内存空间。

**如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？**
硬件先需要存储当前寄存器和堆栈的状态，然后将控制权交给操作系统。
如果是缺页异常则需要进行换页，然后恢复寄存器和堆栈的状态，并重新执行触发异常的语句。
如果页访问溢出异常，则需要中断应用进程。

**实现与参考答案的区别**:
与参考答案的思路基本一致


#### 练习3：释放某虚地址所在的页并取消对应二级页表项的映射（需要编程）
按照注释进行实现
先看该页表项是否present。
如果否，则不进行操作。
如果是，则要找到该页，将ref减一。若ref为0，则代表该物理页没有引用了，应被释放。
清空页表项和快表项。

        pte_t pte = *ptep;
        if (pte & PTE_P) {  //(1) check if this page table entry is present
            struct Page *page = pte2page(*ptep); //(2) find corresponding page to pte
            if (page_ref_dec(page) == 0) { //(3) decrease page reference
                free_page(page); //(4) and free this page when page reference reachs 0
            }
            *ptep = 0;  //(5) clear second page table entry
            tlb_invalidate(pgdir, la);  //(6) flush tlb
        }

**数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？**
有对应关系，每个Page都对应一个物理页，而页目录表和页表项最终指向的也是一个物理页，即对应到相应的Page结构。

**如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？**
将 ucore 的起始地址和内核虚地址设为相同的值，偏移量为设0。

**实现与参考答案的区别**:
与参考答案的思路基本一致


#### 本次实验重要知识点
1.First-Fit算法原理：算法需要维护一个查找有序空闲块（以页为最小单位的连续地址空间）的数据结构。所维护的链表实际上是一个循环的双向链表，利用一个“哨兵节点”可较方便的取得链表头尾。按地址从小到大排列刚好可以按Page结构在内存中的地址排序，对应于实际内存的地址排序。
2.页目录表、页表结构：对于一个给定的虚拟地址，高十位加上页目录表基址找到对应的页表基址。找到的页表基址加上虚拟地址的中间十位找到对应的页表项。
3.页的释放过程：利用Page的ref记录页被页表的引用记数，若ref减为0，表示该物理内存页没有被使用，则应当释放该页。

**实验未覆盖知识点**
页表的pte被释放时，二级页表的释放问题