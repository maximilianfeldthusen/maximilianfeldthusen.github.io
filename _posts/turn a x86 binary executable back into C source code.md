---
title: "Turn a x86 binary back into C code"
layout: post
date: 2023-05-25 22:44
headerImage: false
tag:
- assembly
- c
- reverse engineering
star: true
category: blog
author: torbenfeldthusen
description: Reverse Engineering
---



 

 * Objective: turn a x86 binary executable back into C source code. 
 * Understand how the compiler turns C into assembly code. 
 * Low-level OS structures and executable file format.
 
Arithmetic Instructions
```c
mov eax,2 ; eax = 2 
mov ebx,3 ; ebx = 3
add eax,ebx ; eax = eax + ebx 
sub ebx, 2 ; ebx = ebx - 2
```
Accessing Memory
```c
mox eax, [1234] ; eax = *(int*)1234 
mov ebx, 1234 ; ebx = 1234 
mov eax, [ebx] ; eax = *ebx 
mov [ebx], eax ; *ebx = eax 

```
Conditional Branches

```c
cmp eax, 2 ; compare eax with 2 
je label1 ; if(eax==2) goto label1
 ja label2 ; if(eax>2) goto label2
jb label3 ; if(eax<2) goto label3 
jbe label4 ; if(eax<=2) goto label4
 jne label5 ; if(eax!=2) goto label5
 jmp label6 ; unconditional goto label6

```
Function calls

First calling a function:
call func ; store return address on the stack and jump to func 
The first operations is to save the return pointer:
```c
pop esi ; save esi 
Right before leaving the function:
pop esi ; restore esi
ret ; read return address from the stack and jump to it 
```
Modern Compiler Architecture

C code --> Parsing --> Intermediate representation --> optimization --> 
Low-level intermediate representation --> register allocation --> x86 assembly


High-level Optimizations

Inlining

For example, the function c:
```c
int foo(int a, int b){
     return a+b }
 c = foo(a, b+1) 
```

translates to 
```c
c = a+b+1
```
 
Loop unrolling

The loop:
```c
for(i=0; i<2; i++){
      a[i]=0;
 }
 
```

becomes

```c
   a[0]=0; 
   a[1]=0; 

```
Loop-invariant code motion
```c
The loop:
for (i = 0; i < 2; i++) {
 a[i] = p + q; 
} 

```
becomes:

```c

temp = p + q;
for (i = 0; i < 2; i++) {
    a[i] = temp;
}
 
```
Common subexpression elimination

The variable attributions:

 * Objective: turn a x86 binary executable back into C source code. 
 * Understand how the compiler turns C into assembly code. 
 * Low-level OS structures and executable file format.
 
Arithmetic Instructions
```c
mov eax,2 ; eax = 2 
mov ebx,3 ; ebx = 3
add eax,ebx ; eax = eax + ebx 
sub ebx, 2 ; ebx = ebx - 2
```
Accessing Memory
```c
mox eax, [1234] ; eax = *(int*)1234 
mov ebx, 1234 ; ebx = 1234 
mov eax, [ebx] ; eax = *ebx 
mov [ebx], eax ; *ebx = eax 

```
Conditional Branches

```c
cmp eax, 2 ; compare eax with 2 
je label1 ; if(eax==2) goto label1
 ja label2 ; if(eax>2) goto label2
jb label3 ; if(eax<2) goto label3 
jbe label4 ; if(eax<=2) goto label4
 jne label5 ; if(eax!=2) goto label5
 jmp label6 ; unconditional goto label6

```
Function calls

First calling a function:
call func ; store return address on the stack and jump to func 
The first operations is to save the return pointer:
```c
pop esi ; save esi 
Right before leaving the function:
pop esi ; restore esi
ret ; read return address from the stack and jump to it 
```
Modern Compiler Architecture

C code --> Parsing --> Intermediate representation --> optimization --> 
Low-level intermediate representation --> register allocation --> x86 assembly


High-level Optimizations

Inlining

For example, the function c:
```c
int foo(int a, int b){
     return a+b }
 c = foo(a, b+1) 
```

translates to 
```c
c = a+b+1
```
 
Loop unrolling

The loop:
```c
for(i=0; i<2; i++){
      a[i]=0;
 } 
```

becomes

```c
   a[0]=0; 
   a[1]=0; 

```
Loop-invariant code motion
```c
The loop:
for (i = 0; i < 2; i++) {
 a[i] = p + q; 
} 
```
becomes:
```c
temp = p + q;
for (i = 0; i < 2; i++) {
    a[i] = temp;
}
 
```
Common subexpression elimination

The variable attributions:
```c
a = b + (z + 1)
p = q + (z + 1)
```
becomes
```c
temp = z + 1
a = b + z
p = q + z

```
Constant folding and propagation

The assignments:
```c
a = 3 + 5
b = a + 1
func(b)
```
Becomes:
```c
func(9)

```
Dead code elimination

Delete unnecessary code:
```c
a = 1
if (a < 0) {
printf(“ERROR!”)
}
```
to
```c
a = 1

```
Low-Level Optimizations

Strength reduction

Codes such as:
```c
y = x * 2
y = x * 15
```
Becomes:
```c
y = x + x
y = (x << 4) - x

```
Code block reordering

Codes such as :
```c
if (a < 10) goto l1
printf(“ERROR”)
goto label2
l1:
    printf(“OK”)
l2:
    return;
```
Becomes:
```c
if (a > 10) goto l1
printf(“OK”)
l2:
return
l1:
printf(“ERROR”)
goto l2

```
Register allocation

 * Memory access is slower than registers. 
 * Try to fit as many as local variables as possible in registers. 
 * The mapping of local variables to stack location and registers is not constant.
 
 

Instruction scheduling

Assembly code like:
```c
mov eax, [esi]
add eax, 1
mov ebx, [edi]
add ebx, 1
```
Becomes:
```c
mov eax, [esi]
mov ebx, [edi]
add eax, 1
add ebx, 1

a = b + (z + 1)
p = q + (z + 1)
```
becomes
```c
temp = z + 1
a = b + z
p = q + z

```
Constant folding and propagation

The assignments:
```c
a = 3 + 5
b = a + 1
func(b)
```
Becomes:
```c
func(9)

```
Dead code elimination

Delete unnecessary code:
```c
a = 1
if (a < 0) {
printf(“ERROR!”)
}
```
to
```c
a = 1

```
Low-Level Optimizations

Strength reduction

Codes such as:
```c
y = x * 2
y = x * 15
```
Becomes:
```c
y = x + x
y = (x << 4) - x

```
Code block reordering

Codes such as :
```c
if (a < 10) goto l1
printf(“ERROR”)
goto label2
l1:
    printf(“OK”)
l2:
    return;
```
Becomes:
```c
if (a > 10) goto l1
printf(“OK”)
l2:
return
l1:
printf(“ERROR”)
goto l2

```
Register allocation

 * Memory access is slower than registers. 
 * Try to fit as many as local variables as possible in registers. 
 * The mapping of local variables to stack location and registers is not constant.
 

 * Objective: turn a x86 binary executable back into C source code. 
 * Understand how the compiler turns C into assembly code. 
 * Low-level OS structures and executable file format.
 
Arithmetic Instructions
```c
mov eax,2 ; eax = 2 
mov ebx,3 ; ebx = 3
add eax,ebx ; eax = eax + ebx 
sub ebx, 2 ; ebx = ebx - 2
```
Accessing Memory
```c
mox eax, [1234] ; eax = *(int*)1234 
mov ebx, 1234 ; ebx = 1234 
mov eax, [ebx] ; eax = *ebx 
mov [ebx], eax ; *ebx = eax 

```
Conditional Branches

```c
cmp eax, 2 ; compare eax with 2 
je label1 ; if(eax==2) goto label1
 ja label2 ; if(eax>2) goto label2
jb label3 ; if(eax<2) goto label3 
jbe label4 ; if(eax<=2) goto label4
 jne label5 ; if(eax!=2) goto label5
 jmp label6 ; unconditional goto label6

```
Function calls

First calling a function:
call func ; store return address on the stack and jump to func 
The first operations is to save the return pointer:
```c
pop esi ; save esi 
Right before leaving the function:
pop esi ; restore esi
ret ; read return address from the stack and jump to it 
```
Modern Compiler Architecture

C code --> Parsing --> Intermediate representation --> optimization --> 
Low-level intermediate representation --> register allocation --> x86 assembly


High-level Optimizations

Inlining

For example, the function c:
```c
int foo(int a, int b){
     return a+b }
 c = foo(a, b+1) 

```

translates to 
```c
c = a+b+1
```
 
Loop unrolling

The loop:
```c
for(i=0; i<2; i++){
      a[i]=0;
 } 
```
```c
becomes
   a[0]=0; 
   a[1]=0; 

```
Loop-invariant code motion

The loop:
```c
for (i = 0; i < 2; i++) {
 a[i] = p + q; 
} 
```
becomes:
```c
temp = p + q;
for (i = 0; i < 2; i++) {
    a[i] = temp;
}
 
```
Common subexpression elimination

The variable attributions:

 * Objective: turn a x86 binary executable back into C source code. 
 * Understand how the compiler turns C into assembly code. 
 * Low-level OS structures and executable file format.
 
Arithmetic Instructions
```c
mov eax,2 ; eax = 2 
mov ebx,3 ; ebx = 3
add eax,ebx ; eax = eax + ebx 
sub ebx, 2 ; ebx = ebx - 2
```
Accessing Memory
```c
mox eax, [1234] ; eax = *(int*)1234 
mov ebx, 1234 ; ebx = 1234 
mov eax, [ebx] ; eax = *ebx 
mov [ebx], eax ; *ebx = eax 

```
Conditional Branches

```c
cmp eax, 2 ; compare eax with 2 
je label1 ; if(eax==2) goto label1
 ja label2 ; if(eax>2) goto label2
jb label3 ; if(eax<2) goto label3 
jbe label4 ; if(eax<=2) goto label4
 jne label5 ; if(eax!=2) goto label5
 jmp label6 ; unconditional goto label6

```
Function calls

First calling a function:
call func ; store return address on the stack and jump to func 
The first operations is to save the return pointer:
```c
pop esi ; save esi 
Right before leaving the function:
pop esi ; restore esi
ret ; read return address from the stack and jump to it 
```
Modern Compiler Architecture

C code --> Parsing --> Intermediate representation --> optimization --> 
Low-level intermediate representation --> register allocation --> x86 assembly


High-level Optimizations

Inlining

For example, the function c:
```c
int foo(int a, int b){
     return a+b }
 c = foo(a, b+1) 
```

translates to 
```c
c = a+b+1
```
 
Loop unrolling

The loop:
```c
for(i=0; i<2; i++){
      a[i]=0;
 } 
```

becomes

```c
   a[0]=0; 
   a[1]=0; 

```
Loop-invariant code motion
```c
The loop:
for (i = 0; i < 2; i++) {
 a[i] = p + q; 
} 
```
becomes:
```c
temp = p + q;
for (i = 0; i < 2; i++) {
    a[i] = temp;
}
 
```
Common subexpression elimination



The variable attributions:
```c
a = b + (z + 1)
p = q + (z + 1)
```
becomes
```c
temp = z + 1
a = b + z
p = q + z

```
Constant folding and propagation

The assignments:
```c
a = 3 + 5
b = a + 1
func(b)
```
Becomes:
```c
func(9)

```
Dead code elimination

Delete unnecessary code:
```c
a = 1
if (a < 0) {
printf(“ERROR!”)
}
```
to
```c
a = 1

```
Low-Level Optimizations

Strength reduction

Codes such as:
```c
y = x * 2
y = x * 15
```
Becomes:
```c
y = x + x
y = (x << 4) - x

```
Code block reordering

Codes such as :
```c
if (a < 10) goto l1
printf(“ERROR”)
goto label2
l1:
    printf(“OK”)
l2:
    return;
```
Becomes:
```c
if (a > 10) goto l1
printf(“OK”)
l2:
return
l1:
printf(“ERROR”)
goto l2

```
Register allocation

 * Memory access is slower than registers. 
 * Try to fit as many as local variables as possible in registers. 
 * The mapping of local variables to stack location and registers is not constant.
 
 

Instruction scheduling

Assembly code like:
```c
mov eax, [esi]
add eax, 1
mov ebx, [edi]
add ebx, 1
```
Becomes:
```c
mov eax, [esi]
mov ebx, [edi]
add eax, 1
add ebx, 1

a = b + (z + 1)
p = q + (z + 1)
```
becomes
```c
temp = z + 1
a = b + z
p = q + z

```
Constant folding and propagation

The assignments:
```c
a = 3 + 5
b = a + 1
func(b)
```
Becomes:
```c
func(9)

```
Dead code elimination

Delete unnecessary code:
```c
a = 1
if (a < 0) {
printf(“ERROR!”)
}
```
to
```c
a = 1

```
Low-Level Optimizations

Strength reduction

Codes such as:
```c
y = x * 2
y = x * 15
```
Becomes:
```c
y = x + x
y = (x << 4) - x

```
Code block reordering

Codes such as :
```c
if (a < 10) goto l1
printf(“ERROR”)
goto label2
l1:
    printf(“OK”)
l2:
    return;
```
Becomes:
```c
if (a > 10) goto l1
printf(“OK”)
l2:
return
l1:
printf(“ERROR”)
goto l2

```
Register allocation

 * Memory access is slower than registers. 
 * Try to fit as many as local variables as possible in registers. 
 * The mapping of local variables to stack location and registers is not constant.
 
 

Instruction scheduling

Assembly code like:
```c
mov eax, [esi]
add eax, 1
mov ebx, [edi]
add ebx, 1
```
Becomes:
```c
mov eax, [esi]
mov ebx, [edi]
add eax, 1
add ebx, 1

```
Instruction scheduling

Assembly code like:
```c
mov eax, [esi]
add eax, 1
mov ebx, [edi]
add ebx, 1
```
Becomes:
```c
mov eax, [esi]
mov ebx, [edi]
add eax, 1
add ebx, 1

```
