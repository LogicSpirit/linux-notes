# printk
printk很早就在start_kernel中出现了，但是在用qemu调试的时候并没有观察到相应的打印输出

## printk
```c
asmlinkage int printk(const char *fmt, ...)
{
	va_list args;
	int r;

	va_start(args, fmt);
	r = vprintk(fmt, args);
	va_end(args);

	return r;
}

asmlinkage int vprintk(const char *fmt, va_list args)
{
	unsigned long flags;
	int printed_len;
	char *p;
	static char printk_buf[1024];		// printk内的buffer
	static int log_level_unknown = 1;

	if (unlikely(oops_in_progress))
		zap_locks();

	/* This stops the holder of console_sem just where we want him */
	spin_lock_irqsave(&logbuf_lock, flags);

	/* Emit the output into the temporary buffer */
	printed_len = vscnprintf(printk_buf, sizeof(printk_buf), fmt, args);

	//把printk_buf中的数据每行都添上"<n>"的loglevel，再通过emit_log_char复制到log_buf中
	for (p = printk_buf; *p; p++) {
		if (log_level_unknown) {
			if (p[0] != '<' || p[1] < '0' || p[1] > '7' || p[2] != '>') {
				emit_log_char('<');
				emit_log_char(default_message_loglevel + '0');
				emit_log_char('>');
			}
			log_level_unknown = 0;
		}
		emit_log_char(*p);
		if (*p == '\n')
			log_level_unknown = 1;
	}

	// 如果当前cpu没有在线，而且系统不是在RUNNING状态（在init中才会设置为RUNNING），那么就放锁退出。
	if (!cpu_online(smp_processor_id()) &&
	    system_state != SYSTEM_RUNNING) {
		/*
		 * Some console drivers may assume that per-cpu resources have
		 * been allocated.  So don't allow them to be called by this
		 * CPU until it is officially up.  We shouldn't be calling into
		 * random console drivers on a CPU which doesn't exist yet..
		 */
		spin_unlock_irqrestore(&logbuf_lock, flags);
		goto out;
	}
	//在start_kernel中的smp_prepare_boot_cpu就已经让BSP上线了。
	if (!down_trylock(&console_sem)) {
		console_locked = 1;
		/*
		 * We own the drivers.  We can drop the spinlock and let
		 * release_console_sem() print the text
		 */
		//获得console_sem就获得了drivers的使用权，drivers输出log_buf的内容时需要logbuf_lock，所以要释放logbuf_lock。
		spin_unlock_irqrestore(&logbuf_lock, flags);
		console_may_schedule = 0;
		release_console_sem();	//release_console_sem设计为先flush log_buf,在释放console_sem
	} else {
		//如果没能抢到console_sem，说明此刻有别人正在使用driver，我们只需要释放logbuf_lock让他能继续输出，他会把我们写到logbuf的内容输出的。
		/*
		 * Someone else owns the drivers.  We drop the spinlock, which
		 * allows the semaphore holder to proceed and to call the
		 * console drivers with the output which we just produced.
		 */
		spin_unlock_irqrestore(&logbuf_lock, flags);
	}
out:
	return printed_len;
}

static void emit_log_char(char c)
{
	LOG_BUF(log_end) = c;
	log_end++;
	if (log_end - log_start > log_buf_len)
		log_start = log_end - log_buf_len;
	if (log_end - con_start > log_buf_len)
		con_start = log_end - log_buf_len;
	if (logged_chars < log_buf_len)
		logged_chars++;
}

static spinlock_t logbuf_lock = SPIN_LOCK_UNLOCKED;

static char __log_buf[__LOG_BUF_LEN];
static char *log_buf = __log_buf;
static int log_buf_len = __LOG_BUF_LEN;

#define LOG_BUF_MASK	(log_buf_len-1)
#define LOG_BUF(idx) (log_buf[(idx) & LOG_BUF_MASK])

/*
 * The indices into log_buf are not constrained to log_buf_len - they
 * must be masked before subscripting
 */
static unsigned long log_start;	/* Index into log_buf: next char to be read by syslog() */
static unsigned long con_start;	/* Index into log_buf: next char to be sent to consoles */
static unsigned long log_end;	/* Index into log_buf: most-recently-written-char + 1 */
static unsigned long logged_chars; /* Number of chars produced since last read+clear operation */

/**
 * release_console_sem - unlock the console system
 *
 * Releases the semaphore which the caller holds on the console system
 * and the console driver list.
 *
 * While the semaphore was held, console output may have been buffered
 * by printk().  If this is the case, release_console_sem() emits
 * the output prior to releasing the semaphore.
 *
 * If there is output waiting for klogd, we wake it up.
 *
 * release_console_sem() may be called from any context.
 */
//调用release_console_sem会flush从con_start到log_end的数据，然后con_start后移，但是log_start并没有后移（应该时调用klogd输出完之后再移动）。
//call_console_drivers会遍历全局变量console_drivers，向已经注册的console输出信息（调用其write方法），如果尚未有驱动就啥也不干（在start_kernel开始处的printk(linux_banner)就是这样）。好在这时候log_start还
//记录着原来的数据起始位置，在register_console时，如果这个console有CON_PRINTBUFFER标志（意味着注册时flush），那么con_start=log_start，这样再release_console_sem就可以输出注册console之前printk打印的数据了。
void release_console_sem(void)
{
	unsigned long flags;
	unsigned long _con_start, _log_end;
	unsigned long wake_klogd = 0;

	for ( ; ; ) {
		spin_lock_irqsave(&logbuf_lock, flags);
		wake_klogd |= log_start - log_end;
		if (con_start == log_end)
			break;			/* Nothing to print */
		_con_start = con_start;
		_log_end = log_end;
		con_start = log_end;		/* Flush */
		spin_unlock_irqrestore(&logbuf_lock, flags);
		call_console_drivers(_con_start, _log_end);
	}
	console_locked = 0;
	console_may_schedule = 0;
	up(&console_sem);
	spin_unlock_irqrestore(&logbuf_lock, flags);
	if (wake_klogd && !oops_in_progress && waitqueue_active(&log_wait))
		wake_up_interruptible(&log_wait);
}

```

## setup_early_printk@@setup_arch
```c
//参数opt是由cmdline中"earlyprintk="设置的
int __init setup_early_printk(char *opt) 
{  
	char *space;
	char buf[256]; 

	if (early_console_initialized)	//一个全局标志，初始时为0，如果earlyprintk=的参数正确，初始化完成后置1.
		return -1;

	opt = strchr(opt, '=') + 1;

	strlcpy(buf,opt,sizeof(buf)); 
	space = strchr(buf, ' '); 
	if (space)
		*space = 0; 

	if (strstr(buf,"keep"))
		keep_early = 1;		//全局标志，(#L2)

	if (!strncmp(buf, "serial", 6)) { 
		early_serial_init(buf + 6);		//如果使用串口，需要使用outb配置串口的波特率校验等配置。
		early_console = &early_serial_console;
	} else if (!strncmp(buf, "ttyS", 4)) { 
		early_serial_init(buf);
		early_console = &early_serial_console;		
	} else if (!strncmp(buf, "vga", 3)) {
		early_console = &early_vga_console; 
	} else {
		early_console = NULL; 		
		return -1; 
	}
	early_console_initialized = 1;
	register_console(early_console);       //注册early_console，这样之前printk的内容就会打印出来了
	return 0;
}
```
early_serial_console和early_vga_console都很简单，都只定义了写函数。
```
static struct console early_serial_console = {
	.name =		"earlyser",
	.write =	early_serial_write,			// 通过in,out指令直接操作端口。写完一个字节到忙等待RDY才可以写下一个。
	.flags =	CON_PRINTBUFFER,
	.index =	-1,
};
static struct console early_vga_console = {
	.name =		"earlyvga",
	.write =	early_vga_write,			// 直接读写iomem映射后的虚拟地址空间。好像没有忙等待。
	.flags =	CON_PRINTBUFFER,
	.index =	-1,
};
```


##
