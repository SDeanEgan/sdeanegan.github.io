---
tags: [Rust Project]
---
## Twelfth Status Post

Sometimes there can be something bitter-sweet about reaching the end of a book. I'm really amazed by everything that's been covered, and though part of me will be missing the experience of freshly uncovering the language I can't help but feel excited at how much more I've found ahead of me. Now that we've reached chapter twenty-one, we end with a capstone project. Concepts from the later chapters return in order to help us complete our multithreaded web server. The major highlights are thread concurrency, passing closures, and implementing the `drop` trait. Rather than focus on the small iterative improvements which make up the chapter, let's explore the end result. We'll start with `main.rs`. 

```rust
use server::ThreadPool;
use std::{
    fs,
    io::{prelude::*, BufReader},
    net::{TcpListener, TcpStream},
    thread,
    time::Duration,
};

fn main() {
    let listener = TcpListener::bind("127.0.0.1:7878").unwrap();
    let pool = ThreadPool::new(4);

    for stream in listener.incoming() {
        let stream = stream.unwrap();

        pool.execute(|| {
            handle_connection(stream);
        });
    }

    println!("Shutting down.");
}

fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&stream);
    let request_line = buf_reader.lines().next().unwrap().unwrap();

    let (status_line, filename) = match &request_line[..] {
        "GET / HTTP/1.1" => ("HTTP/1.1 200 OK", "page.html"),
        "GET /sleep HTTP/1.1" => {
            thread::sleep(Duration::from_secs(5));
            ("HTTP/1.1 200 OK", "page.html")
        }
        _ => ("HTTP/1.1 404 NOT FOUND", "404.html"),
    };

    let contents = fs::read_to_string(filename).unwrap();
    let length = contents.len();

    let response =
        format!("{status_line}\r\nContent-Length: {length}\r\n\r\n{contents}");

    stream.write_all(response.as_bytes()).unwrap();
}
```
What surprised me the most with the project may be how simple the Rust standard library makes setting up our server. A `TcpListener` in Rust is a type provided by the standard library that allows your program to listen for incoming TCP connections on a given IP address and port. You can then call the `incoming` method to get an iterator over the connection attempts, each of which yields a `TcpStream` representing a communication channel with a client. Each stream that comes in gets assigned to a call to our `handle_connection` function, where we use buffer handling methods from the standard library to parse where the client is trying to go before we write back. You'll notice that once `main`'s loop has a request, it hands over the associated call to `handle_connection` to a `ThreadPool`. To see where this takes us we can move to `lib.rs`. 

```rust
use std::{
    sync::{mpsc, Arc, Mutex},
    thread,
};

pub struct ThreadPool {
    workers: Vec<Worker>,
    sender: Option<mpsc::Sender<Job>>,
}

type Job = Box<dyn FnOnce() + Send + 'static>;

impl ThreadPool {
    /// Create a new ThreadPool.
    pub fn new(size: usize) -> ThreadPool {
        assert!(size > 0); // panic if argument isn't a possible value

        let (sender, receiver) = mpsc::channel();
        let receiver = Arc::new(Mutex::new(receiver));
        let mut workers = Vec::with_capacity(size);

        for id in 0..size {
            workers.push(Worker::new(id, Arc::clone(&receiver)));
        }

        ThreadPool {
            workers,
            sender: Some(sender),
        }
    }

    pub fn execute<F>(&self, f: F)
    where
        F: FnOnce() + Send + 'static,
    {
        let job = Box::new(f);

        self.sender.as_ref().unwrap().send(job).unwrap();
    }
}

impl Drop for ThreadPool {
    fn drop(&mut self) {
        drop(self.sender.take());

        for worker in &mut self.workers {
            println!("Shutting down worker {}", worker.id);

            if let Some(thread) = worker.thread.take() {
                thread.join().unwrap();
            }
        }
    }
}

struct Worker {
    id: usize,
    thread: Option<thread::JoinHandle<()>>,
}

impl Worker {
    fn new(id: usize, receiver: Arc<Mutex<mpsc::Receiver<Job>>>) -> Worker {
        let thread = thread::spawn(move || loop {
            let message = receiver.lock().unwrap().recv();

            match message {
                Ok(job) => {
                    println!("Worker {id} got a job; executing.");

                    job();
                }
                Err(_) => {
                    println!("Worker {id} disconnected; shutting down.");
                    break;
                }
            }
        });

        Worker {
            id,
            thread: Some(thread),
        }
    }
}

```
Beyond our server code you can see, from `lib.rs` above, that the rest of the project implements logic for pooling threads. But why would we need to do this? Using a single thread to handle all incoming requests means that the server can only process one connection at a time, which becomes a serious bottleneck under load. If one request takes a long time - such as waiting for a loading file - every other client must wait until that task is finished before their request is even acknowledged. This leads to poor responsiveness and timeout errors. A thread pool solves this by keeping multiple "worker" threads alive and ready to handle incoming "jobs" concurrently, allowing the server to continue processing new requests even while other threads are busy. 

This means our `ThreadPool` is really just a channel to facilitate communication across threads and a collection of `Worker`s which wrap around the threads. As we saw in chapter sixteen, `mpsc` channels typically can only have a single receiver per channel, but we can use the combination of smart pointers `Arc` and `Mutex` we learned in chapter fifteen to get around that. Channels normally have a single receiver for a reason, though. Each `Worker` spawns its own thread and loops until a job is received. What prevents active threads from receiving each other's jobs? Consider the line `let message = receiver.lock().unwrap().recv();` inside the loop for the thread of a new `Worker`. Recall that the requirements of the `Mutex` smart pointer are such that the thread must `lock` before accessing the pointee. Other threads cannot access the channel while this is the case. This combines with the behavior of `recv` method for the `mpsc::Receiver`; it blocks until a job is sent across the channel. If there is no job yet, the current thread will wait until a job becomes available. 

In another return to chapter fifteen, we implement the `Drop` trait for our thread pool to ensure that all worker threads are properly shut down when the thread pool goes out of scope. In this implementation, the first step is to call `self.sender.take()` and drop it, which closes the sending half of the channel. This causes all calls to `recv` on the worker threads' receiving ends to return an error, signaling that no more jobs will arrive. Each worker, upon encountering an error, prints a shutdown message and exits its loop. After dropping the sender, the main thread iterates through the workers and calls `join` on each one's thread handle, waiting for them to finish executing. This guarantees that all threads have completed their work and exit cleanly before the program continues. It's important that we do this to avoid leaving background threads running or terminating the program with unfinished tasks.

The project this week featured a few last, subtle qualities we should touch on. Like wrapping sender from the `mpsc::channel` and the `Worker` threads in `Option`s. This is what opens up the ability to implement the `Drop` trait the way we do, because wrapping in an `Option` opens up the `take` method, which is necessary bring ownership of the internal item into the trait method. Other examples include the type alias `Job`, that cuts down on boilerplate in our library's functions, and I believe this may be the first time we've been shown string literals used directly for the patterns of a match expression. 

We've finally concluded *The Rust Programming Language*! I couldn't have asked for a better introduction to the language. Every example has felt useful, and I have especially appreciated the clear, confident voice throughout. Best of all, I'm grateful that *the book* is a free resource, available to anyone. The question then remains, where do we go from here? In a couple weeks I plan to submit a final report for this Rust introduction project, where I'll be covering its outcomes. However, in the mean time I'm planning to have one more post where I present and explore additional Rust resources I've encountered in the past three months. Including a preliminary dive into a popular follow-up, [*Effective Rust*](https://effective-rust.com/title-page.html).