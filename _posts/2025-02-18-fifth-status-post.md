---
tags: [Rust Project]
---
## Fifth Status Post

This week's material covered a wide range of subjects; it also makes for the last week of three chapters of reading!

Chapter seven teaches the basics of package management; that is, how encapsulation and scope control can be used with your projects. While I'm not planning to implement a library with Rust, my future projects will demand using other people's packages. The organization of Rust code can involve a lot of depth, but the access of resources from different modules feels fairly intuitive. It resembles the organizational principles of a system's file path structure, with a larger emphasis placed upon privatization than other languages. 

Returning to concepts that are more direct to the behavior of a given program, chapter eight introduces collections - or at least three key examples. To make good use of this [I included another small project, `collections`, on Github](https://github.com/SDeanEgan/rust-lang.book/tree/main/collections). Its code, `src/main.rs`, is intended as a reference for small examples of several important methods from each of the types: Vectors, Strings, and HashMaps. I expect this to be especially handy if I need to jog my memory about a collection for something like the method's actual name or the type it returns, because the respective pages in the Rust documentation are extremely long. They actually include all the different methods currently available across multiple "release channels" of the language. This includes the official release, Stable version, but also those of the experimental methods currently available under the Nightly release. 

Rust also turns out to offer some useful tools in error handling. Uncertainty with the success of a function's return can be dealt with through enums like `Result` and `Option`, but the process of retrieving a return type they may or may not have is kept simple (and suprisingly short!) via `?` operator and the `panic!` macro. The question mark operator serves to either unwrap a returned type or propogate an error by returning early. This saves a lot of boiler plate code. The panic marcro is focused on circumstances where your program should be stopped. Although *the book* encourages avoiding this wherever an error can be considered recoverable. Critically, Rust does not have exceptions and so requires handling of errors. This keeps things explicit and can help with safety. 

Over the next week I'll cover chapters ten and eleven. These delve into generics, traits, lifetimes, and automated testing. Learning these helps to make code like function signatures or data sturctures either more flexible or more closely related, to increase functionality and reduce duplicate code. 

Now that the project is entering a more mature stage, I need to continue to look ahead. I'm planning on skimming the upcoming project chapter of *the book* to get a sense of what key concepts I can focus on during this week's material. With a clearer sense of where I'm going in mind I think I'll better prevent unexpected confusion and need for review time. 

The topics coming up won't be completely new for me, but are more recent for my development. So, covering the more modern, advanced concepts of Rust should be a good fit for my current skills development needs. 
