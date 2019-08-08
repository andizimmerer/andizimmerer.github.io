---
title: "Creating portable native applications with Mono"
date: 2017-10-22
summary: "While .Net projects usually require a runtime environment, the Mono project allows us to compile a program to native code with all dependencies included. In this blog post, we will explore how to do this."
---


While C# is well supported on all machines running Windows, the [Mono][mono] project aims to bring the .NET Framework and the Common Language Runtime to other platforms and therefore enables cross-platform programs written in C#.

.NET programs are compiled to CIL ([common intermediate language][cil]) code. When executing such a program the CIL is piped through a JIT ([just-in-time][jit]) compiler which creates the binary code on the fly. This is similar to Java Bytecode and the JVM.
Still, some kind of runtime environment is required on the target machine to interpret the CIL, regardless if being implemented by Microsoft or the Mono project, to run the compiled C# code.

But Mono gives us the power to compile C# code to native binary files and to pack everything into one executable so that we do not need a runtime environment on the target machine! Therefore, our user neither needs Mono nor .NET installed!

In this tutorial, we will create a small hello world program, compile it to native code (on Linux) and pack it into one executable file that can be run on any other Linux machine without having Mono installed. This approach, of course, is not platform-independent any more. Although Mono lets you compile your code for other platforms, the final binary will only work on Linux or Windows machines, but not both. But what you gain is being independent of a third-party runtime installed on the user's computer.

I assume that Mono is already installed on your machine. If not, please follow the official installation instructions: [http://www.mono-project.com/download/](http://www.mono-project.com/download/). You should have at least version 5.0 installed.

# Creating a new program

Let's start by creating a new file `hello.cs` with a small C# program that prints out "Hello World":

```csharp
using System;
 
public class HelloWorld
{
 static public void Main()
 {
 Console.WriteLine("Hello World");
 }
}
```

Compile this program by typing in the command line:
```shell
mcs hello.cs
```
This will create a binary file `hello.exe` in the same folder. Although the newly created file has a `.exe` file extension this is just platform-independent CIL code (so no worries Linux users).

We can now execute our program within the Mono runtime by typing
```shell
mono hello.exe
```
And it should print "Hello World" to the command line. 
But this file still requires a runtime for execution. We will now get rid of it ;)


# Packing the files into one executable

Mono comes with a tool called `mkbundle` which does exactly what we want to accomplish.

```shell
mkbundle --deps hello.exe -o hello
```
The `--deps` options bundles all referenced assemblies into one self-contained image. We could also cross-compile to other platforms with the `--cross` option.

If the error occurs that some default libraries can not be found you could try including the option `--sdk /usr/`. This is needed if the default sdk location is not set (e.g. this is the case on Debian with Mono 5.0 from the official repositories)

The command produces a native binary file called `hello` with all dependencies packed into it. We can ensure this by running `ldd hello` which gives us the following output:

```shell
$ ldd hello
linux-vdso.so.1 (0x00007fff05971000)
libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f7b14bc3000)
librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f7b149bb000)
libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f7b147b7000)
libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f7b1459a000)
libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f7b14383000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f7b13fe4000)
/lib64/ld-linux-x86-64.so.2 (0x00007f7b154fa000)
```
`ldd` shows us all shared libraries for the executable file, that have to be provided by the operating system on the target machine.
As you can see no Mono dependencies need to be linked dynamically. This means there is no need to have Mono installed on the target machine.

Finally, you can distribute the resulting `hello` file by just copying it to another Linux machine. 
If you want to release your software to other operating systems you can use the cross-compile functionality of `mkbundle`.


We now learned how to bundle CIL files into one executable native binary. Although this feature is seldom used, it provides a lot of power.





[mono]: http://www.mono-project.com/
[jit]: https://en.wikipedia.org/wiki/Just-in-time_compilation
[cil]: https://en.wikipedia.org/wiki/Common_Intermediate_Language
