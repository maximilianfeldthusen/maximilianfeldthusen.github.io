---
title: "Asynchronous programming in Rust"
layout: post
date: 2023-08-12 22:44
headerImage: false
tag:
- asynchronous
- rust
- programming
star: true
category: blog
author: torbenfeldthusen
description: Rust Programming
---


Asynchronous programming in Rust

Take a look at how async-await works in Rust.

Asynchronous programming: Incredibly useful but difficult to learn. You can't avoid async programming to create a fast and reactive application. Applications with a high amount of file or network I/O or with a GUI that should always be reactive benefit tremendously from async programming. Tasks can be executed in the background while the user still makes inputs. Async programming is possible in many languages, each with different styles and syntax. Rust is no exception. In Rust, this feature is called async-await.

While async-await has been an integral part of Rust since version 1.39.0, most applications depend on community crates. In Rust, except for a larger binary, async-await comes with zero costs. This article gives you an insight into asynchronous programming in Rust.

Under the hood

To get a basic understanding of async-await in Rust, you literally start in the middle.

The center of async-await is the future trait, which declares the method poll (I cover this in more detail below). If a value can be computed asynchronously, the related type should implement the future trait. The poll method is called repeatedly until the final value is available.

At this point, you could repeatedly call the poll method from your synchronous application manually in order to get the final value. However, since I'm talking about asynchronous programming, you can hand over this task to another component: the runtime. So before you can make use of the async syntax, a runtime must be present. I use the runtime from the tokio community crate in the following examples.

A handy way of making the tokio runtime available is to use the #[tokio::main] macro on your main function:

```rust

#[tokio::main]
async fn main(){
    println!("Start!");
    sleep(Duration::from_secs(1)).await;
    println!("End after 1 second");
}

```

When the runtime is available, you can now await futures. Awaiting means that further executions stop here as long as the future needs to be completed. The await method causes the runtime to invoke the poll method, which will drive the future to completion.

In the above example, the tokios sleep function returns a future that finishes when the specified duration has passed. By awaiting this future, the related poll method is repeatedly called until the future completes. Furthermore, the main() function also returns a future because of the async keyword before the fn.

So if you see a function marked with async:

```rust

async fn foo() -> usize { /**/ }

```

Then it is just syntactic sugar for:
    
```rust

fn foo() -> impl Future<Output = usize> { async { /**/ } }

```

Pinning and boxing

To remove some of the shrouds and clouds of async-await in Rust, you must understand pinning and boxing.

If you are dealing with async-await, you will relatively quickly step over the terms boxing and pinning. Since I find that the available explanations on the subject are rather difficult to understand, I have set myself the goal of explaining the issue more easily.

Sometimes it is necessary to have objects that are guaranteed not to be moved in memory. This comes into effect when you have a self-referential type:

```rust

struct MustBePinned {
    a: int16,
    b: &int16 
}

```



If member b is a reference (pointer) to member a of the same instance, then reference b becomes invalid when the instance is moved because the location of member a has changed but b still points to the previous location. You can find a more comprehensive example of a self-referential type in the Rust Async book. All you need to know now is that an instance of MustBePinned should not be moved in memory. Types like MustBePinned do not implement the Unpin trait, which would allow them to move within memory safely. In other words, MustBePinned is !Unpin.

Back to the future: By default, a future is also !Unpin; thus, it should not be moved in memory. So how do you handle those types? You pin and box them.


The Pin<T> type wraps pointer types, guaranteeing that the values behind the pointer won't be moved. The Pin<T> type ensures this by not providing a mutable reference of the wrapped type. The type will be pinned for the lifetime of the object. If you accidentally pin a type that implements Unpin (which is safe to move), it won't have any effect.

In practice: If you want to return a future (!Unpin) from a function, you must box it. Using Box<T> causes the type to be allocated on the heap instead of the stack and thus ensures that it can outlive the current function without being moved. In particular, if you want to hand over a future, you can only hand over a pointer to it as the future must be of type Pin<Box<dyn Future>>.

Using async-wait, you will certainly stumble upon this boxing and pinning syntax. To wrap this topic up, you just have to remember this:

   Rust does not know whether a type can be safely moved.
    Types that shouldn't be moved must be wrapped inside Pin<T>.
    Most types are Unpinned types. They implement the trait Unpin and can be freely moved within memory.
    If a type is wrapped inside Pin<T> and the wrapped type is !Unpin, it is not possible to get a mutable reference out of it.
    Futures created by the async keyword are !Unpin and thus must be pinned.


Future trait

In the future trait, everything comes together:

```rust

pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

```

Here is a simple example of how to implement the future trait:

```rust

struct  MyCounterFuture {
 cnt : u32,
 cnt_final : u32
}

impl MyCounterFuture {
 pub fn new(final_value : u32) -> Self {
  Self {
   cnt : 0,
   cnt_final : final_value
  }
 }
}

impl Future for MyCounterFuture {
 type Output = u32;

 fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<u32>{
  self.cnt += 1;
  if self.cnt >= self.cnt_final {
   println!("Counting finished");
   return Poll::Ready(self.cnt_final);
  }

  cx.waker().wake_by_ref();
  Poll::Pending
 }
}

#[tokio::main]
async fn main(){
 let my_counter = MyCounterFuture::new(42);

 let final_value = my_counter.await;
 println!("Final value: {}", final_value);
}

```

Here is a simple example of how the future trait is implemented manually: The future is initialized with a value to which it shall count, stored in cnt_final. Each time the poll method is invoked, the internal value cnt gets incremented by one. If cnt is less than cnt_final, the future signals the waker of the runtime that the future is ready to be polled again. The return value of Poll::Pending signals that the future has not completed yet. After cnt is >= cnt_final, the poll function returns with Poll::Ready, signaling that the future has completed and providing the final value.

This is just a simple example, and of course, there are other things to take care of. If you consider creating your own futures, I highly suggest reading the chapter Async in depth in the documentation of the tokio crate.


Wrap up

Before I wrap things up, here is some additional information that I consider useful:

    Create a new pinned and boxed type using Box::pin.
    The futures crate provides the type BoxFuture which lets you define a future as return type of a function.
    The async_trait allows you to define an async function in traits (which is currently not allowed).
    The pin-utils crate provides macros to pin values.
    The tokios try_join! macro (a)waits on multiple futures which return a Result<T, E>.


Once the first hurdles have been overcome, async programming in Rust is straightforward. You don't even have to implement the future trait in your own types if you can outsource code that can be executed in parallel in an async function. In Rust, single-threaded and multi-threaded runtimes are available, so you can benefit from async programming even in embedded environments.





This work is licensed under a Creative Commons Attribution-Share Alike 4.0 International License.
