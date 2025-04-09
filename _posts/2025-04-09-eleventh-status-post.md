---
tags: [Rust Project]
---
## Eleventh Status Post

This week I'll be focusing my attention on chapter twenty, *Advanced Features*. So many features get presented, I'm going to have to use large code examples in order to cover as many as I can. As a quick primer, we learn the different purposes of `unsafe` Rust. These include using functions or methods which must be tagged as `unsafe`, but actions which qualify as unsafe include: mutating Rust's static variables, dereferencing new pointer types called raw pointers, or utilizing "external" code sourced from some other language. More critically, we can use unsafe actions within safe (normal) code and abstract away to leave an interface which is safe to use. Further sections include less commonplace features for traits, types, functions, and digging into macros - which help to fill out the language in ways you might have never imagined before. 

```rust
use std::fmt::Debug;
use std::ops::Add;

static mut PROCESS_CALL_COUNT: usize = 0;

trait Processor {
    type Input: Debug;
    type Output: Debug;
    fn process(&self, input: Self::Input) -> Self::Output;
}

struct Plugin<I, O> {
    func: Box<dyn Fn(I) -> O>,
}

impl<I: Debug, O: Debug> Processor for Plugin<I, O> {
    type Input = I;
    type Output = O;
    fn process(&self, input: I) -> O {
        // encapsulate to safely update the global process call count
        unsafe {
            PROCESS_CALL_COUNT += 1;
        }
        (self.func)(input)
    }
}

// implement Add for chaining Plugins: (I -> U) + (U -> O) => (I -> O)
impl<I, U, O> Add<Plugin<U, O>> for Plugin<I, U>
where
    I: Debug + 'static,
    U: Debug + 'static,
    O: Debug + 'static,
{
    type Output = Plugin<I, O>;
    fn add(self, rhs: Plugin<U, O>) -> Self::Output {
        let f = self.func;
        let g = rhs.func;
        
        Plugin {
            func: Box::new(move |x| g(f(x)))
        }
    }
}

// helper function to create a Plugin from a regular function or closure
fn plugin<I, O>(f: impl Fn(I) -> O + 'static) -> Plugin<I, O>
where
    I: Debug + 'static,
    O: Debug + 'static,
{
    Plugin {
        func: Box::new(f)
    }
}

fn uppercase(input: String) -> String {
    input.to_uppercase()
}

fn add_exclamations(input: String) -> String {
    format!("{}!!!", input)
}

fn print_process_call_count() {
    #[allow(static_mut_refs)]
    unsafe {
        println!("\n[PROCESS CALL COUNT]: {}", PROCESS_CALL_COUNT);
    }
}

fn main() {
    let plugin1 = plugin(uppercase);
    let plugin2 = plugin(add_exclamations);
    // apply plugin to strings
    let result1 = plugin1.process("hello".to_string());
    let result2 = plugin2.process("world".to_string());
    println!("Plugin1: {}", result1);
    println!("Plugin2: {}", result2)
    // use addition to synthesize a new function
    let combined_plugin = plugin1 + plugin2;
    let result3 = combined_plugin.process("combined".to_string());
    println!("Combined: {}", result3); 
    
    print_process_call_count();
}

```

This code uses `Plugin` struct which maintains a function we provide at declaration. This showcases several advanced features from the chapter; let's start with `static mut PROCESS_CALL_COUNT: usize = 0;`.  Some actions with `unsafe` Rust only require the actual `unsafe` block when executing on a variable or object in an unsafe way. Here, we are allowed to instantiate `PROCESS_CALL_COUNT` in safe code so long as we follow the requirements. These are similar to `const`, learned about originally in chapter three. Static variables must have their type annotated, are only ever stored in the same memory location, and acting on them is only unsafe when they are declared mutable like we see here. While we've covered trait definitions like `Processor` before, we can now use them to review Associated Types. 

Notice, both `Processor` and `Add` feature code like: `type Output = Plugin<I, O>`. What is this for? In order for Rust's type checker to approve our implementation of these traits' methods for `Plugin` we need to make sure the definition corresponds properly. For our custom trait, `Processor`, this may not be totally necessary when we consider generics, but `Add` should illustrate that we get better flexibility this way. What if we defined some struct which needs to have several other kind of types to be added to it? There is a critical restriction in Rust that we can't implement a trait on a type multiple times, but Associated Types helps us to get around this. 

The key appeal to our `Plugin` code is how it makes use of advanced functions. Implementing the `Add` trait gives us operator overloading, another new feature. We can use the `+` operation to synthesize functions together to make new ones. While the chapter specifically introduces function pointers `fn` and returning closures, this functionality also comes with some important restrictions. Rust actually treats functions and closures as distinct types. In order for a `Plugin` to work with a wider variety of inputs, we use trait objects `Box<dyn Fn(I) -> O>` as is encouraged by the chapter. This also helps get our function synthesis to work with `add`, because a critical safety rule of returning closures demands they haven't captured any values from their environment. 

You may be wondering: since Rust has this rule about not returning closures which capture from their environment, how `Plugin { func: Box::new(move |x| g(f(x))) }` avoids breaking the rule. The true restriction is that you can't return a closure directly from a function when that closure captures values from the function's environment, unless you properly handle the lifetimes or ownership. When you put a closure in a `Box`, you're creating a heap-allocated object with a stable memory address. This turns the closure (which has an unknown size at compile time) into a pointer (which has a known size). The `Box` takes ownership of the closure and manages its memory. By using `move |x| g(f(x))`, you're explicitly telling Rust to transfer ownership of the captured variables (`f` and `g`) into the closure. This means the closure now owns `f` and `g` and doesn't depend on any external references that might become invalid. Notice the `'static` bounds on the type parameters. This ensures that any types used in these functions will live for the entire program duration, preventing issues with references becoming dangling. In essence, you're creating a self-contained package that owns everything it needs to function. Since it owns all its captured variables and is stored on the heap via a `Box`, it can safely be returned from the function. 

The next example isn't for the faint of heart, but it does illustrate even more advanced features. 
```rust
use std::fmt;

pub struct StackDump<T: ?Sized>(pub T);

impl<T: ?Sized> StackDump<T> {
    pub fn dump(&self, num_bytes: usize) {
        let ptr = &self.0 as *const T as *const u8;
        println!("Stack dump from {:p} ({} bytes):", ptr, num_bytes);

        unsafe {
            for i in (0..num_bytes).rev() {
                let byte_ptr = ptr.add(i);
                let byte = *byte_ptr;

                if (num_bytes - i - 1) % 16 == 0 {
                    print!("\n{:p}: ", byte_ptr);
                }
                print!("{:02X} ", byte);
            }
        }
        println!();
    }
}

impl<T> fmt::Display for StackDump<T> {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        let ptr = &self.0 as *const T as *const u8;
        let size = std::mem::size_of::<T>();

        write!(f, "0x")?;
        unsafe {
            for i in (0..size).rev() {
                write!(f, "{:02X}", *ptr.add(i))?;
            }
        }

        Ok(())
    }
}

fn main() {
    let _w = 0xdeadc0deu64;
    let x = 0xdeadbeefu64;
    let on_stack = StackDump(x);

    println!("As hex: {}", on_stack);
    on_stack.dump(128);
}

/*
As hex: 0x00000000DEADBEEF
Stack dump from 0x7fd44f3f88 (128 bytes):

0x7fd44f4007: 00 00 00 55 7D 5F 67 A0 00 00 00 7F D4 4F 3F 88 
0x7fd44f3ff7: 00 00 00 00 DE AD BE EF 00 00 00 00 DE AD C0 DE 
0x7fd44f3fe7: 00 00 00 55 AA D8 24 80 00 00 00 55 7D 5F 67 A0 
0x7fd44f3fd7: 00 00 00 7F D4 4F 3F 88 00 00 00 55 7D 5F 67 A0 
0x7fd44f3fc7: 00 00 00 7F D4 4F 3F 88 00 00 00 00 00 00 00 00 
0x7fd44f3fb7: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 01 
0x7fd44f3fa7: 00 00 00 7F D4 4F 3F C0 00 00 00 00 00 00 00 02 
0x7fd44f3f97: 00 00 00 55 7D 64 E4 58 00 00 00 00 DE AD BE EF 
*/
```
The `StackDump` implementation demonstrates `unsafe` Rust through explicit `unsafe` blocks. Within these blocks the code performs operations that the Rust compiler cannot verify as memory-safe, such as dereferencing raw pointers `*byte_ptr`, and performing pointer arithmetic with `ptr.add(i)`. Similar to where we used mutable static variables before, creating a raw pointer is not itself unsafe. So  `let ptr = &self.0 as *const T as *const u8` occurs in safe code. This shows how Rust allows developers to take manual control of memory when needed, while clearly marking code that bypasses the safety guarantees. We're using raw pointers (`*const T` and `*const u8`) to directly access memory addresses. Converting from references to raw pointers with expressions like these demonstrates how Rust allows dropping down to a lower level when necessary for tasks like memory inspection. 

The struct definition `StackDump<T: ?Sized>` showcases dynamically sized types through the `?Sized` trait bound. This relaxes Rust's default requirement that all generic type parameters must have a known size at compile time. Without this special bound, `StackDump` couldn't wrap types whose size is only known at runtime. The code also demonstrates how to use advanced trait implementations with the `fmt::Display` trait, allowing the `StackDump` type to be formatted using the `{}` syntax in macros like `println!`. This means we can observe the hex representation of the object we passed to identify it in the stack portion we printed to standard out. Additional advanced features include type casting between various pointer types, and abstracting unsafe code within safe code to construct a safer interface.

If you thought after those two examples we'd be done, you're `0xdead` wrong! If there's a subject from this week which deserves its own chapter in *the book*, it's definitely macros. You can think of them as the final form of the match expression; they can generate anything, even more code. So when you write macros you're coding code to code your code.

```rust
macro_rules! debug_log {
    // matches a single log level and message
    ($level:expr, $message:expr) => {
        println!(
            "[{}] ({}) {}: {}",
            $level,
            module_path!(),
            line!(),
            $message
        )
    };
    
    // matches a log level, a format string, and variable arguments
    ($level:expr, $format:expr, $($arg:expr),*) => {
        println!(
            "[{}] ({}) {}: {}",
            $level,
            module_path!(), // a built-in macro for the module running this macro
            line!(), // another built-in for the line from which it is invoked
            format!($format, $($arg),*)
        )
    };
}

fn main() {
    debug_log!("INFO", "Application start");
    
    let x = 42;
    let name = "Guy";
    debug_log!("DEBUG", "Value: {}, Name: {}", x, name);
    debug_log!("ERROR", 
               "Failed #{} - {}", 
               8675309, 
               "Invalid input");
}

/*
[INFO] (macros) 26: Application start
[DEBUG] (macros) 30: Value: 42, Name: Guy
[ERROR] (macros) 31: Failed #8675309 - Invalid input
*/
```
Macros are best where they can handle a variety of cases or include lots of repetition. They're intended to serve a number of key purposes, where the most common appears to be providing a function alternative which accepts variable amounts of arguments or providing quick annotation like expressions of the form `#[...]`. The later is called an "attribute", which often employs a macro to generate code for you (like implementing a trait for your custom type automatically) or interact with the compiler as we saw with an attribute used in the first example. 

Because they can be used to accomplish so much, the above example really only scratches the surface of what's possible but let's explore what we have. Similar to a match expression, `debug_log!` handles two cases. The first is where we feed it a couple of expressions. Doing so leads to a match with `($level:expr, $message:expr)`. Notice that we have what looks like a type annotation, but because we're working in a macro this is instead a "fragment". Fragments can be any of the constructs of the language we've learned about up to now, like `block` for block expressions, `lifetime` for lifetimes, `path` for paths to a type, `ty` for types, and so forth. In our case we only need to feed expressions to the `println!` macro, which itself supports variable numbers of expressions. To fill out the case where we ourselves have variable numbers of expressions we include a pattern with the `$($arg:expr),*` syntax. Where this case is used the call to `println!` has all the additional arguments fed to it by the expansion `format!($format, $($arg),*)` which uses `$format` format string formatted with the `$($arg),*)` arguments using the macro, `format!`. 

We've seen so many new features in chapter twenty! It's for the best that we're only encouraged to use it as a reference for what are likely to only occur rarely, but we've also seen how whether we need to perform work - at a higher or lower level - it will likely involve some advanced Rust. In the next week I'll be finally heading into the end chapter of *The Rust Programming Language*. Now that we're reaching the end web server project we get to see examples of how Rust can interact with TCP, but with everything we learned in this chapter I'm already wondering how we might expand it. *The book* itself suggests exploring external crates devoted to the task, so we have many options available to us. 