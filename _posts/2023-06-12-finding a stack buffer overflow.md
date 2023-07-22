
---
title: "finding a stack buffer overflow"
layout: post
date: 2023-06-12 22:44
headerImage: false
tag:
- buffer overflow
- cpp
- reverse engineering
star: true
category: blog
author: torbenfeldthusen
description: Reverse Engineering
---


One of the danger of C-style arrays is that their length is not attached to the pointer that points to their beginning. This means that there are lots of unsafe library functions that might write beyond the allocated space.

One such example is sprintf function:

```c
int sprintf(char *str, const char *format, ...);
```
It just takes a naked pointer to the buffer and a format string, as well as values to format. If the buffer in str is not large enough, sprintf will not know it and just happily write beyond the bounds.

Take the following program which writes a long string to a buffer which is too short.

```cpp
#include <stdio.h>
#include <stdlib.h>

int main(int argc, char **argv) {
    char buffer[10];

    sprintf(buffer, "This string is too long for the buffer");

    return 0;
}
```
When you compile it with a recent enough version of GCC with -Wall -Wpedantic, it will already figure this out in this simple example:

```cpp
$ gcc -Wall -Wpedantic -g overflow.c
overflow.c:7:32: warning: 'This string is too long for ...' directive writing 38 bytes into a region of size 10 [-Wformat-overflow=]
     sprintf(buffer, "This string is too long for the buffer");
                      ~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~
overflow.c:7:5: note: 'sprintf' output 39 bytes into a destination of size 10
     sprintf(buffer, "This string is too long for the buffer");
  ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```
When you execute the program, it will just crash with a segmentation fault. Let us assume that we do not know where this error came from and try to find it with the use of tools. In reality the problem is more intricate and not as obvious as in this minimal example.
GDB

Running it in GDB gives this output:

```
Program received signal SIGSEGV, Segmentation fault.

0x0000000000400511 in main (argc=1, argv=0x7fffffffdec8) at overflow.c:10
```
And this is the backtrace:
```
#0  0x0000000000400511 in main (argc=1, argv=0x7fffffffdec8) at overflow.c:10

```
Line 10 is the end of the program, so that is not particularly helpful really. We need better tools.
Valgrind

One way to check this is using valgrind:
```c
$ valgrind ./a.out
==26992== Memcheck, a memory error detector
==26992== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==26992== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==26992== Command: ./a.out
==26992==
==26992== Jump to the invalid address stated on the next line
==26992==    at 0x6F6620676E6F6C20: ???
==26992==    by 0x7562206568742071: ???
==26992==    by 0x72656665: ???
==26992==    by 0x10003FFFF: ???
==26992==    by 0x4004E5: ??? (in a.out)
==26992==  Address 0x6f6620676e6f6c20 is not stack'd, malloc'd or (recently) free'd
==26992==
==26992==
==26992== Process terminating with default action of signal 11 (SIGSEGV): dumping core
==26992==  Bad permissions for mapped region at address 0x6F6620676E6F6C20
==26992==    at 0x6F6620676E6F6C20: ???
==26992==    by 0x7562206568742071: ???
==26992==    by 0x72656665: ???
==26992==    by 0x10003FFFF: ???
==26992==    by 0x4004E5: ??? (in a.out)
==26992==
==26992== HEAP SUMMARY:
==26992==     in use at exit: 0 bytes in 0 blocks
==26992==   total heap usage: 0 allocs, 0 frees, 0 bytes allocated
==26992==
==26992== All heap blocks were freed -- no leaks are possible
==26992==
==26992== For counts of detected and suppressed errors, rerun with: -v
==26992== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```
Valgrind is not the right tool for finding static buffer overflows, therefore it is not helpful here. If we had used a dynamically allocated buffer, like in the following code example, it would work much better. Let us change the following in the above program:
```c
//char buffer[10];
char *buffer = malloc(10);
```
When we run Valgrind again, we get this:
```c
$ valgrind ./a.out
==27320== Memcheck, a memory error detector
==27320== Copyright (C) 2002-2017, and GNU GPL'd, by Julian Seward et al.
==27320== Using Valgrind-3.13.0 and LibVEX; rerun with -h for copyright info
==27320== Command: ./a.out
==27320==
==27320== Invalid write of size 1
==27320==    at 0x4C35F55: mempcpy (vg_replace_strmem.c:1524)
==27320==    by 0x4EB8AF7: _IO_default_xsputn (in /usr/lib64/libc-2.27.so)
==27320==    by 0x4E8A849: vfprintf (in /usr/lib64/libc-2.27.so)
==27320==    by 0x4EAD760: vsprintf (in /usr/lib64/libc-2.27.so)
==27320==    by 0x4E93763: sprintf (in /usr/lib64/libc-2.27.so)
==27320==    by 0x400568: main (in /home/mu/Projekte/eigene-webseite/articles/finding-stack-buffer-overflows/a.out)
==27320==  Address 0x51fa065 is 21 bytes after a block of size 16 in arena "client"
==27320==
==27320== Invalid write of size 1
==27320==    at 0x4EAD76F: vsprintf (in /usr/lib64/libc-2.27.so)
==27320==    by 0x4E93763: sprintf (in /usr/lib64/libc-2.27.so)
==27320==    by 0x400568: main (in /home/mu/Projekte/eigene-webseite/articles/finding-stack-buffer-overflows/a.out)
==27320==  Address 0x51fa066 is 22 bytes after a block of size 16 in arena "client"
==27320==
==27320==
==27320== HEAP SUMMARY:
==27320==     in use at exit: 10 bytes in 1 blocks
==27320==   total heap usage: 1 allocs, 0 frees, 10 bytes allocated
==27320==
==27320== LEAK SUMMARY:
==27320==    definitely lost: 10 bytes in 1 blocks
==27320==    indirectly lost: 0 bytes in 0 blocks
==27320==      possibly lost: 0 bytes in 0 blocks
==27320==    still reachable: 0 bytes in 0 blocks
==27320==         suppressed: 0 bytes in 0 blocks
==27320== Rerun with --leak-check=full to see details of leaked memory
==27320==
==27320== For counts of detected and suppressed errors, rerun with: -v
==27320== ERROR SUMMARY: 29 errors from 2 contexts (suppressed: 0 from 0)
```
Here you can neatly see that the error happens within sprintf and this should be enough cue to track down this bug.
Address sanitizer

The best tool for this job is the address sanitizer. You need a somewhat recent version of GCC or Clang to have it built in. And it only works on Linux. Compile the code with the sanitizer:
```cpp
gcc -Wall -Wpedantic -fsanitize=address -g overflow.c
```
When we now let the program run and we get the following output:
```c
$ ./a.out
=================================================================
==28566==ERROR: AddressSanitizer: stack-buffer-overflow on address 0x7ffe6256d1fa at pc 0x7fbbab43705f bp 0x7ffe6256d0c0 sp 0x7ffe6256c850
WRITE of size 39 at 0x7ffe6256d1fa thread T0
    #0 0x7fbbab43705e in vsprintf (/lib64/libasan.so.5+0x4f05e)
    #1 0x7fbbab4373de in sprintf (/lib64/libasan.so.5+0x4f3de)
    #2 0x40086d in main overflow.c:7
    #3 0x7fbbab04c24a in __libc_start_main (/lib64/libc.so.6+0x2324a)
    #4 0x400719 in _start (a.out+0x400719)

Address 0x7ffe6256d1fa is located in stack of thread T0 at offset 42 in frame
    #0 0x4007e5 in main overflow.c:4

  This frame has 1 object(s):
    [32, 42) 'buffer' <== Memory access at offset 42 overflows this variable
```
HINT: this may be a false positive if your program uses some custom stack unwind mechanism or swapcontext
```cpp
      (longjmp and C++ exceptions *are* supported)
SUMMARY: AddressSanitizer: stack-buffer-overflow (/lib64/libasan.so.5+0x4f05e) in vsprintf
Shadow bytes around the buggy address:
  0x10004c4a59e0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10004c4a59f0: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10004c4a5a00: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10004c4a5a10: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10004c4a5a20: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
=>0x10004c4a5a30: 00 00 00 00 00 00 00 00 00 00 f1 f1 f1 f1 00[02]
  0x10004c4a5a40: f2 f2 f3 f3 f3 f3 00 00 00 00 00 00 00 00 00 00
  0x10004c4a5a50: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10004c4a5a60: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10004c4a5a70: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
  0x10004c4a5a80: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
Shadow byte legend (one shadow byte represents 8 application bytes):
  Addressable:           00
  Partially addressable: 01 02 03 04 05 06 07
  Heap left redzone:       fa
  Freed heap region:       fd
  Stack left redzone:      f1
  Stack mid redzone:       f2
  Stack right redzone:     f3
  Stack after return:      f5
  Stack use after scope:   f8
  Global redzone:          f9
  Global init order:       f6
  Poisoned by user:        f7
  Container overflow:      fc
  Array cookie:            ac
  Intra object redzone:    bb
  ASan internal:           fe
  Left alloca redzone:     ca
  Right alloca redzone:    cb
==28566==ABORTING
```
We see that it is a “stack buffer overflow” and it happens in the sprintf. The memory map at the bottom shows us the amount of space that is allocated on the stack and where we wanted to write. This makes finding the bug really easy.
snprintf

The whole problem only arose because we used a function with the assumption that the allocated buffer is large enough. There is on hard check in place and this can lead to crashes or even security risks for our application. Therefore it is better to use something safe.

In C this is done with the snprintf function:
```
int snprintf(char *str, size_t size, const char *format, ...);
```
This is passed the length of the buffer and returns the number of characters that would have been written. Therefore we need to change the code:
```
int const buffer_size = 10;
int const chars_written = snprintf(buffer, buffer_size, "This string is too long for the buffer");
if (chars_written >= buffer_size) {
    fprintf(stderr, "The allocated space was insufficient, allocate more!\n");
    abort();
}
```
This way we make sure that the buffer is long enough. Otherwise we have an error message. We could instead of calling abort() also try to allocate a buffer that is long enough and try again.
std::ostringstream and boost::format¶

At this point one is tempted to write a wrapper function for that and it already exists: C++ has the ostringstream which allows building a string from parts. The Boost format class provides printf-style format syntax without all the pain of manually allocating buffers.



