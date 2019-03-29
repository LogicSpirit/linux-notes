1. 查看寄存器

(gdb) i r
(gdb) i r a                     # 查看所有寄存器（包括浮点、多媒体）
(gdb) i r esp
(gdb) i r pc


2. 查看内存

(gdb) x /wx 0x80040000    # 以16进制显示指定地址处的数据
(gdb) x /8x $esp
(gdb) x /16x $esp+12
(gdb) x /16s 0x86468700   # 以字符串形式显示指定地址处的数据
(gdb) x /24i 0x8048a51      # 以指令形式显示指定地址处的数据（24条）


3. 修改寄存器的值

(gdb) set $v0 = 0x004000000
(gdb) set $epc = 0xbfc00000


4. 修改内存的值

(gdb) set {unsigned int}0x8048a51=0x0
(gdb) set *(unsigned int*)0x8048a54=0x55aa55aa


5. 内存搜索

Usage: find 



6. 断点、监测点

(gdb) b *0x80400000
(gdb) watch *(unsigned int *)0xbffff400==0x90909090

