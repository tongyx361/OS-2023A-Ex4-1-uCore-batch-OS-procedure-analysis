# 分析 uCore 批处理操作系统中的执行过程

> [课后练习 4.1](https://www.yuque.com/xyong-9fuoz/qczol5/uzf18vbnscar3hzi#k9k5d)
> 依据自己的实验选择，分析 uCore 或 rCore 中下面执行过程，并形成文档，分享到本页面的交流回复区。
>
> 1. **批处理操作系统**中**应用程序管理数据结构**的**组成**；
> 2. 应用程序管理数据结构的**初始化**过程；
> 3. **`trapframe` 数据结构**的**组成**；
> 4. 在系统调用过程中的 `trapframe` 数据结构的**保存**过程；
> 5. 在系统调用返回过程中的**从 ` trapframe` 数据结构恢复应用程序执行上下文的过程**；
> 6. 系统调用执行过程中的**参数和返回值传递过程**；
>

## 1. 批处理操作系统中应用程序管理数据结构的组成

批处理操作系统中应用程序管理数据结构由以下数据组成：

```C
# os/loader.c
static int app_cur, app_num;
static uint64 *app_info_ptr;
```

其中

- `app_cur` 表示当前运行的应用的编号
- `app_num` 表示要运行的应用的总数
- `app_info_ptr` 表示储存有应用分界地址信息的内存地址

## 2. 应用程序管理数据结构的初始化过程

上述应用程序管理数据结构由汇编代码 `os/link_app.S` 与函数 `loader_init()` 初始化：

```RISC-V
# os/link_app.s
    .align 4
    .section .data
    .global _app_num
_app_num:
    .quad 3
    .quad app_0_start
    .quad app_1_start
    .quad app_2_start
    .quad app_2_end

    .global _app_names
_app_names:
   .string "ch2b_exit"
   .string "ch2b_hello_world"
   .string "ch2b_power"

    .section .data.app0
    .global app_0_start
app_0_start:
    .incbin "./user/target/bin/ch2b_exit"

    .section .data.app1
    .global app_1_start
app_1_start:
    .incbin "./user/target/bin/ch2b_hello_world"

    .section .data.app2
    .global app_2_start
app_2_start:
    .incbin "./user/target/bin/ch2b_power"
app_2_end:
```

从 `os/link_app.S` 可见，从全局符号 `_app_num` 对应的地址开始的内存空间中，储存了 5 个 4 字 = 8 字节的数据，依次为 `3` （`app_num`） 以及几个应用的分界地址。

```C
# os/loader.c
void loader_init()
{
	if ((uint64)ekernel >= BASE_ADDRESS) {
		panic("kernel too large...\n");
	}
	app_info_ptr = (uint64 *)_app_num;
	app_cur = -1;
	app_num = *app_info_ptr;
}
```

从 `loader_init()` 可见，三个数据结构依次为初始化为：

- `app_info_ptr`：全局符号 `_app_num` 对应的地址
- `app_cur`: `-1`，表示还没有正式开始运行应用
- `app_num`: 全局符号 `_app_num` 对应的地址后第一个 8 字节数据块内的数据，即 `3`，也即此处的应用总数

## 3. trapframe 数据结构的组成

### 3.1. 声明

`trapframe` 数据结构在 `trap.h` 中被声明，各成员含义可见注释或变量名

```C
# os/trap.h
struct trapframe {
	/*   0 */ uint64 kernel_satp; // kernel page table
	/*   8 */ uint64 kernel_sp; // top of process's kernel stack
	/*  16 */ uint64 kernel_trap; // usertrap()
	/*  24 */ uint64 epc; // saved user program counter
	/*  32 */ uint64 kernel_hartid; // saved kernel tp
	/*  40 */ uint64 ra;
	/*  48 */ uint64 sp;
	/*  56 */ uint64 gp;
	/*  64 */ uint64 tp;
	/*  72 */ uint64 t0;
	/*  80 */ uint64 t1;
	/*  88 */ uint64 t2;
	/*  96 */ uint64 s0;
	/* 104 */ uint64 s1;
	/* 112 */ uint64 a0;
	/* 120 */ uint64 a1;
	/* 128 */ uint64 a2;
	/* 136 */ uint64 a3;
	/* 144 */ uint64 a4;
	/* 152 */ uint64 a5;
	/* 160 */ uint64 a6;
	/* 168 */ uint64 a7;
	/* 176 */ uint64 s2;
	/* 184 */ uint64 s3;
	/* 192 */ uint64 s4;
	/* 200 */ uint64 s5;
	/* 208 */ uint64 s6;
	/* 216 */ uint64 s7;
	/* 224 */ uint64 s8;
	/* 232 */ uint64 s9;
	/* 240 */ uint64 s10;
	/* 248 */ uint64 s11;
	/* 256 */ uint64 t3;
	/* 264 */ uint64 t4;
	/* 272 */ uint64 t5;
	/* 280 */ uint64 t6;
};
```

### 3.2. 定义/初始化

从 `os/main.c` 中的 `main()` 出发，首次定义一个 `trapframe` 变量是在 `os/loader.c` 的函数 `run_next_app()` 中：

```C
# os/loader.c
int run_next_app()
{
	struct trapframe *trapframe = (struct trapframe *)trap_page;
	app_cur++;
	app_info_ptr++;
	if (app_cur >= app_num) {
		return -1;
	}
	infof("load and run app %d", app_cur);
	uint64 length = load_app(app_info_ptr);
	debugf("bin range = [%p, %p)", *app_info_ptr, *app_info_ptr + length);
	memset(trapframe, 0, 4096);
	trapframe->epc = BASE_ADDRESS;
	trapframe->sp = (uint64)user_stack + USER_STACK_SIZE;
	usertrapret(trapframe, (uint64)boot_stack_top);
	return 0;
}
```

可以看到，

1. `trapframe` 指针被赋值为 `trap_page`，而后者由同文件上方的 `__attribute__((aligned(4096))) char trap_page[TRAP_PAGE_SIZE];` 声明，是一块大小为 `TRAP_PAGE_SIZE` 字节的内存空间的起始地址
2. `trapframe` 指向的结构体的内容被赋值
   1. `memset(trapframe, 0, 4096);` 清空了 `trapframe` 结构体的内容，
      1. 此处 `4096` 即为 `TRAP_PAGE_SIZE`
   2. `trapframe->epc = BASE_ADDRESS;` 将成员 `epc` （即应用程序的 `pc` 应储存的地址）设置为 `BASE_ADDRESS`，
      1. 此处 `BASE_ADDRESS` 在 `loader.h` 中被定义为 `#define BASE_ADDRESS (0x80400000)`
      2. 注意与 `os/kernel[_app].ld` 中的变量 `BASE_ADDRESS = 0x80200000;` 区分，后者是链接器脚本中的变量，仅用于帮助阅读
   3. `trapframe->sp = (uint64)user_stack + USER_STACK_SIZE;` 将成员 `sp` （即应用程序的 `sp` 应储存的地址，也即用户栈的初始栈顶）设置为 `(uint64)user_stack + USER_STACK_SIZE`，即在用户栈基址 `user_stack` 的基础上加上用户栈的大小 `USER_STACK_SIZE`
      1. `user_stack` 由同文件上方的 `__attribute__((aligned(4096))) char user_stack[USER_STACK_SIZE];` 声明，是一块大小为 `USER_STACK_SIZE` 字节的内存空间的起始地址
      2. 注意栈的地址是从高到低的，故此处 `sp` 为用户栈的栈底

随后，在调用 `usertrapret(trapframe, (uint64)boot_stack_top)` 中：

```C
//
// return to user space
//
void usertrapret(struct trapframe *trapframe, uint64 kstack)
{
	trapframe->kernel_satp = r_satp(); // kernel page table
	trapframe->kernel_sp = kstack + PGSIZE; // process's kernel stack
	trapframe->kernel_trap = (uint64)usertrap;
	trapframe->kernel_hartid = r_tp(); // hartid for cpuid()

	w_sepc(trapframe->epc);
	// set up the registers that trampoline.S's sret will use
	// to get to user space.

	// set S Previous Privilege mode to User.
	uint64 x = r_sstatus();
	x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
	x |= SSTATUS_SPIE; // enable interrupts in user mode
	w_sstatus(x);

	// tell trampoline.S the user page table to switch to.
	// uint64 satp = MAKE_SATP(p->pagetable);
	userret((uint64)trapframe);
}
```

其中

```C
	trapframe->kernel_satp = r_satp(); // kernel page table
	trapframe->kernel_sp = kstack + PGSIZE; // process's kernel stack
	trapframe->kernel_trap = (uint64)usertrap;
	trapframe->kernel_hartid = r_tp(); // hartid for cpuid()
```

定义了 trapframe 的前四个成员，具体含义参见成员名或注释即可。

**注意**：每次运行新的应用时，都是通过 `run_next_app()` 运行的，`trapframe` 指向的结构体都会被重新初始化一次。

## 4. 在系统调用过程中的 trapframe 数据结构的保存过程

用户程序中的系统调用都会进入 `os/syscall.c` 定义的 `syscall()` 函数
如果是合法的系统调用，则会触发 `trap_init()` 指定的 `stvec`，即异常处理程序的地址 `(uint64)uservec & ~0x3`
而 `uservec` 在 `os/trampoline.S` 中被定义：

```RISC-V
.align 4
.globl uservec
uservec:
	#
        # trap.c sets stvec to point here, so
        # traps from user space start here,
        # in supervisor mode, but with a
        # user page table.
        #
        # sscratch points to where the process's p->trapframe is
        # mapped into user space, at TRAPFRAME.
        #

	# swap a0 and sscratch
        # so that a0 is TRAPFRAME
        csrrw a0, sscratch, a0

        # save the user registers in TRAPFRAME
        sd ra, 40(a0)
        sd sp, 48(a0)
        sd gp, 56(a0)
        sd tp, 64(a0)
        sd t0, 72(a0)
        sd t1, 80(a0)
        sd t2, 88(a0)
        sd s0, 96(a0)
        sd s1, 104(a0)
        sd a1, 120(a0)
        sd a2, 128(a0)
        sd a3, 136(a0)
        sd a4, 144(a0)
        sd a5, 152(a0)
        sd a6, 160(a0)
        sd a7, 168(a0)
        sd s2, 176(a0)
        sd s3, 184(a0)
        sd s4, 192(a0)
        sd s5, 200(a0)
        sd s6, 208(a0)
        sd s7, 216(a0)
        sd s8, 224(a0)
        sd s9, 232(a0)
        sd s10, 240(a0)
        sd s11, 248(a0)
        sd t3, 256(a0)
        sd t4, 264(a0)
        sd t5, 272(a0)
        sd t6, 280(a0)

	# save the user a0 in p->trapframe->a0
        csrr t0, sscratch
        sd t0, 112(a0)

        csrr t1, sepc
        sd t1, 24(a0)

        ld sp, 8(a0)
        ld tp, 32(a0)
        ld t1, 0(a0)
        # csrw satp, t1
        # sfence.vma zero, zero
        ld t0, 16(a0)
        jr t0
```

其中通过 `sscratch` 获取了 `trapframe` 指针，将各种寄存器中的数据储存到了对应成员中。

而之所以 `sscratch` 储存了 `trapframe` 指针，是因为每次运行应用程序时，`run_next_app()` 都会调用 `usertrapret(trapframe, (uint64)boot_stack_top)`，后者又调用了 `userret((uint64)trapframe)`，
`userret` 则在 `os/trampoline.S` 中被定义：

```RISC-V
.globl userret
userret:
        # userret(TRAPFRAME, pagetable)
        # switch from kernel to user.
        # usertrapret() calls here.
        # a0: TRAPFRAME, in user page table.
        # a1: user page table, for satp.

        # switch to the user page table.
        # csrw satp, a1
        # sfence.vma zero, zero

        # put the saved user a0 in sscratch, so we
        # can swap it with our a0 (TRAPFRAME) in the last step.
        ld t0, 112(a0)
        csrw sscratch, t0

        # restore all but a0 from TRAPFRAME
        ld ra, 40(a0)
        ld sp, 48(a0)
        ld gp, 56(a0)
        ld tp, 64(a0)
        ld t0, 72(a0)
        ld t1, 80(a0)
        ld t2, 88(a0)
        ld s0, 96(a0)
        ld s1, 104(a0)
        ld a1, 120(a0)
        ld a2, 128(a0)
        ld a3, 136(a0)
        ld a4, 144(a0)
        ld a5, 152(a0)
        ld a6, 160(a0)
        ld a7, 168(a0)
        ld s2, 176(a0)
        ld s3, 184(a0)
        ld s4, 192(a0)
        ld s5, 200(a0)
        ld s6, 208(a0)
        ld s7, 216(a0)
        ld s8, 224(a0)
        ld s9, 232(a0)
        ld s10, 240(a0)
        ld s11, 248(a0)
        ld t3, 256(a0)
        ld t4, 264(a0)
        ld t5, 272(a0)
        ld t6, 280(a0)

	# restore user a0, and save TRAPFRAME in sscratch
        csrrw a0, sscratch, a0

        # return to user mode and user pc.
        # usertrapret() set up sstatus and sepc.
        sret
```

这里 `csrrw a0, sscratch, a0` 将 `a0` （即储存参数 `(uint64)trapframe` 的寄存器）中的 `trapframe` 地址写入了 `sscratch`。

**注意**：由于我们的 os 一开始就运行在内核态中，是通过依此调用 `run_next_app()` -> `usertrapret(trapframe, (uint64)boot_stack_top)` -> `userret((uint64)trapframe)` 进入的用户态，
故是先初始化了 `trapframe` ，再将其写入了 `sscratch`，随后才在系统调用时借助 `sscratch` 重新获取 `trapframe`。

## 5. 在系统调用返回过程中的从 trapframe 数据结构恢复应用程序执行上下文的过程

系统调用的过程为

1. `uservec()`: 末尾处 `ld t0, 16(a0)` 将 `16(a0)` 即 `trapframe->kernel_trap` 写入 `t0`，此处 `kernel_trap` 先前被初始化为 `user_trap`，故 `jr t0` 会跳转到 `user_trap` 处
2. `usertrap()`: 若 `scause` 显示异常的起因为用户态的系统调用，就会调用 `syscall()`

在系统调用返回过程中的从 trapframe 数据结构恢复应用程序执行上下文的过程由调用 `usertrapret(trapframe, (uint64)boot_stack_top)` 实现，其中

```C
	w_sepc(trapframe->epc);
	// set up the registers that trampoline.S's sret will use
	// to get to user space.

	// set S Previous Privilege mode to User.
	uint64 x = r_sstatus();
	x &= ~SSTATUS_SPP; // clear SPP to 0 for user mode
	x |= SSTATUS_SPIE; // enable interrupts in user mode
	w_sstatus(x);

	// tell trampoline.S the user page table to switch to.
	// uint64 satp = MAKE_SATP(p->pagetable);
	userret((uint64)trapframe);
```

设置了 `sepc` 与 `sstatus`，并调用 `userret((uint64)trapframe)`

最后在 `userret()` 中，恢复了 `trapframe` 指向的结构体，与保存过程基本为镜像关系

## 6. 系统调用执行过程中的参数和返回值传递过程

系统调用执行过程为 `ecall` -> `uservec()` -> `syscall()`，

### 6.1. 参数传递过程

1. 在 `uservec()` 中从寄存器传递到了 `trapframe` 的对应成员中，详见其定义中有关参数寄存器的若干 `sd` 指令；
2. 在 `syscall()` 中从结构体成员再次以函数参数的形式传递到了对应的系统调用函数中

```C
	struct trapframe *trapframe = (struct trapframe *)trap_page;
	int id = trapframe->a7, ret;
	uint64 args[6] = { trapframe->a0, trapframe->a1, trapframe->a2,
			   trapframe->a3, trapframe->a4, trapframe->a5 };
	tracef("syscall %d args = [%x, %x, %x, %x, %x, %x]", id, args[0],
	       args[1], args[2], args[3], args[4], args[5]);
	switch (id) {
	case SYS_write:
		ret = sys_write(args[0], (char *)args[1], args[2]);
		break;
	case SYS_exit:
		sys_exit(args[0]);
		// __builtin_unreachable();
	default:
		ret = -1;
		errorf("unknown syscall %d", id);
	}
```

### 6.2. 返回值传递过程

1. `syscall()` 中，系统调用函数（`sys_exit()` 除外，其不返回）的返回值被保存在 `ret` 中，进而被 `trapframe->a0 = ret` 保存在 `trapframe->a0` 中
2. 随后 `syscall()` 返回到 `usertrap()`，再执行 `return usertrapret(trapframe, (uint64)boot_stack_top);`
3. 在 `usertrapret(trapframe, (uint64)boot_stack_top)` 中，最后又执行了 `userret((uint64)trapframe)`
4. 在 `userret((uint64)trapframe)` 中，
   1. `ld t0, 112(a0)` 将返回值写入了寄存器 `t0`
   2. `csrw sscratch, t0` 又将返回值写入了 CSR `sscratch`
   3. `csrrw a0, sscratch, a0` 最后将返回值写入了 `a0`
5. 最终，执行 `sret` 返回到用户态，此时寄存器 `a0` 中储存了系统调用的返回值
