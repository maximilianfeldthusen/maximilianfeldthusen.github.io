---
title: "Reverse engineering of Rust binaries"
layout: post
date: 2022-12-05 22:48
image: /assets/images/markdown.jpg
headerImage: false
tag:
- Rust
- cpp
- extra
category: blog
author: torbenfeldthusen
description: reverse engineering
---


Reverse Engineering of Rust Binaries


**Introduction**

The Rust programming language is like rust on a vehicle for malware analysts and reverse engineers. The adoption of the language by malware authors spreads like cancer the longer it is in active development. This is due to convenient static linking and support for many operating systems, yielding a binary with little to no dependencies. These features are excellent for the distribution of malware. Every time we need to reverse engineer a Rust binary, we would rather embrace the sweet release of death. We need to take a deeper look into why our current tools are failing to help us perform our jobs effectively. As always with new technologies, it will take some time for our tools to catch up with new compilers. To better understand Rust, let’s have a look at its official definition.

Rust is a multi-paradigm, general-purpose programming language. Rust emphasizes performance, type safety, and concurrency. Rust enforces memory safety—that is, that all references point to valid memory—without requiring the use of a garbage collector or reference counting present in other memory-safe languages. To simultaneously enforce memory safety and prevent concurrent data races, Rust’s borrow checker tracks the object lifetime and variable scope of all references in a program during compilation. Rust is popular for systems programming, but also offers high-level features, including functional programming constructs. - Wikipedia

Now that you have a basic understanding of what Rust is, we will now break down some common misconceptions. After this, we will discuss the toolset and reverse engineering challenges.

**Rust’s Origin Story**

Rust, like pretty much all modern programming languages these days, was created using LLVM. The very language Rust developers hate, C or C++ is used to create Rust. Let that shit sink into your heads for a minute.



Compilers are not magic, they are usually created from lower level programming languages. Without an egg, how do you get chickens or your American breakfast in the morning, It must originate from somewhere.

If you need entertainment, tell a Rust fanboy or fangirl that Rust came from C++ and LLVM. Then watch them question everything they thought they knew. 

**Memory**

Don’t be fooled, although it has been stated that Rust does not have a garbage collector, this is based on a definition technicality. Yes, we are talking about semantics at this point, in my opinion. 

Rust does more memory garbage collection calculation during compile time, instead of during runtime.



Although Rust fanatics think the language is a safe haven with no vulnerabilities, you can still perform unsafe memory operations in Rust as well by using the unsafe keyword. In the unsafe block you can dereference raw pointers, implement unsafe traits, access or modify a mutable static variable, call unsafe functions or methods.

```rust

extern crate libc; 
use std::mem;
fn main() {
    unsafe {
        let my_num: *mut i32 = libc::malloc(mem::size_of::<i32>() as libc::size_t) as *mut i32;
        libc::free(my_num as *mut libc::c_void);
    }
}

```

It is important to note that although unsafe is used, the rustc compiler will still insert some memory safety features.

This makes rustc a very powerful compiler where the developer can completely bypass safety to do lower level operations just like in C, Assembly or C++.

Rust provides “memory safe” options much like in C++, when you use std::string the compiler will deallocate the object once it is no longer referenced. As with any technology, the language is not completely “memory safe” as they may advertise. With any application there can be vulnerabilities, humans did write the Rust programming language, developers can still insert unsafe code, and we are not infallible creatures.



We now learned that Rust developers can introduce unsafe memory operations into their applications. Also, we now know that garbage collection still happens but more so, it’s added to the program during compile time rather than handled during runtime. Again, if you have a “memory safe” language, the cleanup of memory must be handled for you somehow, that garbage has to go somewhere, it just does not disappear into the void magically.

**Rust Toolset**

In order to understand how to reverse engineer Rust binaries, we should have a basic understanding of the toolset provided to developers.

**rustc**

The compiler rustc is used to compile Rust code.

Most developers do not invoke rustc directly, but instead use cargo.

```rust
 cargo build --verbose 
```
cargo

Cargo is a Rust package manager, it will download package dependencies, compiles packages, distributes packages by uploading them to crates.io.

Think of it like pip for Python.

**Reverse Engineering Challenges**

This section will cover challenges we face with our existing tooling when reverse engineering Rust binaries.

Strings without NULL Termination

Rust will store strings differently than most compilers, it will store them without NULL termination between strings, then reference the very long strings with a table.

Rust will store strings differently than most compilers, it will store them as multiple strings all together without NULL terminators between them. The address of these very long strings will be referenced by pointer and size in a table.

We can emulate how Rust prints strings by using the format strings with %.*s and how it avoids using NULL terminators in the following code.


```cpp

#include <stdio.h>

#define STRING_SIZE   27
#define STRING_SIZE_0 13
#define STRING_SIZE_1 13

typedef struct _RUST_STRINGS {
  char *ascii;
  unsigned int size;
} RUST_STRINGS, *PRUST_STRINGS;

int main(int argc, char **argv){
  char g_strings[STRING_SIZE] = "Hello World!\nHello There!\n\0";
  RUST_STRINGS rstrings[2] = {
	  {(char *)*&g_strings, STRING_SIZE_0},
	  {(char *)&g_strings+STRING_SIZE_0, STRING_SIZE_1}};
  printf("%.*s", rstrings[0].size, rstrings[0].ascii);
  printf("%.*s", rstrings[1].size, rstrings[1].ascii);
  return 0;
}

```
 

While this may seem like a complete waste of time, there are a few benefits to storing strings without the NULL terminators this way.

Fewer calls to strlen() or other equivalent system calls

String manipulation, concatenation, truncation etc.

The functional equivalent code in Rust would be as follows.


```rust

fn main() {
	let string_0 = "Hello World!";
	let string_1 = "Hello There!";
    println!("{}", string_0);
    println!("{}", string_1);
}

```

Static Linking





 



NOTE: This Rust code is not 100% the equivalent, just covering the basic principles

We can combat string issues in Rust binaries by defining the proper table structures in your decompiler.

This type of string operations will potentially increase performance at the expense of your sanity.

Static Linking




