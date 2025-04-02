---
tags: [Rust Project]
---
## Tenth Status Post

After a short break, the time has come to get back to *the book*. Now, we do some examining how Rust does and doesn't qualify as object oriented, formally learn trait objects, and run through various syntaxes of patterns. 

Up to now the text has expressed an outlook that Rust is not object oriented, which receives deeper clarification in our next chapter. Briefly, we already know that the languaage offers constructions which combine data and associated operations. These are stucts an enums. But according to some interpretations a system of inheritance is necessary in order for a language to qualify as object oriented. As we have seen, Rust does not offer inheritance. Instead, a combination of traits and generics may be used to affect similar outcomes. We have seen traits and generics used several times before now, but what should be fairly new to us are trait objects. 

In the further sections of chapter eighteen, trait objects are explored. While they haven't been explained until now, we've actually used them a couple times. Such as returning a variety of error types from a function in chapter twelve, or in chapter seventeen where we build a collection of variably typed futures to get our async code to work properly. Without them there wouldn't normally be this kind of flexibility, as the type checker should require a specific concrete type by compile time. How can we get around this with trait objects? Let's work with an example similar to the situation from chapter seventeen and use a variety of types at the same time within a collection. 

```rust
use std::fmt;
use std::io;

struct ErrorLog {
    log: Vec<Box<dyn std::error::Error>>, // Stores trait objects
}

impl ErrorLog {
    fn new() -> Self {
        Self { log: Vec::new() }
    }
    fn add<E: std::error::Error + 'static>(&mut self, error: E) {
        self.log.push(Box::new(error));
    }
    fn give(&self) {
        for (i, error) in self.log.iter().enumerate() {
            println!("Error {}: {}", i + 1, error);
        }
    }
}

fn main() {
    let mut errors = ErrorLog::new();

    // Simulate an I/O error and a formatting error
    let io_error = io::Error::new(io::ErrorKind::NotFound, "File not found");
    errors.add(io_error);
    let fmt_error = fmt::Error;
    errors.add(fmt_error);

    // Print logged errors
    errors.give();
}
```

Two examples of errors in Rust come from `std::io` and `std::fmt`, which stem from different contexts and have different space requirements. However, our program may need to make active use of either kind. Say the `log` vector of our `ErrorLog` struct only used a generic with a trait bound. Then we would still need type homogeneity in the collection, but because a trait object can be dynamically sized we can have variety within the collection simultaneously so long as they all implement the required trait. In our case, both error kinds implement the trait `std::error::Error`. Notice, we still use a trait bound (along with lifetime `'static`) in the signature for `add`. This method works for either error because the compiler will create a separate specialized version of `add` for each concrete error type that gets used in the program. This is likely preferable because it should be faster than the alternative to also use a `Box<dyn Error>` trait object with `add`, as this introduces some runtime overhead. 

A trait object points to both an instance of a type implementing our specified trait and a table used to look up trait methods on that type at runtime. In a way, trait objects provide Rust with some of the flexibility seen in dynamically typed languages. In dynamically typed languages like Python or JavaScript, you can store different types in the same collection without explicit type constraints. Rust, being statically typed, enforces type safety at compile time, so you usually need a single concrete type for collections. Trait objects then offer polymorphism, similar to what dynamic languages provide, but with Rust's ownership rules and borrowing system still in place. A trade-off is that method calls on trait objects use dynamic dispatch instead of being inlined at compile time like generic functions, which can introduce some runtime cost. So while it's not the same as full dynamic typing, it gives Rust a way to achieve similar flexibility within safe boundaries.

The following chapter covers special syntax cases, patterns. It's actually one of the shorter chapters of *the book* and feels more eclectic to read, but is also some of the most important information we've seen this far for good reading comprehension of Rust code in the real world. Part of me wonders why the *Patterns and Matching* chapter doesn't appear sooner in the text, but this is likely because they apply virtually everywhere and might need to be retaught as more advanced concepts came along. Still though, part of me is surprised we got this far without discussing guard patterns. 

```rust
    let num = Some(4);

    match num {
        Some(x) if x % 2 == 0 => println!("The number {x} is even"),
        Some(x) => println!("The number {x} is odd"),
        None => (),
    }
```

In the first arm of our match expression we include an `if` followed by a boolean condition. This pattern seems misleading simple, but the added specificity can be huge. It's the ability to use boolean conditions, and patterns in conjunction that make the inclusion of patterns with the language so valuable. 

```rust
fn main() {
    let favorite_color: Option<&str> = None;
    let is_tuesday = false;
    let age: Result<u8, _> = "34".parse();

    if let Some(color) = favorite_color {
        println!("Using your favorite color, {color}, as the background");
    } else if is_tuesday {
        println!("Tuesday is green day!");
    } else if let Ok(age) = age {
        if age > 30 {
            println!("Using purple as the background color");
        } else {
            println!("Using orange as the background color");
        }
    } else {
        println!("Using blue as the background color");
    }
}
```

This rich example from the text illustrates how the control flow of conditional branching we know can be expanded to circumstances where a boolean evaluation may not make sense or to cut down on boilerplate assignments through the inherent destructuring accomplished through patterns. Here, `color` and `age` are unpacked and used in the branch's scope, but we can still have the boolean branch `else if is_tuesday` which is more traditional. The downside here is that our combination of patterns and boolean evaluation requires us to consider the additional cases which may be opened as a result in order to maintain comprehension. This prevents the exhaustiveness checks which the compiler would otherwise provide. 

There are a wealth of other examples from the chapter, like destructuring directly into parameters of a function signature or `@` bindings to cover where a variable should possess a value within some range. As I move on with the language I look forward to discovering more of these. Looking ahead, all that is left is two chapters; these are *Advanced Features* and the final project. I'm currently hoping that no more than two weeks are necessary for these. However, both deserve significant attention. For example, chapter twenty introduces unsafe Rust, a critical feature for incorporating Rust code into existing operating systems - and obviously much more. Chapter twenty-one will complete *The Rust Programming Language* with a web server program that employs concurrency features. These seem like a perfect ending for *the book*. Although, there remains some questions for me as to what moving on with learning Rust should mean. The time remaining to decide on the follow-up is drawing to a close, but I'm more than satisfied with what I've been able to learn so far. 