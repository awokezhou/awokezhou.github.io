---
title: Secondary Program Loader(SPL)
date: 2022-05-23 15:01:21
tags: [嵌入式, uboot]
categories:
- CS
- uboot
---

SPL在uboot的启动过程中是一个非常重要的概念，在uboot的启动相关代码中，可以看到很多与SPL相关判断和处理，了解SPL对于理解CPU以及整个系统的启动过程是很有帮助的。本文主要对以下内容进行介绍

* 什么是SPL，为什么需要SPL？

* SPL在uboot编译层面是如何设计的？

* 启动过程中SPL主要做了什么？

# 为什么需要SPL？

SPL全程是Secondary Program Loader，叫做第二阶段引导程序，这个"Secondary"第二阶段是怎么来的呢？uboot目前将整个启动过程设计为4个阶段

* ROM Boot

* SPL

* uboot

* kernel

第一阶段ROM Boot是固化在SoC内部的一小段启动程序，一般由芯片厂商在出厂时固化好，一般不需要对这段代码关心，需要关心的是芯片的启动模式有哪些，如何设置启动模式，启动地址是多少。ROM Boot会引导SPL的启动，按照预先设置的启动模式，看是从SD卡、emmc还是Flash加载SPL到片上RAM中运行；SPL引导uboot的加载，将uboot拷贝到DDR中，重定向到uboot运行；uboot又加载kernel，重定向到kernel运行

以上这整个过程中，为什么不直接用ROM Boot引导uboot到RAM中，然后让uboot将kernel拷贝到DDR中再切换到kernel运行，而要多出来一个SPL多此一举呢？

关键点在于一方面uboot慢慢发展，代码量及功能越来越多，另一方面SoC厂商由于程本、面积等方面的考虑，一般不会将片内RAM设计的太大，一般不会超过100KB，而uboot编译下来一般都会超过200KB，因此在RAM空间上就产生了矛盾。uboot为了规避这种问题，引入了SPL的概念，即将原本的uboot功能一分为二，输出2个独立的程序，一个是SPL，另一个是真正的uboot，SPL程序只包含CPU底层相关的关键启动代码，体积较小，能够完全运行在片上RAM中，SPL负责对DDR进行初始化，并将uboot拷贝到DDR中，切换到uboot运行。下图显示了不同启动阶段及引导程序的运行介质

{% asset_img 01.png %}

CPU上电后，首先是从片内ROM加载ROM Boot程序，进行片上系统初始化，然后将启动介质中的SPL加载到片内RAM运行，SPL对DDR进行初始化，然后将uboot拷贝到DDR中，uboot在DDR中运行，拷贝kernel到DDR中

# 从编译的层面理解SPL

SPL与真正的uboot在源代码上是同一套代码，通过编译选项`CONFIG_SPL_BUILD`进行区分，在uboot代码的很多地方，通过以下形式区分了SPL处理还是uboot处理

```c
#ifdef CONFIG_SPL_BUILD
    /* spl process */
#else
    /* uboot process */
#endif
```

在uboot的Makefile设计里，将最终uboot生成的二进制输出文件分离为了u-boot-spl.bin和u-boot.bin，u-boot-spl.bin就是SPL程序的二进制文件，而u-boot.bin是真正uboot的二进制文件。从u-boot-spl.lds和u-boot.lds链接文件可以看出，SPL和真正uboot的启动都是从_start符号开始，`_start`->`lowlevel_init`->`_main`->`board_init_f`的代码调用流程，SPL和uboot都会走一遍，只是其中执行的内容不同

# SPL主要内容

这里以u-boot-2012.10版本源码为参考，主要分析ARMv7架构的SPL处理。下图是SPL整个运行过程调用时序

{% asset_img 02.png %}

## start.S

arch/arm/cpu/armv7/start.S，入口符号为`_start`

```asm6502
.globl _start
_start: b	reset
    ldr	pc, _undefined_instruction
	ldr	pc, _software_interrupt
	ldr	pc, _prefetch_abort
	ldr	pc, _data_abort
	ldr	pc, _not_used
	ldr	pc, _irq
	ldr	pc, _fiq
```

`_start`地址为0x00000000，设置后续的几个地址为SPL的异常向量，跳转到reset

```asm6502
reset:
	bl	save_boot_params
	/*
	 * set the cpu to SVC32 mode
	 */
	mrs	r0, cpsr
	bic	r0, r0, #0x1f
	orr	r0, r0, #0xd3
	msr cpsr,r0
```

对armv7的cpsr寄存器进行设置，设置CPU的模式为超级模式

```asm6502
#if !(defined(CONFIG_OMAP44XX) && defined(CONFIG_SPL_BUILD))
	/* Set V=0 in CP15 SCTRL register - for VBAR to point to vector */
	mrc	p15, 0, r0, c1, c0, 0	@ Read CP15 SCTRL Register
	bic	r0, #CR_V		@ V = 0
	mcr	p15, 0, r0, c1, c0, 0	@ Write CP15 SCTRL Register

	/* Set vector address in CP15 VBAR register */
	ldr	r0, =_start
	mcr	p15, 0, r0, c12, c0, 0	@Set VBAR
#endif
```

通过armv7的CP15协处理器将异常向量设置到VBAR，关于ARMv7协处理器CP15以及VBAR的详细内容后续会出其他相关文章专门介绍。这里进行了`CONFIG_SPL_BUILD`宏定义判断，说明设置这里的异常向量是SPL的操作，也就是说`_start`后面的几个异常处理是针对SPL程序而言的，uboot应该会设置其他的异常向量

```asm6502
#ifndef CONFIG_SKIP_LOWLEVEL_INIT
	bl	cpu_init_cp15
	bl	cpu_init_crit
#endif
```

这里是调用了`cpu_init_cp15`和`cpu_init_crit`两个函数

```asm6502
ENTRY(cpu_init_cp15)
	/*
	 * Invalidate L1 I/D
	 */
	mov	r0, #0			@ set up for MCR
	mcr	p15, 0, r0, c8, c7, 0	@ invalidate TLBs
	mcr	p15, 0, r0, c7, c5, 0	@ invalidate icache
	mcr	p15, 0, r0, c7, c5, 6	@ invalidate BP array
	mcr     p15, 0, r0, c7, c10, 4	@ DSB
	mcr     p15, 0, r0, c7, c5, 4	@ ISB

	/*
	 * disable MMU stuff and caches
	 */
	mrc	p15, 0, r0, c1, c0, 0
	bic	r0, r0, #0x00002000	@ clear bits 13 (--V-)
	bic	r0, r0, #0x00000007	@ clear bits 2:0 (-CAM)
	orr	r0, r0, #0x00000002	@ set bit 1 (--A-) Align
	orr	r0, r0, #0x00000800	@ set bit 11 (Z---) BTB
#ifdef CONFIG_SYS_ICACHE_OFF
	bic	r0, r0, #0x00001000	@ clear bit 12 (I) I-cache
#else
	orr	r0, r0, #0x00001000	@ set bit 12 (I) I-cache
#endif
	mcr	p15, 0, r0, c1, c0, 0
	mov	pc, lr			@ back to my caller
ENDPROC(cpu_init_cp15)
```

`cpu_init_cp15`的处理是通过设置CP15寄存器来禁用TLB、指令和数据cache，禁用MMU及cache

```asm6502
ENTRY(cpu_init_crit)
	/*
	 * Jump to board specific initialization...
	 * The Mask ROM will have already initialized
	 * basic memory. Go here to bump up clock rate and handle
	 * wake up conditions.
	 */
	b	lowlevel_init		@ go setup pll,mux,memory
ENDPROC(cpu_init_crit)
#endif
```

`cpu_init_crit`的处理是直接跳转到`lowlevel_init`，这定义在lowlevel_init.S中

## lowlevel_init.S

arch/arm/cpu/armv7/lowlevel_init.S中只定义了`lowlevel_init`一个函数

```asm6502
ENTRY(lowlevel_init)
	/*
	 * Setup a temporary stack
	 */
	ldr	sp, =CONFIG_SYS_INIT_SP_ADDR
	bic	sp, sp, #7 /* 8-byte alignment for ABI compliance */

	/*
	 * Save the old lr(passed in ip) and the current lr to stack
	 */
	push	{ip, lr}

	/*
	 * go setup pll, mux, memory
	 */
	bl	s_init
	pop	{ip, pc}
ENDPROC(lowlevel_init)
```

这里有一个保存栈的操作，先将ip入栈，然后调用了`s_init()`C函数，定义在`xxx/board.c`中，主要是初始化系统时钟等，然后返回到start.S中这里

```asm6502
/* Set stackpointer in internal RAM to call board_init_f */
call_board_init_f:
	ldr	sp, =(CONFIG_SYS_INIT_SP_ADDR)
	bic	sp, sp, #7 /* 8-byte alignment for ABI compliance */
	ldr	r0,=0x00000000
	bl	board_init_f
```

调用lib/board.c中的`board_init_f`函数

## lib/board.c

arch/arm/lib/board.c中定义了`board_init_f`

```c
void board_init_f(ulong bootflag)
{
	bd_t *bd;
	init_fnc_t **init_fnc_ptr;
	gd_t *id;
	ulong addr, addr_sp;
#ifdef CONFIG_PRAM
	ulong reg;
#endif

	bootstage_mark_name(BOOTSTAGE_ID_START_UBOOT_F, "board_init_f");

	/* Pointer is writable since we allocated a register for it */
	gd = (gd_t *) ((CONFIG_SYS_INIT_SP_ADDR) & ~0x07);
	/* compiler optimization barrier needed for GCC >= 3.4 */
	__asm__ __volatile__("": : :"memory");

	memset((void *)gd, 0, sizeof(gd_t));

	gd->mon_len = _bss_end_ofs;
#ifdef CONFIG_OF_EMBED
	/* Get a pointer to the FDT */
	gd->fdt_blob = _binary_dt_dtb_start;
#elif defined CONFIG_OF_SEPARATE
	/* FDT is at end of image */
	gd->fdt_blob = (void *)(_end_ofs + _TEXT_BASE);
#endif
	/* Allow the early environment to override the fdt address */
	gd->fdt_blob = (void *)getenv_ulong("fdtcontroladdr", 16,
						(uintptr_t)gd->fdt_blob);

	for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr) {
		if ((*init_fnc_ptr)() != 0) {
			hang ();
		}
	}

#ifdef CONFIG_OF_CONTROL
	/* For now, put this check after the console is ready */
	if (fdtdec_prepare_fdt()) {
		panic("** CONFIG_OF_CONTROL defined but no FDT - please see "
			"doc/README.fdt-control");
	}
#endif

	debug("monitor len: %08lX\n", gd->mon_len);
	/*
	 * Ram is setup, size stored in gd !!
	 */
	debug("ramsize: %08lX\n", gd->ram_size);
#if defined(CONFIG_SYS_MEM_TOP_HIDE)
	/*
	 * Subtract specified amount of memory to hide so that it won't
	 * get "touched" at all by U-Boot. By fixing up gd->ram_size
	 * the Linux kernel should now get passed the now "corrected"
	 * memory size and won't touch it either. This should work
	 * for arch/ppc and arch/powerpc. Only Linux board ports in
	 * arch/powerpc with bootwrapper support, that recalculate the
	 * memory size from the SDRAM controller setup will have to
	 * get fixed.
	 */
	gd->ram_size -= CONFIG_SYS_MEM_TOP_HIDE;
#endif

	addr = CONFIG_SYS_SDRAM_BASE + gd->ram_size;

#ifdef CONFIG_LOGBUFFER
#ifndef CONFIG_ALT_LB_ADDR
	/* reserve kernel log buffer */
	addr -= (LOGBUFF_RESERVE);
	debug("Reserving %dk for kernel logbuffer at %08lx\n", LOGBUFF_LEN,
		addr);
#endif
#endif

#ifdef CONFIG_PRAM
	/*
	 * reserve protected RAM
	 */
	reg = getenv_ulong("pram", 10, CONFIG_PRAM);
	addr -= (reg << 10);		/* size is in kB */
	debug("Reserving %ldk for protected RAM at %08lx\n", reg, addr);
#endif /* CONFIG_PRAM */

#if !(defined(CONFIG_SYS_ICACHE_OFF) && defined(CONFIG_SYS_DCACHE_OFF))
	/* reserve TLB table */
	addr -= (4096 * 4);

	/* round down to next 64 kB limit */
	addr &= ~(0x10000 - 1);

	gd->tlb_addr = addr;
	debug("TLB table at: %08lx\n", addr);
#endif

	/* round down to next 4 kB limit */
	addr &= ~(4096 - 1);
	debug("Top of RAM usable for U-Boot at: %08lx\n", addr);

#ifdef CONFIG_LCD
#ifdef CONFIG_FB_ADDR
	gd->fb_base = CONFIG_FB_ADDR;
#else
	/* reserve memory for LCD display (always full pages) */
	addr = lcd_setmem(addr);
	gd->fb_base = addr;
#endif /* CONFIG_FB_ADDR */
#endif /* CONFIG_LCD */

	/*
	 * reserve memory for U-Boot code, data & bss
	 * round down to next 4 kB limit
	 */
	addr -= gd->mon_len;
	addr &= ~(4096 - 1);

	debug("Reserving %ldk for U-Boot at: %08lx\n", gd->mon_len >> 10, addr);

#ifndef CONFIG_SPL_BUILD
	/*
	 * reserve memory for malloc() arena
	 */
	addr_sp = addr - TOTAL_MALLOC_LEN;
	debug("Reserving %dk for malloc() at: %08lx\n",
			TOTAL_MALLOC_LEN >> 10, addr_sp);
	/*
	 * (permanently) allocate a Board Info struct
	 * and a permanent copy of the "global" data
	 */
	addr_sp -= sizeof (bd_t);
	bd = (bd_t *) addr_sp;
	gd->bd = bd;
	debug("Reserving %zu Bytes for Board Info at: %08lx\n",
			sizeof (bd_t), addr_sp);

#ifdef CONFIG_MACH_TYPE
	gd->bd->bi_arch_number = CONFIG_MACH_TYPE; /* board id for Linux */
#endif

	addr_sp -= sizeof (gd_t);
	id = (gd_t *) addr_sp;
	debug("Reserving %zu Bytes for Global Data at: %08lx\n",
			sizeof (gd_t), addr_sp);

	/* setup stackpointer for exeptions */
	gd->irq_sp = addr_sp;
#ifdef CONFIG_USE_IRQ
	addr_sp -= (CONFIG_STACKSIZE_IRQ+CONFIG_STACKSIZE_FIQ);
	debug("Reserving %zu Bytes for IRQ stack at: %08lx\n",
		CONFIG_STACKSIZE_IRQ+CONFIG_STACKSIZE_FIQ, addr_sp);
#endif
	/* leave 3 words for abort-stack    */
	addr_sp -= 12;

	/* 8-byte alignment for ABI compliance */
	addr_sp &= ~0x07;
#else
	addr_sp += 128;	/* leave 32 words for abort-stack   */
	gd->irq_sp = addr_sp;
#endif

	debug("New Stack Pointer is: %08lx\n", addr_sp);

#ifdef CONFIG_POST
	post_bootmode_init();
	post_run(NULL, POST_ROM | post_bootmode_get(0));
#endif

	gd->bd->bi_baudrate = gd->baudrate;
	/* Ram ist board specific, so move it to board code ... */
	dram_init_banksize();
	display_dram_config();	/* and display it */

	gd->relocaddr = addr;
	gd->start_addr_sp = addr_sp;
	gd->reloc_off = addr - _TEXT_BASE;
	debug("relocation Offset is: %08lx\n", gd->reloc_off);
	memcpy(id, (void *)gd, sizeof(gd_t));

	relocate_code(addr_sp, id, addr);

	/* NOTREACHED - relocate_code() does not return */
}
```

维护了一个global data结构，计算uboot的地址及大小，初始化DDR，将uboot拷贝到DDR中，然后再调用`relocate_code`将uboot拷贝到DDR中，并跳转到uboot
