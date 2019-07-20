si 执行一条指令

si N可以在同一时间跳过N？

b *0x7c00 在地址0x7C00处设置一个断点

c 执行到下一个断点处

x/30i 0x7c00 这条gdb指令是把存放在0x7c00以及之后30字节对的内存里面的指令反汇编出来了。

x/8x 0x100000 查看从内存地址0x100000处开始之后8个字的内容



- c = co = cont = continue

- s = step 

step runs one line of code at a time. When there is a
function call, it steps into the called function

- next

next does the same thing, except that it steps over
function calls.

- nexti

- si = stepi

stepi and nexti do the same thing for assembly
instructions rather than lines of code.

上述所有的指令都可以加上一个指定的数字参数，去重复同样的操作。All take a numerical argument to specify repetition.

执行完上述指令之后，按回车是执行上一条指令。Pressing the enter key repeats the previous command.

- c = continue 

continue runs code until a breakpoint is encountered or
you interrupt it with Control-C.

- finish

finish runs code until the current function returns.

- advance

advance  runs code until the instruction
pointer gets to the specified location

- b = break

break  sets a breakpoint at the specified
location.

Locations can be memory addresses (“*0x7c00”) or
names (“mon backtrace”, “monitor.c:71”).

Modify breakpoints using delete, disable, enable.

- x

x prints the raw contents of memory in whatever format
you specify (x/x for hexadecimal, x/i for assembly, etc)

- print

print evaluates a C expression and prints the result as
its proper type. It is often more useful than x.

The output from p *((struct elfhdr *) 0x10000)
is much nicer than the output from x/13x 0x10000.

- info

info registers prints the value of every register

info frame prints the current stack frame.

- list

list  prints the source code of the function
at the specified location.

- backtrace

backtrace might be useful as you work on lab 1!

- layout

layout  switches to the given layout

GDB has a text user interface that shows useful
information like code listing, disassembly, and register
contents in a curses UI.

- set

set command to change the value of a
variable during execution.

- symbol-file

You have to switch symbol files to get function and
variable names for environments other than the kernel.
For example, when debugging JOS:
symbol-file obj/user/
symbol-file obj/kern/kernel

