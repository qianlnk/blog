# core文件分析

1.  修改属性使系统允许产生完整的core文件

    ```
    Lsattr –El sys0
    
    Chdev –l sys0 –a fullcore=true
    ```
 

2.  生成一个core文件

    假定杀死一个远程telnet到本地的pts终端

    ```
    F50:[/]#ps -ef  | grep pts
    
       root 14438 19000   1 10:14:43  pts/1  0:00 -ksh
    
       root 14676 14438   1 10:36:43  pts/1  0:00 grep pts
    
       root 19776 14438   5 10:36:43  pts/1  0:00 ps -ef
    
       root 20386 20846   0 10:36:00  pts/2  0:00 -ksh
    
    F50:[/]#kill -11 20386
    
    F50:[/]#find / -name core -print
    
    /core
    ```
 

3.  在根目录下生成一个core文件

 
4. 使用简单的命令分析该core文件是由什么应用产生的

    ```
       F50:[/]#lquerypv -h core 6b0 64
    
    000006B0   7FFFFFFF FFFFFFFF 7FFFFFFF FFFFFFFF  |................|
    
    000006C0   00000000 000007D0 7FFFFFFF FFFFFFFF  |................|
    
    000006D0   00120000 14C5C0A0 00000000 00000040  |...............@|
    
    000006E0   6B736800 00000000 00000000 00000000  |ksh.............|
    
    000006F0   00000000 00000000 00000000 00000000  |................|
    
    00000700   00000000 00000000 00000000 0000007D  |...............}|
    
    00000710   00000000 00000038 00000000 0000007D  |.......8.......}|
    ```

    可知是由ksh产生的

 

5. 使用gdb分析，之前需先安装gdb软件

    ```
    F50:[/solaris/usr/local/bin]#./gdb -core=/core
    
    GNU gdb 6.4
    
    Copyright 2005 Free Software Foundation, Inc.
    
    GDB is free software, covered by the GNU General Public License, and you are
    
    welcome to change it and/or distribute copies of it under certain conditions.
    
    Type "show copying" to see the conditions.
    
    There is absolutely no warranty for GDB.  Type "show warranty" for details.
    
    This GDB was configured as "powerpc-ibm-aix5.3.0.0".
    
    Core was generated by `ksh'.
    
    Program terminated with signal 11, Segmentation fault.
    
     
    
    warning: Could not open `ksh' as an executable file: A file or directory in the
    
    path name does not exist.
    
    #0  0xd026e91c in read () from /usr/lib/libc.a(shr.o)
    ```
 

6、使用dbx分析