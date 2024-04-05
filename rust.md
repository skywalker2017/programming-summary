# Pragma

## Array and slice
the length of an array is known at compile time, while sice is a two word object, the first object is the pointer to the slice, and the second word is the length of the slice
+ type signature of array is [T; length]
+ Out of bound indexing on array causes compile time error.
+ Out of bound indexing on slice causes runtime error.
+ borrow a section of the array as the slice:
  + xs[..]: borrow the whole array as the slice
  + xs[start..end] borrow [start, end)

## Custom types
### struct
  + you can replace the struct with struct update syntax:
    ```
    let point: Point = Point { x: 10.3, y: 0.4 };
    let bottom_right = Point { x: 5.2, ..point };    
    ```
    ```
    point coordinates: (10.3, 0.4)
    second point: (5.2, 0.4)
    ```
### constants
  + `const`: An unchangeable value (the common case).
  + `static`: A possibly mutable variable with 'static lifetime. The static lifetime is inferred and does not have to be specified. Accessing or modifying a mutable static variable is unsafe.

## Conversion

### Convert any type to String
To convert any type to a String is as simple as implementing the ToString trait for the type. Rather than doing so directly, you should implement the fmt::Display trait which automagically provides ToString and also allows printing the type as discussed in the section on print!.
```Rust
use std::fmt;

struct Circle {
    radius: i32
}

impl fmt::Display for Circle {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        write!(f, "Circle of radius {}", self.radius)
    }
}

fn main() {
    let circle = Circle { radius: 6 };
    println!("{}", circle.to_string());
}
```

## Flow Control

### loop
there is a tricky usage of look that you can return a value in a loop and pass it to rest of the code, it can be used to retry an operation until it succeed
```Rust
fn main() {
    let mut counter = 0;

    let result = loop {
        counter += 1;

        if counter == 10 {
            break counter * 2;
        }
    };

    assert_eq!(result, 20);
}
```
### for
+ `iter`: This borrows each element of the collection through each iteration. Thus leaving the collection untouched and available for reuse after the loop.
```Rust
fn main() {
    let names = vec!["Bob", "Frank", "Ferris"];

    for name in names.iter() {
        match name {
            &"Ferris" => println!("There is a rustacean among us!"),
            // TODO ^ Try deleting the & and matching just "Ferris"
            _ => println!("Hello {}", name),
        }
    }
    
    println!("names: {:?}", names);
}
```
+ `into_iter`: This consumes the collection so that on each iteration the exact data is provided. Once the collection has been consumed it is no longer available for reuse as it has been 'moved' within the loop.
```Rust
fn main() {
    let names = vec!["Bob", "Frank", "Ferris"];

    for name in names.into_iter() {
        match name {
            "Ferris" => println!("There is a rustacean among us!"),
            _ => println!("Hello {}", name),
        }
    }
    
    println!("names: {:?}", names);
    // FIXME ^ Comment out this line
}
```
got error:
```
2   |     let names = vec!["Bob", "Frank", "Ferris"];
    |         ----- move occurs because `names` has type `Vec<&str>`, which does not implement the `Copy` trait
3   |
4   |     for name in names.into_iter() {
    |                       ----------- `names` moved due to this method call
...
11  |     println!("names: {:?}", names);
    |                             ^^^^^ value borrowed here after move
```

+ `iter_mut`: This mutably borrows each element of the collection, allowing for the collection to be modified in place.
```
fn main() {
    let mut names = vec!["Bob", "Frank", "Ferris"];

    for name in names.iter_mut() {
        *name = match name {
            &mut "Ferris" => "There is a rustacean among us!",
            _ => "Hello",
        }
    }

    println!("names: {:?}", names);
}
```
result:
```
names: ["Hello", "Hello", "There is a rustacean among us!"]
```

### match
#### binding

Indirectly accessing a variable makes it impossible to branch and use that variable without re-binding. match provides the @ sigil for binding values to names:

```Rust
// A function `age` which returns a `u32`.
fn age() -> u32 {
    15
}

fn main() {
    println!("Tell me what type of person you are");

    match age() {
        0             => println!("I haven't celebrated my first birthday yet"),
        // Could `match` 1 ..= 12 directly but then what age
        // would the child be? Instead, bind to `n` for the
        // sequence of 1 ..= 12. Now the age can be reported.
        n @ 1  ..= 12 => println!("I'm a child of age {:?}", n),
        n @ 13 ..= 19 => println!("I'm a teen of age {:?}", n),
        // Nothing bound. Return the result.
        n             => println!("I'm an old person of age {:?}", n),
    }
}
```

## Functions
+ there is no restriction on the order of function definitions
+ Functions that "don't" return a value, actually return the unit type `()`
```Rust
fn fizzbuzz(n: u32) -> () {
    if is_divisible_by(n, 15) {
        println!("fizzbuzz");
    } else if is_divisible_by(n, 3) {
        println!("fizz");
    } else if is_divisible_by(n, 5) {
        println!("buzz");
    } else {
        println!("{}", n);
    }
}
```

### Closure
+ A closure taking no arguments which returns an `i32`.
```Rust
let one = || 1;
println!("closure returning one: {}", one());
```
+ a borrowed value cannot be reborrowed again until 
```Rust
let mut count = 0;
// A closure to increment `count` could take either `&mut count` or `count`
// but `&mut count` is less restrictive so it takes that. Immediately
// borrows `count`.
//
// A `mut` is required on `inc` because a `&mut` is stored inside. Thus,
// calling the closure mutates `count` which requires a `mut`.
let mut inc = || {
  count += 1;
  println!("`count`: {}", count);
};

// Call the closure using a mutable borrow.
inc();

// The closure still mutably borrows `count` because it is called later.
// An attempt to reborrow will lead to an error.
// let _reborrow = &count; 
// ^ TODO: try uncommenting this line.
inc();

// The closure no longer needs to borrow `&mut count`. Therefore, it is
// possible to reborrow without an error
let _count_reborrowed = &mut count; 
```

# Errors

## Move
```
Compiling playground v0.0.1 (/playground)
error[E0382]: use of moved value: `point`
  --> src/main.rs:43:16
   |
38 | fn square(point: Point, len: f32) -> Rectangle {
   |           ----- move occurs because `point` has type `Point`, which does not implement the `Copy` trait
39 |     Rectangle{
40 |         top_left: point,
   |                   ----- value moved here
...
43 |             y: point.y + len,
   |                ^^^^^^^ value used here after move

For more information about this error, try `rustc --explain E0382`.
error: could not compile `playground` (bin "playground") due to 1 previous error
```
The error message you received in Rust indicates that a move operation is happening with the variable  point , which has type  Point , and this type does not implement the  Copy  trait. In Rust, types that do not implement the  Copy  trait will be moved when assigned to another variable or passed as a function argument, meaning the ownership is transferred to the new location, which can lead to errors if the original variable is accessed afterward.

## Mutable Static
```Rust
static mut LANGUAGE: &str = "Rust";
LANGUAGE = "sss";
```
```
error[E0133]: use of mutable static is unsafe and requires unsafe function or block
  --> src/main.rs:19:5
   |
19 |     LANGUAGE = "sss";
   |     ^^^^^^^^ use of mutable static
   |
   = note: mutable statics can be mutated by multiple threads: aliasing violations or data races will cause undefined behavior
```
to solve the problem, you can wrap the modifying code with unsafe expression
```Rust
static mut LANGUAGE: &str = "Rust";
unsafe{
LANGUAGE = "sss";
}
```
