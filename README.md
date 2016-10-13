# Format-String-Vulnerabilities
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
