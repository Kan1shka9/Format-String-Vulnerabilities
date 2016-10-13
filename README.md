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
