#Lab1  Report
#####2014011300   计42   王欣然

####练习1  理解通过make生成执行文件的过程

**1.操作系统镜像文件ucore.img是如何一步一步生成的？(需要比较详细地解释Makefile中每一条相关命令和命令参数的含义，以及说明命令导致的结果)**

执行   make  "-v"命令，可以看到执行make命令的部分过程信息如下：
```
+ cc kern/init/init.c
+ cc kern/libs/readline.c
+ cc kern/libs/stdio.c
+ cc kern/debug/kdebug.c
+ cc kern/debug/kmonitor.c
+ cc kern/debug/panic.c
+ cc kern/driver/clock.c
+ cc kern/driver/console.c
+ cc kern/driver/intr.c
+ cc kern/driver/picirq.c
+ cc kern/trap/trap.c
+ cc kern/trap/trapentry.S
+ cc kern/trap/vectors.S
+ cc kern/mm/pmm.c
+ cc libs/printfmt.c
+ cc libs/string.c
+ ld bin/kernel
+ cc boot/bootasm.S
+ cc boot/bootmain.c
+ cc tools/sign.c
+ ld bin/bootblock

```

打开Makefile文件，其中有这样一段代码
```
# create ucore.img
UCOREIMG	:= $(call totarget,ucore.img)

$(UCOREIMG): $(kernel) $(bootblock)
	$(V)dd if=/dev/zero of=$@ count=10000
	$(V)dd if=$(bootblock) of=$@ conv=notrunc
	$(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc

$(call create_target,ucore.img)

# >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
```
这是生成ucore.img的代码，再加上make "v="的输出信息，我们可得知，要生成ucore，需要先生成kernel和bootblock。

①若要生成kernel，其关键代码如下：
```
# create kernel target
kernel = $(call totarget,kernel)

$(kernel): tools/kernel.ld

$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)

$(call create_target,kernel)

# -------------------------------------------------------------------
```
生成kernel所需要的有 kernel.ld,init.o,readline.o,stdio.o,kdebug.o,kmonitor.o, panic.o,clock.o,console.o,intr.o,picirq.o,trap.o,trapentry.o vectors.o pmm.o,printfmt.o,string.o
其中kernel.ld已存在。而对于.o文件，对应的makefile代码为
```
$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
```
它们的生成过程我们可在make "v="的信息中看到详细信息。如readline.o的为
```
gcc -Ikern/libs/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Ikern/debug/ -Ikern/driver/ -Ikern/trap/ -Ikern/mm/ -c kern/libs/readline.c -o obj/kern/libs/readline.o
```

②若要生成bootblock，其关键代码如下：
```
# create bootblock
bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))

bootblock = $(call totarget,bootblock)

$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
	@$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)
	@$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
	@$(call totarget,sign) $(call outfile,bootblock) $(bootblock)

$(call create_target,bootblock)

# -------------------------------------------------------------------
```
由make信息我们可得知生成bootblock需要bootasm.o、bootmain.o、sign。
其中bootfiles = $(call listf_cc,boot)
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc))是生成bootasm.o、bootmain.o的关键代码，实际的代码是批量生成的。

生成bootasm.o需要用到bootasm.S，用到的命令为
```
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootasm.S -o obj/boot/bootasm.o
```
生成bootmain.o需要用到bootmain.c，用到的命令为
```
gcc -Iboot/ -fno-builtin -Wall -ggdb -m32 -gstabs -nostdinc  -fno-stack-protector -Ilibs/ -Os -nostdinc -c boot/bootmain.c -o obj/boot/bootmain.o
```

生成sign在makefile中用到的命令为
```
# create 'sign' tools
$(call add_files_host,tools/sign.c,sign,sign)
$(call create_target_host,sign,sign)
# -------------------------------------------------------------------
```
实际用到的命令为
```
gcc -Itools/ -g -Wall -O2 -c tools/sign.c -o obj/sign/tools/sign.o
gcc -g -Wall -O2 obj/sign/tools/sign.o -o bin/sign
```
生成bootbolck.o用到
```
ld -m    elf_i386 -nostdlib -N -e start -Ttext 0x7C00 obj/boot/bootasm.o obj/boot/bootmain.o -o obj/bootblock.o
```
**参数说明**
ggdb：生成可供gdb使用的调试信息。
m32：生成适用于32bit环境的代码。
gstabs：生成stabs格式的调试信息。
nostdinc  不使用标准库。因为OS内核要提供服务
nostdlib：不使用标准库
fno-stack-protector：不生成用于检测缓冲区溢出的代码
Os：为减小代码大小而进行优化。因为大小不能超过主引导扇区大小
I<dir>：添加搜索头文件的路径
m <emulation>：模拟为i386上的连接器
N：设置代码段和数据段均可读写
e <entry>：指定入口
Ttext：制定代码段开始位置

**2.一个被系统认为是符合规范的硬盘主引导扇区的特征是什么?**
答：在sign.c中我们可以看到，一个磁盘主引导扇区只有512字节。且
第510个（倒数第二个）字节是0x55，
第511个（倒数第一个）字节是0xAA。
```
char buf[512];
buf[510] = 0x55;
buf[511] = 0xAA;
```

####练习二 使用qemu执行并调试lab1中的软件
**1.从CPU加电后执行的第一条指令开始，单步跟踪BIOS的执行。**
打开tools/gdbinit增加一句:
```
set architecture i8086  //设置当前调试的CPU是8086
target remote :1234
```
 在 lab1目录下，执行make debug，在gdb界面中输入
```
si
```
便可单步跟踪BIOS，然后再输入
```
 x /2i $pc  
```
显示当前eip处的汇编指令，2是看两条。

**2.在初始化位置0x7c00设置实地址断点,测试断点正常。**
在gdbinit结尾加上
```
set architecture i8086  
b *0x7c00  //在0x7c00处设置断点
continue         
x /5i $pc 
```
运行make debug可得
```
	Breakpoint 2, 0x00007c00 in ?? ()
	=> 0x7c00:      cli    
	   0x7c01:      cld    
	   0x7c02:      xor    %eax,%eax
	   0x7c04:      mov    %eax,%ds
	   0x7c06:      mov    %eax,%es
```
**3.从0x7c00开始跟踪代码运行,将单步跟踪反汇编得到的代码与bootasm.S和 bootblock.asm进行比较。**
在makefile中找到Debug的执行过程，加上一句-d in_asm -D d.log，使其将运行的汇编代码输入到文件中```$(V)$(QEMU) -d in_asm -D d.log -S -s -parallel stdio -hda $< -serial null &```
打开d.log文件，可得到
```
----------------
IN: 
0xfffffff0:  ljmp   $0xf000,$0xe05b

----------------
IN: 
0x000fe05b:  cmpl   $0x0,%cs:0x6574
0x000fe062:  jne    0xfd2b6

----------------
IN: 
0x000fe066:  xor    %ax,%ax
0x000fe068:  mov    %ax,%ss

----------------
IN: 
0x000fe06a:  mov    $0x7000,%esp

----------------
IN: 
0x000fe070:  mov    $0xf3c24,%edx
0x000fe076:  jmp    0xfd124

----------------
……
```
对比可得，这与bootasm.S和bootblock.asm中的代码相同。

**4.自己找一个bootloader或内核中的代码位置，设置断点并进行测试。**
在gdbinit中对断点位置进行修改。例如在0x00007c10设置断点的信息输出如下：
```
Breakpoint 2, 0x00007c10 in ?? ()
=> 0x7c10:      mov    $0xd1,%al
   0x7c12:      out    %al,$0x64
   0x7c14:      in     $0x64,%al
   0x7c16:      test   $0x2,%al
   0x7c18:      jne    0x7c14
```

####练习3  分析bootloader进入保护模式的过程
BIOS通过读取硬盘主引导扇区到内存，并转跳到对应内存中的位置执行bootloader。打开bootasm.s，可以看到bootloader运行的具体过程。
首先，从0x7c00的地址进入，然后进行一系列初始化的操作，对各寄存器进行初始化：
```
.code16                                             # Assemble for 16-bit mode
cli                                             # Disable interrupts
cld                                             # String operations increment # Set up the important data segment registers (DS, ES, SS).
xorw %ax, %ax                                   # Segment number zero
movw %ax, %ds                                   # -> Data Segment
movw %ax, %es                                   # -> Extra Segment
movw %ax, %ss                                   # -> Stack Segment
```
接下来是启动A20，将A20线置于高电位，使32条地址线都可用。
```
seta20.1:
    inb $0x64, %al                                  # 等待8042输入buffer为空
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                                 # 0xd1 -> port 0x64
    outb %al, $0x64                                 # 发送写8042输出的指令

seta20.2:
    inb $0x64, %al                                  
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                                 # 0xdf -> port 0x60，打开A20
    outb %al, $0x60                                 # 设高电位
```
利用bootstrap GDT和分段地址转换从实模式进入保护模式。
```
lgdt gdtdesc                                              #载入GDT表
movl %cr0, %eax
orl $CR0_PE_ON, %eax
movl %eax, %cr0                             #将cr0寄存器PE位置1，开启保护模式
```
跳转，设置保护模式的段寄存器，并建立栈堆
```
 ljmp $PROT_MODE_CSEG, $protcseg
.code32                                             # Assemble for 32-bit mode
protcseg:
movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
movw %ax, %ds                                   # -> DS: Data Segment
movw %ax, %es                                   # -> ES: Extra Segment
movw %ax, %fs                                   # -> FS
movw %ax, %gs                                   # -> GS
movw %ax, %ss     
```

完成转保护模式，进入bootmain
```
call bootmain
```

####练习4 分析bootloader加载ELF格式的OS的过程
打开bootmain.c文件。bootmain函数是bootloader进入的入口。
```
void
bootmain(void) {
    // 读取ELF第一页
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // 判断ELF文件是否合法
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    //根据ELF头部的信息，将ELF文件加载到对应的内存位置
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // 根据ELF头部储存的入口信息，找到内核的入口
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```
readsect函数功能为将数据从secno扇区读取到dst
```
static void
readsect(void *dst, uint32_t secno) {
    // 等待OK
    waitdisk();

    outb(0x1F2, 1);                         // 设置扇区数量count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);     //将32位的扇区号29-31位设为1，28位表示访问的disk数，0-27位是28位的偏移量
    outb(0x1F7, 0x20);                      // cmd 0x20 - 读取扇区

    waitdisk();

    // 读取扇区
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

readseg将count bytes从kernel读到虚拟地址中
```
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;

    va -= offset % SECTSIZE;

    // 从扇区1开始
    uint32_t secno = (offset / SECTSIZE) + 1;

    // 会比要求的写更多到内存里，但因为是升序写，所以不影响
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
}
```

####练习5 实现函数调用堆栈跟踪函数
代码具体见kdebug.c，函数上方的说明基本上已经包含了要实现的步骤，将其翻译为代码就好。
ebp指向的是calling function的ebp地址，ebp+4是调用时的eip，ebp+8是参数。

运行make qemu后得到：
```
ebp:0x00007b08 eip:0x001009a6 args:0x00010094 0x00000000 0x00007b38 0x00100092 
    kern/debug/kdebug.c:305: print_stackframe+21
ebp:0x00007b18 eip:0x00100c95 args:0x00000000 0x00000000 0x00000000 0x00007b88 
    kern/debug/kmonitor.c:125: mon_backtrace+10
ebp:0x00007b38 eip:0x00100092 args:0x00000000 0x00007b60 0xffff0000 0x00007b64 
    kern/init/init.c:48: grade_backtrace2+33
ebp:0x00007b58 eip:0x001000bb args:0x00000000 0xffff0000 0x00007b84 0x00000029 
    kern/init/init.c:53: grade_backtrace1+38
ebp:0x00007b78 eip:0x001000d9 args:0x00000000 0x00100000 0xffff0000 0x0000001d 
    kern/init/init.c:58: grade_backtrace0+23
ebp:0x00007b98 eip:0x001000fe args:0x001032fc 0x001032e0 0x0000130a 0x00000000 
    kern/init/init.c:63: grade_backtrace+34
ebp:0x00007bc8 eip:0x00100055 args:0x00000000 0x00000000 0x00000000 0x00010094 
    kern/init/init.c:28: kern_init+84
ebp:0x00007bf8 eip:0x00007d68 args:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8 
    <unknow>: -- 0x00007d67 --
```
最后即使最深的那层是第一个call的函数，也就是bootmain，可知其ebp为0x00007bf8

####练习6  完善中断初始化和处理
**1.中断描述符表（也可简称为保护模式下的中断向量表）中一个表项占多少字节？其中哪几位代表中断处理代码的入口？**
答：中断向量表一个表项占8字节。0-1字节和6-7字节构成偏移值，2-3字节是段选择子，两者共同构成中断处理程序的入口地址。

**2.请编程完善kern/trap/trap.c中对中断向量表进行初始化的函数idt_init。在idt_init函数中，依次对所有中断入口进行初始化。使用mmu.h中的SETGATE宏，填充idt数组内容。每个中断的入口由tools/vectors.c生成，使用trap.c中声明的vectors数组即可。**
**3.请编程完善trap.c中的中断处理函数trap，在对时钟中断进行处理的部分填写trap函数中处理时钟中断的部分，使操作系统每遇到100次时钟中断后，调用print_ticks子程序，向屏幕上打印一行文字”100 ticks”。**
详见kern/trap/trap.c文件中的代码


**实验总结**
不像lab0是晕头转向的各种配环境出状况，这次是正儿八经地进行各种操作，感觉量其实很大，有很多地方都是自己想没想明白或者是理解有偏差，拿着答案对的时候才纠正过来。以及学习到的经验是，一定要尽早开始做实验，因为难度都不小，还有很多拓展材料需要阅读思考，并且要考虑自己的电脑会出状况的情况（……）。这次时间上都搞得有些紧张了。