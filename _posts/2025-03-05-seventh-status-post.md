---
tags: [Rust Project]
---
## Seventh Status Post

Chapter twelve has readers making [a command line program](https://github.com/SDeanEgan/rust-lang.book/tree/main/minigrep) to search documents and find lines containing matches to a provided query. This is of course the same principle as the classic tool `grep`, and so is called "minigrep", though there isn't included functionality to utilize actual regular expressions. Many earlier concepts make an appearance again: structs, methods, lifetimes, testing, error handling, as well as new functionality like interacting with environment variables and reading a file. Chapter thirteen expands this functionality by introducing how iterators and closures can be used to streamline code and improve performance. 

Besides what features get use in minigrep, what sticks out even more is methodology. There's an ideology at work here that I plan to use as much as possible going forward. `main` isn't used to handle all of the work involved, and we don't simply export repetitively needed code to a function or a data structure. In particular, `main` is used as a staging area which collects the needed resources and performs error handling. Further functionality to perform the program itself is transferred to the `run` function, as a main for main of sorts. Consequently the file for the `main` function just contains imports and the function itself. Because the functionality is split between a `main.rs` and `lib.rs` while using this separation of powers, the working memory needed for which logic is kept where is surprisingly intuitive. 

I also appreciated how the proceeding section taught test writing, specifically using TDD method for developing the program further. The exercise of minigrep works well for this. We can pick what we want our code to be able to do next and make a test for that feature. Then we write a function signature with effectively no body (Rust's compiler is only concerned with signatures anyway) and check that the test fails as we expect. Now our goal is to pass the test before we move on to the next feature. This example does a good job of illustrating the value of TDD, because making a test that checks for the basic result we want and writing a signature serves as an easily digestible portion of work. Also, the effort to pass the test afterward ensures a clear conclusion for an immediate goal. The remainder of minigrep built out well by simply repeating this. 

With a working program and passing tests in hand, we then immediately replace much of the function bodies with a new feature, iterators. Rather than a concrete type however, iterators in Rust are a category of types which implement a trait called `Iterator`. As it turns out, an example of such a type is the collection of command line arguments `Arg` that we worked with to make minigrep all along! As you've probably seen in other languages by now, iterators can be combined with their respective methods and anonymous functions to clear up your code while also keeping it fast. 

Anonymous functions in Rust, closures, are shown to be very concise syntactically, but you may be wondering: if Rust is type-safe, how are closures handled as inputs for structs or functions? To express closures as a type, Rust uses a similar logic as with its references/owned values. Say you define a method `method_name(self, arg)...`, Rust's methodology here is to treat this use of keyword `self` to mean that the object calling that method will be consumed. This allows for safe elimination of resources where transformations may be applied. Much of the time you may not want this though, so instead your method can be defined with a reference, i.e. `&self` or `&mut self`. The reason there are two versions is that Rust distinguishes between references which have only read permissions and those which also include write permissions (with accompanying rules for usage depending on the kind). This is clarified further in the chapter on ownership, chapter four. But what about closures?? They can be identified through this same "three levels of permissions" lens when we want to use them as their own type. 

```rust
impl<T> Option<T> {
    pub fn unwrap_or_else<F: FnOnce() -> T>(self, f: F) -> T
    {
        match self {
            Some(x) => x,
            None => f(),
        }
    }
}
```
The way Rust's "function traits" (`FnOnce`, `Fn`, `FnMut`) work with closures is analogous to how methods use `self`, `&self`, and `&mut self` to control ownership and mutability. Now when we define a struct or enum which takes a closure as input we can specify that one of its parameters uses a generic type, `f: F`, where we also specify what `F` is by its trait bounds: `F: FnOnce() -> T`. 

```rust
fn make_adder(x: i32) -> impl Fn() -> i32 {
    move || x + 1
}
```
Similarly, we can specify the return type for our function as a closure (this one can be used as much as we like) by putting `impl Fn() -> i32` in the signature.

With another project added to the collection, this week will mark a transition to the more advanced Rust concepts and features for the duration of the course. 
In this next week, chapter fourteen teaches package management through Cargo, then back to language features with smart pointers. I'm expecting smart pointers to fill out the functionality of references in my programs, to better match with the capabilities we expect from older languages. So I'll be focusing on chapter fifteen especially. There's some time now before the next project chapter comes along, but I'm still planning on including further project packages in the repository and might just invent another Rust program all my own before my introduction to the language is over.
