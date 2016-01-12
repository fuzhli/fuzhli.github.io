---
layout: post
title = "FreeBSD syscall在amd64架构下的实现"
date = "2014-11-05T01:11:13-05:00"
categories: blog
tags: [总结,FreeBSD]
description: 分析了FreeBSD如何在amd64架构下实现syscall，并讨论了syscall在多核情况下能否并行执行。
---

### 缘起


同事问了我一个问题：在x86多核环境下，FreeBSD的syscall是只在某一个核上(如cpu0)串行执行还是在所有的核上并行执行的？一时不知从何处入手，所以就尝试分析FreeBSD的syscall在amd64架构下的实现。

### int0x80

#### int0x80之误入迷途

所有关于Unix-like的巨著上都会提到在x86下syscall是通过int 0x80来实现的。于是[查找setidt被引用的地方](http://fxr.watson.org/fxr/ident?v=FREEBSD10;i=setidt)，设置0x80 IDT的地方只有在[函数ia32_syscall_enable](http://fxr.watson.org/fxr/source/amd64/ia32/ia32_syscall.c?v=FREEBSD10#L208):

	static void	ia32_syscall_enable(void *dummy)	{		setidt(IDT_SYSCALL, &IDTVEC(int0x80_syscall), SDT_SYSIGT, SEL_UPL, 0);	}

函数int0x80_syscall是一段汇编代码，定义在[sys/amd64/ia32/ia32_exception.S文件](http://fxr.watson.org/fxr/source/amd64/ia32/ia32_exception.S?v=FREEBSD10)中。这个汇编函数在构造完一个通用的trapframe之后再调用[函数ia32_syscall](http://fxr.watson.org/fxr/source/amd64/ia32/ia32_syscall.c?v=FREEBSD10#L175),ia32_syscall又继续调用[syscallenter](http://fxr.watson.org/fxr/source/kern/subr_syscall.c?v=FREEBSD10#L56)。syscallenter的前面有如下代码：

	PCPU_INC(cnt.v_syscall);	p = td->td_proc;
	
由此可见，syscall不是被固定在某个core上来运行的。再考虑到IDT对所有的core都是一样的，由此推出在多核条件下syscall应该是在所有core上并行执行的。

#### int0x80之验证

需要验证的是两个事情：

1. 可以在任何core上执行syscall
2. 同一个syscall可以同时在多个core上运行

最初的想法是自己新添加一个系统调用，通过增加调试信息来验证这两点。后来忽然想到系统调用[nanosleep(2)](http://www.freebsd.org/cgi/man.cgi?query=nanosleep&apropos=0&sektion=0&manpath=FreeBSD+10.1-RELEASE&arch=default&format=html)其实就能直接拿来作为例子：

1. nanosleep必须要保证两个在不同core上运行的进程在调用nanosleep时都能被执行。否则就只能保证某一个core上的进程能调用nanosleep。
2. nanosleep也必须保证两个在不同core上运行的进程能同时调用nanosleep。如果只能顺序调用，那么就很难保证sleep的精度。

#### int0x80之推倒重来

分析本该到此就结束了，但是我忍不住好奇心，想看一看libc是如何实现syscall的。所以就去分析[syscall(2)](http://www.freebsd.org/cgi/man.cgi?query=syscall&apropos=0&sektion=0&manpath=FreeBSD+10.1-RELEASE&arch=default&format=html)的源代码。

在src/lib/libc目录下搜不到syscall.c文件，但是在/usr/obj/${SRC}/lib/libc这个编译目录下找到了syscall.S,内容很简单

	#include "compat.h"
	#include "SYS.h"
	RSYSCALL(syscall)
		.section .note.GNU-stack,"",%progbits
		
RSYSCALL这个宏定义在[src/lib/libc/amd64/SYS.h文件](https://github.com/freebsd/freebsd/blob/428b45aa532260e8c6ddf0217ec31db2234d29a8/lib/libc/amd64/SYS.h#L39)中

	#define RSYSCALL(x)  ENTRY(__CONCAT(__sys_,x));                      \
                        .weak CNAME(x);                                 \
                        .set CNAME(x),CNAME(__CONCAT(__sys_,x));        \
                        .weak CNAME(__CONCAT(_,x));                     \
                        .set CNAME(__CONCAT(_,x)),CNAME(__CONCAT(__sys_,x)); \
                        mov __CONCAT($SYS_,x),%eax; KERNCALL;           \
                        jb HIDENAME(cerror); ret;                       \
                        END(__CONCAT(__sys_,x))
                        
	#define KERNCALL        movq %rcx, %r10; syscall
	
这里根本就没有使用int指令啊，syscall又是什么指令呢？难道之前的分析全都是错的！？
于是我开始分析/usr/lib/libc.a文件，看syscall(2)的汇编代码是不是和上面的代码是一样的。
通过objdump -D /usr/lib/libc.a，输出中有如下片段：

	syscall.o:     file format elf64-x86-64-freebsd
	Disassembly of section .text:
	0000000000000000 <__sys_syscall>:
		0:   b8 00 00 00 00          mov    $0x0,%eax
		5:   49 89 ca                mov    %rcx,%r10
		8:   0f 05                   syscall
		a:   0f 82 00 00 00 00       jb     10 <__sys_syscall+0x10>
		10:   c3                      retq

确实是syscall.S文件对应的指令，所以对syscall的实现从新开始进行分析。

### syscall指令

syscall居然是一条x86的指令，Intel指令手册上的描述是：

	SYSCALL—Fast System Call	Description	SYSCALL saves the RIP of the instruction following SYSCALL to RCX and loads a new RIP from the IA32_LSTAR (64-bit mode). Upon return, SYSRET copies the value saved in RCX to the RIP.	SYSCALL saves RFLAGS (lower 32 bit only) in R11. It then masks RFLAGS with an OS-defined value using the IA32_FMASK (MSR C000_0084). The actual mask value used by the OS is the complement of the value written to the IA32_FMASK MSR. None of the bits in RFLAGS are automatically cleared (except for RF). SYSRET restores RFLAGS from R11 (the lower 32 bits only).
	
IA32_LSTAR在FreeBSD中定义为：

	#define	MSR_LSTAR	0xc0000082	/* long mode SYSCALL target rip */
	
引用它的地方有两处，分别是hammer_time和init_secondary,使用方法一样：

	wrmsr(MSR_LSTAR, (u_int64_t)IDTVEC(fast_syscall));
	
fast_syscall是一段汇编代码，定义在文件[sys/amd64/amd64/exception.S](http://fxr.watson.org/fxr/source/amd64/amd64/exception.S?v=FREEBSD10#L350).fast_syscall和前面的int0x80_syscall和类似，都需要先构造一个trapframe；不同的地方是，fast_syscall之后会调用amd64_syscall()，而amd64_syscall()最终也会调用到函数syscallenter().

看来syscall(2)确实如前面分析的那样，是在syscall.S文件中通过调用syscall指令来进入到kernel执行对应的syscall的。

#### syscall指令之syscall.S的生成

在FreeBSD的源代码树中找不到syscall.S文件，只有在编译目录/usr/obj/${SRC}下才能找到。那么syscall.S是怎么生成的呢？以RSYSCALL为关键字进行搜索，发现了其生成的原理；不止syscall.S是这样生成的，很多其他的系统调用都是这样生成的。其生成步骤是：

##### 1. src/lib/libc/sys/Makefile.inc文件（这是个要被其他Makefile include的文件）中首先把src/sys/sys/syscall.mk include进来。
在syscall.mk中定义了MIASM列表中包含的syscall对应的obj文件，其中就包括了syscall.o

##### 2. 在sys/Makefile.inc中对MIASM进行过滤操作：


'''make
	.for _asm in ${MIASM}
	.if (${MDASM:R:M${_asm:R}} == "")
	.if (${NOASM:R:M${_asm:R}} == "")
		ASM+=$(_asm)
	.endif
	.endif
	.endfor
'''
	
上面代码中的:R:M属于bmake中的variable modifier，在[bmake手册中](http://www.freebsd.org/cgi/man.cgi?query=make&apropos=0&sektion=0&manpath=FreeBSD+10.0-RELEASE&arch=default&format=html)有解释。简单来说:R就是把文件的后缀去掉，:M就是进行模式匹配。所以${MDASM:R:M${_asm:R}的意思就是：把MDASM中的每一项去掉后缀（假设这个新生成的列表为L），然后把_asm的后缀也去掉（假设这个新生成的元素为s），看s是否包含在L中。

所以上面的整段代码把MIASM中所有不会被MDASM或NOASM包含的元素组成一个新的列表ASM

##### 3. 把ASM列表中的后缀由o替换为S生成新的列表SASM并添加到SRCS中

	ASM=   ${ASM:S/.o/.S/}
	SRCS+=  ${SASM} ${SPSEUDO}

##### 4. 定义SASM的依赖，自动生成了对应的.S文件

	${SASM}:
		printf '#include "compat.h"\n' > ${.TARGET}
		printf '#include "SYS.h"\nRSYSCALL(${.PREFIX})\n' >> ${.TARGET}
		printf  ${NOTE_GNU_STACK} >>${.TARGET}
		
##### 5. src/lib/libc/Makefile会include src/lib/libc/sys/Makefile.inc， 继承了步骤3中的SRCS的值；最后include <bsd.lib.mk>.

bsd.lib.mk会对SRCS进行处理，最终生成了libc库。


#### syscall指令之疑惑

syscall(2)传入的前两个参数为0和syscall-id；
