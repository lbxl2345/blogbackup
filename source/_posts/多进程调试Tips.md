####
如果gdb没有对多进程调试的支持，gdb会在fork的时候继续调试parent进程；如果在子进程中设置了断点，会直接造成子进程终止（个人理解是无法处理SIGTRAP信号）。  
有一种绕弯子的方式能够去调试子进程：那就是在fork子进程之后，进行睡眠。在子进程睡眠的时候，就可以利用gdb进行attach，从而调试子进程。  
第二种方法是利用`set detach-on-fork off`来让父、子进程都被gdb控制。默认的情况是parent会被debug，而child会被挂起。但通过设置`set follow-fork-mode child`可以改变这种方式，此时parent会被挂起，而child会被debug。