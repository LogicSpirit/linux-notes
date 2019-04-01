		+------------------------+ (4k align)
		+------------------------+
		| bootmem bitmap		 |
		+------------------------+ <- min_low_pfn (4k align)
		+------------------------+ <- init_pg_tables_end
		|	init_pg_tables		 |
		+------------------------+ <- pg0 (4k align)
		+------------------------+ <- __bss_stop, _end (4 align)
		+------------------------+
		|	.bss				 |
		|	.bss.page_aligned	 |
		+------------------------+ <- __init_end, __bss_start (4k align)
		+------------------------+ <- __per_cpu_end
		|	.data.percpu		 |
		+------------------------+ <- __per_cpu_start (32 align)
		+------------------------+ <- __initramfs_end
		|	.init.ramfs			 |
		+------------------------+ <- __initramfs_start (4k align)
		+------------------------+
		|	.exit.data			 |
		|	.exit.text			 |
		+------------------------+ <- __alt_instructions_end
		|  .altinstr_replacement |
		+------------------------+ <- __alt_instructions_end
		|	.altinstructions	 |
		+------------------------+ <- __alt_instructions (4 align)
		+------------------------+ <- __security_initcall_end
		|.security_initcall.init |
		+------------------------+ <- __con_initcall_end, __security_initcall_start
		|	.con_initcall.init	 |
		+------------------------+ <- __initcall_end, __con_initcall_start
		|	.initcallx.init		 |
		+------------------------+ <- __setup_end, __initcall_start
		|	.init.setup			 |
		+------------------------+ <- __setup_start (16 align)
		+------------------------+
		|	.init.data			 |
		|	.init.text			 |
		+------------------------+ <- __init_begin (4k align)
		+------------------------+
		|	.data.init_task		 |
		+------------------------+ (4k align)
		+------------------------+ <- _edata
		|.data.cacheline_aligned |
		|	.data.idt			 |
		+------------------------+ <- __nosave_end
		|	.data.nosave		 |
		+------------------------+ <- __nosave_begin (4k align)
		+------------------------+
		~						 ~
		|		.data			 |
		|		.rodata			 |
		~						 ~
		|		__ex_table		 |
		+------------------------+ <- _etext
		~						 ~
        |		.text			 |
		~						 ~
100000  +------------------------+ <- _text
		|  I/O memory hole		 |
0A0000	+------------------------+
		|  Reserved for BIOS	 |	Leave as much as possible unused
		~                        ~
		|  Command line			 |	(Can also be below the X+10000 mark)
X+10000	+------------------------+
		|  Stack/heap			 |	For use by the kernel real-mode code.
X+08000	+------------------------+	
		|  Kernel setup			 |	The kernel real-mode code.
		|  Kernel boot sector	 |	The kernel legacy boot sector.
X       +------------------------+
		|  Boot loader			 |	<- Boot sector entry point 0000:7C00
001000	+------------------------+
		|  Reserved for MBR/BIOS |
000800	+------------------------+
		|  Typically used by MBR |
000600	+------------------------+ 
		|  BIOS use only		 |
000000	+------------------------+
