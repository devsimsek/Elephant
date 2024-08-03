# Elephant, The PHP Compiler

[![CircleCI](https://circleci.com/gh/devsimsek/Elephant.svg?style=svg)](https://circleci.com/gh/devsimsek/Elephant)

Main purpose of this fork is to be able to add the php 8 support as well as try to create another version of php-compiler.

Special thanks to @ircmaxell of his beautiful creation.

So here we go :)

!!!Note!!!
I haven't started this project yet. My main purpose to make this compiler work with php 8.

# Installation

Install PHP 7.4, being sure to enable the FFI extension, OpenSSL extension, mbstring extension, and zlib extension (`--with-ffi --with-openssl --enable-mbstring --with-zlib`).

Also, you need to install the system dependency `llvm-4.0`. On Ubuntu:

```console
me@local:~$ sudo apt-get install llvm-4.0-dev clang-4.0
```

Then run `composer install`.

## Using docker

This project comes with one working and one non-working (yet) Dockerfile. The makefile uses a reasonably old version of Ubuntu (16.04), and once FFIMe is fixed for newer versions of glibc, it will switch to use 18.04 (or newer).

To build, use make:

```console
me@local:~$ make build
```

This will take a while (upwards of 10 minutes likely). It will install an Ubuntu container with a custom compile of PHP-7.4 and everything you need to get up and running. It will also composer install all dependencies as well as run the pre-processor. Once it's done, you can run tests:

```console
me@local:~$ make test
```

This will execute all unit tests inside the container.

To run your own code or play with the compiler, you can open a shell using `make shell`:

```console
me@local:~$ make shell
root@662c59ae4527:/compiler# php bin/jit.php -r 'echo "Hello World\n";'
Hello World
```

# Running Code

There are three main ways of using this compiler:

## VM - Virtual Machine

This compiler mode implements its own PHP Virtual Machine, just like PHP does. This is effectively a giant switch statement in a loop.

No, seriously. It's literally [a giant switch statement](lib/VM.php)...

Practically, it's a REALLY slow way to run your PHP code. Well, it's slow because it's in PHP, and PHP is already running on top of a VM written in C. 

But what if we could change that...

## JIT - Just In Time

This compiler mode takes PHP code and generates machine code out of it. Then, instead of letting the code run in the VM above, it just calls to the machine code.

It's WAY faster to run (faster than PHP 7.4, when you don't account for compile time).

But it also takes a long time to compile (compiling is SLOW, because it's being compiled from PHP).

Every time you run it, it compiles again. 

That brings us to our final mode:

## Compile - Ahead Of Time Compilation

This compiler mode actually generates native machine code, and outputs it to an executable.

This means, that you can take PHP code, and generate a standalone binary. One that's implemented **without a VM**. That means it's (in theory at least) as fast as native C.

Well, that's not true. But it's pretty dang fast.

# Okay, Enough, How can I try?

There are four CLI entrypoints, and all 4 behave (somewhat) like the PHP cli:

 * `php bin/vm.php` - Run code in a VM
 * `php bin/jit.php` - Compile all code, and then run it
 * `php bin/compile.php` - Compile all code, and output a `.o` file.
 * `php bin/print.php` - Compile and output CFG and the generated OpCodes (useful for debugging)

## Executing Code

Specifying code from `STDIN` (this works for all 4 entrypoints):

```console
me@local:~$ echo '<?php echo "Hello World\n";' | php bin/vm.php
Hello World
```

You can also specify on the CLI via `-r` argument:

```console
me@local:~$ php bin/jit.php -r 'echo "Hello World\n";'
Hello World
```

And you can specify a file:

```console
me@local:~$ echo '<?php echo "Hello World\n";' > test.php
me@local:~$ php bin/vm.php test.php
```

When compiling using `bin/compile.php`, you can also specify an "output file" with `-o` (this defaults to the input file, with `.php` removed). This will generate an executable binary on your system, ready to execute

```console
me@local:~$ echo '<?php echo "Hello World\n";' > test.php
me@local:~$ php bin/compile.php -o other test.php
me@local:~$ ./other
Hello World
```

Or, using the default:

```console
me@local:~$ echo '<?php echo "Hello World\n";' > test.php
me@local:~$ php bin/compile.php test.php
me@local:~$ ./test
Hello World
```

## Linting Code

If you pass the `-l` parameter, it will not execute the code, but instead just perform the compilation. This will allow you to test to see if the code even will compile (hint: most currently will not).

## Debugging

Sometimes, you want to see what's going on. If you do, try the `bin/print.php` entrypoint. It will output two types of information. The first is the Control Flow Graph, and the second is the compiled opcodes.

```console
me@local:~$ php bin/print.php -r 'echo "Hello World\n";'

Control Flow Graph:

Block#1
    Terminal_Echo
        expr: LITERAL<inferred:string>('Hello World
        ')
    Terminal_Return


OpCodes:

block_0:
  TYPE_ECHO(0, null, null)
  TYPE_RETURN_VOID(null, null, null)
```

# Future Work

Right now, this only supports an EXTREMELY limited subset of PHP. There is no support for dynamic anything. Arrays aren't supported. Neither Object properties nor methods are supported. And the only builtin functions that are supported are `var_dump` and `strlen`.

But it's a start...

# Debugging

Since this is bleeding edge, debuggability is key. To that vein, both `bin/jit.php` and `bin/compile.php` accept a `-y` flag which will output a pair of debugging files (they default to the prefix of the name of the script, but you can specify another prefix following the flag).

```console
me@local:~$ echo '<?php echo "Hello World\n";' > demo.php
me@local:~$ php bin/compile.php -y demo.php
# Produces: 
#   demo - executable of the code
#   demo.bc - LLVM intermediary bytecode associated with the compiled code
#   demo.s - assembly generated by the compiled code

```

Checkout the [examples](examples/) folder.

# Performance

So, is this thing any fast? Well, let's look at the internal benchmarks. You can run them yourself with `make bench`, and it'll give you the following output (running 5 iterations of each test, and averaging the time).

Check out the results in the [Benchmarks](benchmarks/) folder.

This is after the port to using LLVM under the hood. So the port to LLVM appears to have been well worth it, even just from a performance standpoint.

To run the benchmarks yourself, you need to pass a series of ENV vars for each PHP version you want to test. For example, the above chart is generated with::

Without opcache doing optimizations, the `bin/jit.php` is actually able to get close to native PHP with ack(3,9) and mandelbrot (without opcache) for 7.3 and 7.4. It's even able to hang with PHP 8's experimental JIT compiler for ack(3,9). For ack(3,10) it's able to be the fastest execution method.  

Most other tests are actually WAY slower with the `bin/jit.php` compiler. That's because the test itself is slower than the baseline time to parse and compile a file (about 0.2 seconds right now).

And note that this is running the compiler on top of PHP. At some point, the goal is to get the compiler to compile itself, hopefully cutting the time to compile down by at least a few hundred percent.

Simply look at the difference between everything and the "compiled time" column (which is the result of the AOT compiler generating a binary). This shows the potential in this compilation approach. If we can solve the overhead of parsing/compiling in PHP for the `bin/jit.php` examples, then man could this fly...

So yeah, there's definitely potential here... *evil grin*
