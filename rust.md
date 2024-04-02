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
