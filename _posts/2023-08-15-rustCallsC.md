---
title: "Rust calls to C libraries"
layout: post
date: 2023-08-12 22:44
headerImage: false
tag:
- rust
- programming
star: true
category: blog
author: torbenfeldthusen
description: Rust Programming
---




Introducing Rust calls to C library functions
The Rust FFI and the bindgen utility are well designed for making Rust calls out to C libraries. Rust talks readily to C and thereby to any other language that talks to C.

Why call C functions from Rust? The short answer is software libraries. A longer answer touches on where C stands among programming languages in general and towards Rust in particular. C, C++, and Rust are systems languages, which give programmers access to machine-level data types and operations. Among these three systems languages, C remains the dominant one. The kernels of modern operating systems are written mainly in C, with assembly language accounting for the rest. The standard system libraries for input and output, number crunching, cryptography, security, networking, internationalization, string processing, memory management, and more, are likewise written mostly in C. These libraries represent a vast infrastructure for applications written in any other language. Rust is well along the way to providing fine libraries of its own, but C libraries—​around since the 1970s and still growing—​are a resource not to be ignored. Finally, C is still the lingua franca among programming languages: most languages can talk to C and, through C, to any other language that does so.
Two proof-of-concept examples

Rust has an FFI (Foreign Function 
Interface) that supports calls to C 
functions. An issue for any FFI is
whether the calling language covers 
the data types in the called 
language.For example, ctypes is an FFI
for calls from Python into C, but
Python doesn't cover the unsigned 
integer types available in C. As a 
result, ctypes must resort to 
workarounds.

By contrast, Rust covers all the 
primitive (that is, machine-level) 
types in C. For example, the Rust i32
type matches the C int type. C 
specifies only that the char type must
be one byte in size and other types,
such as int, must be at least this 
size; but nowadays every reasonable C
compiler supports a four-byte int, an
eight-byte double (in Rust, the f64 
type), and so on.

There is another challenge for an FFI
directed at C: Can the FFI handle C's 
raw pointers, including pointers to
arrays that count as strings in C? C
does not have a string type, but 
rather implements strings as character
arrays with a non-printing terminating
character, the null terminator of 
legend. By contrast, Rust has two 
string types: String and &str (string
slice). The question, then, is whether 
the Rust FFI can transform a C string 
into a Rust one—​and the answer is yes.

Pointers to structures also are common
in C. The reason is efficiency. By
default, a C structure is passed by
value (that is, by a byte-per-byte 
copy) when a structure is either an
argument passed to a function or a 
value returned from one. C structures,
like their Rust counterparts, can
include arrays and nest other 
structures and so be arbitrarily large
in size. Best practice in either 
language is to pass and return 
structures by reference, that is, by 
passing or returning the structure's
address rather than a copy of the 
structure. Once again, the Rust FFI is
up to the task of handling C pointers
to structures, which are common in C 
libraries.

The first code example focuses on 
calls to relatively simple C library 
functions such as abs (absolute value)
and sqrt (square root). These 
functions take non-pointer scalar 
arguments and return a non-pointer
scalar value. The second code example,
which covers strings and pointers to
structures, introduces the bindgen
utility, which generates Rust code
from C interface (header) files such
as math.h and time.h. C header files 
specify the calling syntax for C 
functions and define structures used
in such calls. The two code examples
are available on my homepage.
Calling relatively simple C functions

The first code example has four Rust
calls to C functions in the standard mathematics library: one call apiece to abs (absolute value) and pow (exponentiation), and two calls to sqrt (square root). The program can be built directly with the rustc compiler, or more conveniently with the cargo build command:


```rust



use std::os::raw::c_int;    // 32 bits
use std::os::raw::c_double; // 64 bits

// Import three functions from the standard library libc.
// Here are the Rust declarations for the C functions:
extern "C" {
    fn abs(num: c_int) -> c_int;
    fn sqrt(num: c_double) -> c_double;
    fn pow(num: c_double, power: c_double) -> c_double;
}

fn main() {
    let x: i32 = -123;
    println!("\nAbsolute value of {x}: {}.",
	     unsafe { abs(x) });

    let n: f64 = 9.0;
    let p: f64 = 3.0;
    println!("\n{n} raised to {p}: {}.",
	     unsafe { pow(n, p) });

    let mut y: f64 = 64.0;
    println!("\nSquare root of {y}: {}.",
	     unsafe { sqrt(y) });
    y = -3.14;
    println!("\nSquare root of {y}: {}.",
	     unsafe { sqrt(y) }); //** NaN = NotaNumber
}



```

The two use declarations at the top are for the Rust data types c_int and c_double, which match the C types int and double, respectively. The standard Rust module std::os::raw defines fourteen such types for C compatibility. The module std::ffi has the same fourteen type definitions together with support for strings.

The extern "C" block above the main function then declares the three C library functions called in the main function below. Each call uses the standard C function's name, but each call must occur within an unsafe block. As every programmer new to Rust discovers, the Rust compiler enforces memory safety with a vengeance. Other languages (in particular, C and C++) do not make the same guarantees. The unsafe block thus says: Rust takes no responsibility for whatever unsafe operations may occur in the external call.

The first program's output is:

Absolute value of -123: 123.
9 raised to 3: 729
Square root of 64: 8.
Square root of -3.14: NaN.

In the last output line, the NaN stands for Not a Number: the C sqrt library function expects a non-negative value as its argument, which means that the argument -3.14 generates NaN as the returned value.



Calling C functions involving pointers

C library functions in security, networking, string processing, memory management, and other areas regularly use pointers for efficiency. For example, the library function asctime (time as an ASCII string) expects a pointer to a structure as its single argument. A Rust call to a C function such as asctime is thus trickier than a call to sqrt, which involves neither pointers nor structures.

The C structure for the asctime function call is of type struct tm. A pointer to such a structure also is passed to library function mktime (make a time value). The structure breaks a time into units such as the year, the month, the hour, and so forth. The structure's fields are of type time_t, an alias for for either int (32 bits) or long (64 bits). The two library functions combine these broken-apart time pieces into a single value: asctime returns a string representation of the time, whereas mktime returns a time_t value that represents the number of elapsed seconds since the epoch, which is a time relative to which a system's clock and timestamp are determined. Typical epoch settings are January 1 00:00:00 (zero hours, minutes, and seconds) of either 1900 or 1970.

The C program below calls asctime and mktime, and uses another library function strftime to convert the mktime returned value into a formatted string. This program acts as a warm-up for the Rust version:
    

```c


#include <stdio.h>
#include <time.h>

int main () {
  struct tm sometime;  /* time broken out in detail */
  char buffer[80];
  int utc;

  sometime.tm_sec = 1;
  sometime.tm_min = 1;
  sometime.tm_hour = 1;
  sometime.tm_mday = 1;
  sometime.tm_mon = 1;
  sometime.tm_year = 1;
  sometime.tm_hour = 1;
  sometime.tm_wday = 1;
  sometime.tm_yday = 1;

  printf("Date and time: %s\n", asctime(&sometime));

  utc = mktime(&sometime);
  if( utc < 0 ) {
    fprintf(stderr, "Error: unable to make time using mktime\n");
  } else {
    printf("The integer value returned: %d\n", utc);
    strftime(buffer, sizeof(buffer), "%c", &sometime);
    printf("A more readable version: %s\n", buffer);
  }

  return 0;
}

```

The program outputs:

Date and time: Fri Feb  1 01:01:01 1901
The integer value returned: 2120218157
A more readable version: Fri Feb  1 01:01:01 1901

In summary, the Rust calls to library
functions asctime and mktime must deal
with two issues:

Passing a raw pointer as the
single argument to each library
function.

Converting the C string returned
from asctime into a Rust string.

Rust calls to asctime and mktime

The bindgen utility generates Rust
support code from C header files such
as math.h and time.h. In this example,
a simplified version of time.h will do
but with two changes from the original:
The built-in type int is used 	
instead of the alias type time_t.
The bindgen utility can handle the
time_t type but generates some 
 distracting warnings along the way 
 because time_t does not follow Rust 
 naming conventions: in time_t an 
 underscore separates the t at the end
 from the time that comes first; Rust
 would prefer a CamelCase name such as
 TimeT.

The type struct tm type is given 
StructTM as an alias for the same 
reason.

Here is the simplified header file 
with declarations for mktime and 
asctime at the bottom:
    
```c

typedef struct tm {
    int tm_sec;    /* seconds */
    int tm_min;    /* minutes */
    int tm_hour;   /* hours */
    int tm_mday;   /* day of the month */
    int tm_mon;    /* month */
    int tm_year;   /* year */
    int tm_wday;   /* day of the week */
    int tm_yday;   /* day in the year */
    int tm_isdst;  /* daylight saving time */
} StructTM;

extern int mktime(StructTM*);
extern char* asctime(StructTM*);


```

With bindgen installed, % as the command-line prompt, and mytime.h as the header file above, the following command generates the required Rust code and saves it in the file mytime.rs:

```rust

% bindgen mytime.h > mytime.rs

```

Here is the relevant part of mytime.rs:
    
```rust

/* automatically generated by rust-bindgen 0.61.0 */


#[repr(C)]
#[derive(Debug, Copy, Clone)]
pub struct tm {
    pub tm_sec: ::std::os::raw::c_int,
    pub tm_min: ::std::os::raw::c_int,
    pub tm_hour: ::std::os::raw::c_int,
    pub tm_mday: ::std::os::raw::c_int,
    pub tm_mon: ::std::os::raw::c_int,
    pub tm_year: ::std::os::raw::c_int,
    pub tm_wday: ::std::os::raw::c_int,
    pub tm_yday: ::std::os::raw::c_int,
    pub tm_isdst: ::std::os::raw::c_int,
}

pub type StructTM = tm;

extern "C" {
    pub fn mktime(arg1: *mut StructTM) -> ::std::os::raw::c_int;
}

extern "C" {
    pub fn asctime(arg1: *mut StructTM) -> *mut ::std::os::raw::c_char;
}

#[test]
fn bindgen_test_layout_tm() {
    const UNINIT: ::std::mem::MaybeUninit<tm> =
       ::std::mem::MaybeUninit::uninit();
    let ptr = UNINIT.as_ptr();
    assert_eq!(
        ::std::mem::size_of::<tm>(),
        36usize,
        concat!("Size of: ", stringify!(tm))
    );
    ...

    
```

The Rust structure struct tm, like the C original, contains nine 4-byte integer fields. The field names are the same in C and Rust. The extern "C" blocks declare the library functions asctime and mktime as taking one argument apiece, a raw pointer to a mutable StructTM instance. (The library functions may mutate the structure via the pointer passed as an argument.)

The remaining code, under the #[test] attribute, tests the layout of the Rust version of the time structure. The test can be run with the cargo test command. At issue is that C does not specify how the compiler must lay out the fields of a structure. For example, the C struct tm starts out with the field tm_sec for the second; but C does not require that the compiled version has this field as the first. In any case, the Rust tests should succeed and the Rust calls to the library functions should work as expected.
Getting the second example up and running

The code generated from bindgen does not include a main function and, therefore, is a natural module. Below is the main function with the StructTM initialization and the calls to asctime and mktime:
    
```rust

mod mytime;
use mytime::*;
use std::ffi::CStr;

fn main() {
    let mut sometime  = StructTM {
        tm_year: 1,
        tm_mon: 1,
        tm_mday: 1,
        tm_hour: 1,
        tm_min: 1,
        tm_sec: 1,
        tm_isdst: -1,
        tm_wday: 1,
        tm_yday: 1
    };

    unsafe {
        let c_ptr = &mut sometime; // raw pointer

        // make the call, convert and then own
        // the returned C string
        let char_ptr = asctime(c_ptr);
        let c_str = CStr::from_ptr(char_ptr);
        println!("{:#?}", c_str.to_str());

        let utc = mktime(c_ptr);
        println!("{}", utc);
    }
}

```

The Rust code can be compiled (using either rustc directly or cargo) and then run. 

The output is:

Ok(
    "Mon Feb  1 01:01:01 1901\n",
)
2120218157

The calls to the C functions asctime and mktime again must occur inside an unsafe block, as the Rust compiler cannot be held responsible for any memory-safety mischief in these external functions. For the record, asctime and mktime are well behaved. In the calls to both functions, the argument is the raw pointer ptr, which holds the (stack) address of the sometime structure.

The call to asctime is the trickier of the two calls because this function returns a pointer to a C char, the character M in Mon of the text output. Yet the Rust compiler does not know where the C string (the null-terminated array of char) is stored. In the static area of memory? On the heap? The array used by the asctime function to store the text representation of the time is, in fact, in the static area of memory. In any case, the C-to-Rust string conversion is done in two steps to avoid compile-time errors:

   1. The call Cstr::from_ptr(char_ptr) converts the C string to a Rust string and returns a reference stored in the c_str variable.

  2. The call to c_str.to_str() ensures that c_str is the owner.

The Rust code does not generate a human-readable version of the integer value returned from mktime, which is left as an exercise for the interested. The Rust module chrono::format includes a strftime function, which can be used like the C function of the same name to get a text representation of the time.
Calling C with FFI and bindgen

The Rust FFI and the bindgen utility are well designed for making Rust calls out to C libraries, whether standard or third-party. Rust talks readily to C and thereby to any other language that talks to C. For calling relatively simple library functions such as sqrt, the Rust FFI is straightforward because Rust's primitive data types cover their C counterparts.

For more complicated interchanges—​in particular, Rust calls to C library functions such as asctime and mktime that involve structures and pointers—​the bindgen utility is the way to go. This utility generates the support code together with appropriate tests. Of course, the Rust compiler cannot assume that C code measures up to Rust standards when it comes to memory safety; hence, calls from Rust to C must occur in unsafe blocks.



This work is licensed under a Creative Commons Attribution-Share Alike 4.0 International License.
