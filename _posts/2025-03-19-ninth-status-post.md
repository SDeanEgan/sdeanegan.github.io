---
tags: [Rust Project]
---
## Ninth Status Post

The next *Rust Programming Language* material covers concurrency. These are chapters sixteen and seventeen, which teach multithreading and asynchronous features. These amounted to far more depth than most topics seen so far; [there is even a continued reading book just for covering async Rust](https://rust-lang.github.io/async-book/). For this status post, I'll stick to some of the more primary concepts, because a write-up like this is better off as a jumping-off point. Enough is taught that I would imagine most programmers who made it this far can get started with using Rust's concurrency themselves, but I'm glad that these will be coming up again in the next project chapter. 

The Rust book proposes two contexts with concurrency, message passing and shared-state. Message passing refers to a task where data is transferred between threads using "channels", ensuring that only one thread owns the data at a time. This prevents race conditions by enforcing ownership transfer rather than allowing multiple threads to access the same memory simultaneously. Rust provides the `mpsc` (multiple producer, single consumer) channel for this purpose, allowing threads to send messages safely to a receiving thread. 

Shared-state concurrency, on the other hand, involves multiple threads accessing and modifying the same piece of data. These tasks or goals are organized around the principles of whether a type or data should be capable of being accessed in parallel across threads, or `Sync`, and whether it should be capable of being transferred between spawned threads in the first place, or `Send`. This requires synchronization mechanisms like the smart pointer `Mutex<T>`, to ensure that only one thread can mutate the data at a time, preventing race conditions. Because the Rust compiler enforces ownership and borrowing rules, shared-state concurrency often relies on `Arc<T>` to enable multiple threads to hold references to shared data while maintaining memory safety. Unlike message passing, this approach allows multiple threads to access and/or modify the same data, but it requires atomic primitives to enforce safety. A `Mutex<T>` ensures that only one thread can access the data at a time, preventing race conditions, while `Arc<T>` provides shared ownership, and combining these enables multiple threads to reference the same data mutably without violating Rust's ownership rules. This resembles the example used from the smart pointers chapter, but applied in the context of Rust's concurrency traits `Sync` and `Send`. 

```rust
use std::sync::{mpsc, Arc, Mutex};
use std::thread;

enum MainMessage { Incr, Get, Quit }

enum SpawnMessage { Get(usize) }


fn main() {
    // make a channel going in and a channel going out
    let (spawn_tx, main_rx) = mpsc::channel();
    let (main_tx, spawn_rx) = mpsc::channel();
    // Arc for shared ownership and Mutex for interior mutability
    let counter = Arc::new(Mutex::new(0));
    let thread_counter = Arc::clone(&counter);
    
    let spawn = thread::spawn(move || {
        // move keyword in closure captures ownership of thread_counter
        loop {
            match spawn_rx.recv().unwrap() {
                MainMessage::Quit => break,
                MainMessage::Incr => {
                    *thread_counter.lock().unwrap() += 1
                }
                MainMessage::Get => {
                    spawn_tx.send(
                        SpawnMessage::Get(
                            *thread_counter.lock().unwrap()
                        )
                    ).unwrap()
                }
            }
        }
    });
    
    let send_messages = [MainMessage::Incr, 
                         MainMessage::Incr,
                         MainMessage::Get, 
                         MainMessage::Quit];

    for msg in send_messages {
        main_tx.send(msg).unwrap();
    }
    
    spawn.join().unwrap(); // make a race condition by putting this last

    // use destructuring assignment to unpack c
    let SpawnMessage::Get(c) = main_rx.recv().unwrap();
    println!("All messages received: {}", *counter.lock().unwrap() == c);
    
}
```
The above code is somewhat long for an example, but illustrates message sending and shared-state principles. A channel is established into and out of the thread. Data transmitted across an `mpsc::channel` needs to be type homogeneous, which we accomplish by using variants of an enum for our messages. Both threads can access the same state with our counter variable because the `Arc` smart pointer allows shared ownership, and can be written to thanks to the internal `Mutex`. 

The chapter which follows delves into asynchronous programming. As a programming paradigm "async" isn't tied to a specific language but can be implemented using different techniques, such as callbacks, promises, futures, or async/await syntax. In Rust, it is achieved through the `async` and `await` keywords, combined with an "executor" that schedules and runs tasks. While async in Rust resembles the multithreading we've been discussing, it doesn't necessarily involve parallel execution (multiple threads) but rather allows tasks to run efficiently within a single thread. Async programming in Rust allows functions to run without blocking the execution of other tasks. Instead of waiting for an operation to complete, a block of async code or function is managed by an imported package called a runtime. This runtime is responsible for managing the flow of execution needed to allow a program to continue to accomplish further work while the process of waiting for some other data or computation remains ongoing. This is useful for handling tasks like network requests, file operations, or any process that takes time but does not need the CPU continuously.  

Async may be the topic with which I'm the least familiar of any covered in the Rust book, but I was surprised by how conceptually consistent the principles were kept with multithreading/parallelism. This is illustrated in *the book* by the chapter's use of a specially developed package `trpl`, which serves as the async runtime and uses naming to match with thread API so that functions which serve similar purposes can be better appreciated. However, while thread concurrency uses separate OS threads to execute tasks in parallel, async functions share the same thread and use an executor (in our case abstracted simply to the runtime we choose) to decide when each task runs. This makes async programming more efficient for tasks that spend time waiting, like I/O operations, while threads are better for CPU-heavy work.

```rust
use trpl::{Either, Html};
use std::env;

async fn page_title(url: &str) -> (&str, Option<String>) {
    // async postfix keyword "await" is lazy, notice await has no `()`
    let text = trpl::get(url).await.text().await;
    let title = Html::parse(&text)
                    .select_first("title")
                    .map(|title_element| title_element.inner_html());
    (url, title)
}

fn main() {
    // use collect method to get vec from Args iterator
    let args: Vec<String> = env::args().collect();

    trpl::run(async {
        let title_future_1 = page_title(&args[1]);
        let title_future_2 = page_title(&args[2]);
        // race the two async function calls
        let (url, maybe_title) = 
            match trpl::race(title_future_1, title_future_2).await {
                Either::Left(left) => left,
                Either::Right(right) => right,
            };
        
        println!("{url} returned first");
        // try to extract title
        match maybe_title {
            Some(title) => println!("Its page title is: '{title}'"),
            None => println!("Its title could not be parsed."),
        }
    })
    
}
```
This example, from the chapter, runs a race between two web page requests. Inside the call to `trpl::run` we specify our block of code as `async`. The `await` keyword is used when calling an async function or handling a "future" in Rust. When an async function is called (like the one we define here called `page_title`), it does not necessarily complete immediately but can be returned to by the runtime where the call is still pending. This allows other tasks to run in the mean time, making more efficient use of system resources. The `await` keyword can only be used inside an async function because Rust requires an async runtime (`trpl`) to manage execution.

I've tried to keep to a simpler example of async for the write-up here, but there were quite a lot of new concepts and types covered in this large chapter. Many examples were used to cover them, which I've collected together [under the `async-tasks` folder in the project repository](https://github.com/SDeanEgan/rust-lang.book/tree/main). The examples here will also be made available. This has amounted to a substantial block of new concepts for me, so I'm looking forward to the relatively lighter demand next week. Chapters eighteen and nineteen cover Rust's object oriented features and pattern syntax. These will shed light on how trait objects work and unique syntax for special circumstances, which have helped in multiple projects up to now. I'm expecting this to provide a few new tools as well as a different lens through which to view language features, which will be fun. However, additional available time will need to be spent well. I'll be using that to look into further directions for Rust development. Specifically, personal projects that might use object oriented features, advanced patterns, or otherwise expand what I've seen. 