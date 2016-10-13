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
* Info leak (View process memory)
* Overwrite memory with arbitrary data

#### Correct implementation of Format strings.
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

#### Incorrect implementation of Format strings.
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
#### Stack layout
<pre>
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
</pre>

#### Debugging incorrect_implementation.c in GDB
<pre>
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
</pre>
<pre>
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
</pre>
