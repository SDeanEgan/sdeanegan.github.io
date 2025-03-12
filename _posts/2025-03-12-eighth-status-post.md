---
tags: [Rust Project]
---
## Eighth Status Post

These next chapters of *The Rust programming Language* help to answer the question of what else we can do with Cargo, and how we can accomplish memory tasks which would normally be impossible given the rules we've seen so far. Our informational resources are extended, giving us even more to read at some point. The new types presented require more involved explanation. 

When the book taught minigrep, and encouraged incorporating additional command line arguments, you might have wondered what sort of arguments grep actually offered. The program provides dozens of configuration choices, but we're not expected to memorize them all in order to use grep. That's why when I started reading chapter fourteen I wondered about what a comprehensive look at Cargo configuration might be. Thankfully, I found that [the Cargo book](https://doc.rust-lang.org/cargo/) offers exactly this. The Cargo book serves the dual purpose of introducing/explaining the manager and providing a comprehensive, clearly organized reference. This is just like the Rust book itself but also with material that is closer to more esoteric, official documentation - allowing me to satisfy my curiosity and move forward. 

Initially, I wasn't even sure if I would necessarily include chapter fourteen material. It's about logistical concerns, is that even interesting? Now that I'm done reading I'm actually blown away by the nicety of Cargo. If I write a package seriously, I understand that it's expected to possess "proper documentation", but this always felt like a barrier to contribution. By now you likely know what it's like to read bad docs, and are familiar with visiting a self-hosted version for some package somewhere. What does it really take to make sure your documentation is good and is readily available? Cargo, the resource we've had all along, already has this covered. When you write a `lib.rs` file you likely would include single or multiline comments. However, when planning to publish you can instead pivot to "documentation comments", `//!` and `///`. Fill lines produced with these with markdown formatting (something I already know!) and `cargo doc --open` can generate the document HTML directly. Even better, the code examples you include in your commentary receive automatic testing when you use `cargo test`, which makes sure you know if changes to the package have caused them to break. Yes, Cargo can generate documentation for any dependency or package you download, and you too can publish a package without looking like a scrub.

Things take a more complicated turn where we begin with smart pointers. Rust offers a wide variety of these, including several not in chapter fifteen which  are specifically used for concurrency. This makes the chapter focused on more common use-cases in need of smart pointer flexibility. The new flexibility does not override Rust's established principles, however. Rather than eliminating the restrictions for references which we've established, smart pointers `Rc<T>` and `RefCelll<T>` can be better said to extend functionality while following existing rules. This creates a helpful logical consistency, so long as we appreciate their new contexts. 

The `Rc<T>` smart pointer allows multiple parts of a Rust program to share ownership of the same value. Normally, Rust enforces that each value has only one owner, but `Rc<T>` enables multiple references by actively keeping count of how many owners exist. So now, only when the last owner goes out of scope, is the value automatically cleaned up. Critically, `Rc<T>` only allows immutable access to the value it holds, meaning you cannot modify the value directly. `Rc<T>` is typically used in cases like graph structures or trees where multiple parts of the program need to read the same data without making copies.

Let's make a comparison to ensure things are clear. Borrowing with `&T` allows multiple immutable (standard, read-only) references, but these references are temporary and tied to the lifetime of the original owner. If the owner of the value goes out of scope (somewhere), all borrowed references become invalid. In contrast, `Rc<T>` provides true shared ownership because it ensures the value remains valid as long as at least one `Rc<T>` instance exists. This means that different parts of a program can hold an `Rc<T>` independently without needing to track a specific owner or ensure lifetimes align. For example, in a tree structure where multiple nodes need to reference the same parent, `Rc<T>` allows each node to maintain a reference to shared data without requiring a single owner to manage the entire tree’s lifetime. This kind of flexibility is not possible with standard borrowing, as references must always be tied to a clear, singular owner that dictates their validity.

The `RefCell<T>` smart pointer provides a way to work around Rust's borrowing rules while still enforcing them at runtime. Normally, Rust checks borrowing rules at compile time, ensuring that you either have one mutable reference or multiple immutable references, but never both. `RefCell<T>` allows you to borrow values mutably or immutably within a single owner, but it enforces borrowing rules at runtime instead of compile time. If you violate borrowing rules, your program will panic. This may resemble references from other languages, but is designed to allow "interior mutability". This makes `RefCell<T>` useful in cases where you need to mutate something even when Rust's normal rules would prevent it, such as the example provided with the chapter where references held within our types need to be strung together along a chain. This would prevent being able to mutate the contents of a collection owned further down the line while borrowing still occurs further up. Or perhaps when using `Rc<T>` to allow shared ownership while still needing mutability. The `RefCell<T>` smart pointer is often used in conjunction with `Rc<T>` to provide this:

```rust
use std::cell::RefCell;
use std::rc::Rc;

struct SharedString {
    data: RefCell<String>,
}

fn main() {
    let smart = Rc::new(SharedString {
        data: RefCell::new(String::from("I'm a crab!")),
    });

    let clone = Rc::clone(&smart);

    // Mutate the value inside RefCell even though data is behind an Rc
    smart.data.borrow_mut().push_str(" *pinch*");
    clone.data.borrow_mut().push_str(" *pinch*");

    println!("{}", smart.data.borrow()); // says *pinch* twice
}
```
Keeping things simple, `RefCell<T>` allows mutation of the `count` field even though `counter` is wrapped behind an `Rc<T>`. Without `RefCell<T>`, the borrow checker would prevent any modification because `Rc<T>` only allows immutable access to its contents. [The more elaborate example from the chapter can also be seen in the `src/lib.rs` file available in the `smart_pointers` folder I've included with the Rust project repository on Github.](https://github.com/SDeanEgan/rust-lang.book/tree/main/smart_pointers)

Coming up, we'll finally get to explore how Rust uses concurrency, as well as a newly added chapter of *the book*, Async and Await. These chapters work in conjunction to teach modern performance optimization while maintaining stability in our programs. These are sure to get practical review with our multithreaded web server program, coming up in a few weeks. To get a more thorough sense of how Rust is handling these properties, I'll be contrasting what I learn here with lab work I've previously done from other courses - like the now famous [tiny shell lab](https://www.cs.cmu.edu/afs/cs/academic/class/15213-s02/www/applications/labs/lab5/shlab.html). 
