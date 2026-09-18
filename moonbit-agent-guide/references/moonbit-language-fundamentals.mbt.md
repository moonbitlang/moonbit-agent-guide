# MoonBit Language Fundamentals


## Quick reference:

```mbt check
///|
/// comments doc string
pub fn sum(x : Int, y : Int) -> Int {
  x + y
}

///|
struct Rect {
  width : Int
  height : Int
}

///|
fn Rect::area(self : Rect) -> Int {
  self.width * self.height
}

///|
pub impl Show for Rect with fn output(_self, logger) {
  logger.write_string("Rect")
}

///|
enum MyOption {
  MyNone
  MySome(Int)
} derive(Debug, ToJson, Eq, Compare)

///|
///  match + loops are expressions
test "everything is expression in MoonBit" {
  // tuple
  let (n, opt) = (1, MySome(2))
  // if expressions return values
  let msg : String = if n > 0 { "pos" } else { "non-pos" }
  let res = match opt {
    MySome(x) => {
      inspect(x, content="2")
      1
    }
    MyNone => 0
  }
  let status = MySome(10)
  // match expressions return values
  let description = match status {
    MySome(value) => "Success: \{value}"
    MyNone => "No value"
  }
  inspect(msg, content="pos")
  inspect(res, content="1")
  inspect(description, content="Success: 10")
  assert_true(MyNone is MyNone)
  let array = [1, 2, 3, 4, 5]
  let mut i = 0 // mutable bindings (local only, globals are immutable)
  let target = 3
  // loops return values with 'break'
  let found : Int? = while i < array.length() {
    if array[i] == target {
      break Some(i) // Exit with value
    }
    i = i + 1
  } nobreak {
    None
  } // Value when loop completes normally
  assert_eq(found, Some(2)) // Found at index 2
}

///|
/// global bindings
pub let my_name : String = "MoonBit"

///|
pub const PI : Double = 3.14159 // constants use UPPER_SNAKE or PascalCase

///|
pub fn maximum(xs : Array[Int]) -> Int raise {
  // Toplevel functions are *mutually recursive* by default
  // `raise` annotation means the function would raise any Error
  //  Only add `raise XXError` when you do need track the specific error type
  match xs {
    [] => fail("Empty array") // fail() is built-in for generic errors
    [x] => x
    // pattern match over array, the `.. rest` is a rest pattern
    // it is of type `ArrayView[Int]` which is a slice
    [x, .. rest] => {
      let mut max_val = x // `mut` only allowed in local bindings
      for y in rest {
        if y > max_val {
          max_val = y
        }
      }
      max_val // return can be omitted if the last expression is the return value
    }
  }
}

///|
/// pub(all) means it can be both read and created outside the package
pub(all) struct Point {
  x : Int
  mut y : Int
} derive(Debug, ToJson)

///|
pub enum LoadState {
  Loading // semicolon `;` is optional when we have a newline
  Ready(Int) // Enum variants must start uppercase
} derive(Debug, Eq, ToJson)
// pub means it can only be pattern matched outside the package
// but it can not be created outside the package, use `pub(all)` otherwise

///|
/// pub (open) means the trait can be implemented for outside packages
pub(open) trait Comparable {
  fn compare(Self, Self) -> Int // `Self` refers to the implementing type
}

///|
test "inspect test" {
  let rect : Rect = { width: 3, height: 4, }
  inspect(rect.area(), content="12")
  debug_inspect(Loading, content="Loading")
  debug_inspect(Ready(1), content="Ready(1)")
  let result = sum(1, 2)
  inspect(result, content="3")
  // The `content` can be auto-corrected by running `moon test --update`
  let point = Point::{ x: 10, y: 20, }
  // For complex structures, use json_inspect for better readability:
  json_inspect(point, content={ "x": 10, "y": 20 })
}
```


## Complex Types

```mbt check
///|
pub type UserId = Int // Int is aliased to UserId - like symlink

///|
///  Tuple-struct for callback
pub struct Handler((String) -> Unit) // A newtype wrapper

///|
/// Tuple-struct syntax for single-field newtypes
struct Meters(Int) // Tuple-struct syntax

///|
let distance : Meters = Meters(100)

///|
let raw : Int = distance.0 // Access first field with .0

///|
struct Addr {
  host : String
  port : Int
} derive(Debug, Eq, ToJson, FromJson)

///|
/// Structural types with literal syntax
let config : Addr = Addr::{
  // `Type::` can be omitted since the type is already known
  host: "localhost",
  port: 8080,
}

///|
test "newtypes and records" {
  inspect(raw, content="100")
  inspect(config.host, content="localhost")
  let received : Ref[String] = Ref::{ val: "", }
  let handler = Handler(message => received.val = message)
  (handler.0)("hello")
  inspect(received.val, content="hello")
}
```

## Common Derivable Traits

Most types can automatically derive standard traits using the `derive(...)` syntax:

- **`Debug`** - Enables `debug_inspect()` for structural test/diagnostic output; the derivable default for your own data types. For interpolation of composed values use `\{Repr(value)}`
- **`Show`** - Produces specialized display strings (JSON, XML, user-facing text). Deriving it for debugging is deprecated in favor of `Debug`; write a manual `impl Show for T with fn output(self, logger) { ... }` only for genuine display formats
- **`Eq`** - Enables `==` and `!=` equality operators
- **`Compare`** - Enables `<`, `>`, `<=`, `>=` comparison operators
- **`ToJson`** - Enables `json_inspect()` for readable test output
- **`Hash`** - Enables use as Map keys

```mbt check
///|
struct Coordinate {
  x : Int
  y : Int
} derive(Debug, Eq, ToJson)

///|
enum Status {
  Active
  Inactive
} derive(Debug, Eq, Compare)

///|
test "derived traits" {
  let point : Coordinate = { x: 1, y: 2, }
  json_inspect(point, content={ "x": 1, "y": 2 })
  assert_true(Active < Inactive)
}
```

**Best practice**: Derive `Debug` and `Eq` for data types (use `debug_inspect()` in tests; `\{Repr(value)}` for interpolation). Add `ToJson` if you plan to test them with `json_inspect()`. Implement `Show` by hand only for specialized display formats (JSON, XML, user-facing text).

## Reference Semantics by Default

MoonBit passes most types by reference semantically (the optimizer may copy
immutables):

```mbt check
///|
///  Structs with 'mut' fields are always passed by reference
struct Counter {
  mut value : Int
}

///|
fn increment(c : Counter) -> Unit {
  c.value += 1 // Modifies the original
}

///|
/// Arrays and Maps are mutable references
fn modify_array(arr : Array[Int]) -> Unit {
  arr[0] = 999 // Modifies original array
}

///|
test "reference semantics" {
  let c : Counter = { value: 0, }
  increment(c)
  inspect(c.value, content="1")
  let counter : Ref[Int] = Ref::{ val: 0, }
  counter.val += 1
  assert_true(counter.val is 1)
  let arr : Array[Int] = [1, 2, 3] // unlike Rust, no `mut` keyword needed
  modify_array(arr)
  assert_true(arr[0] is 999)
  let mut x = 3 // `mut` neeed for re-assignment to the bindings
  x += 2
  assert_true(x is 5)
}
```

## Pattern Matching

```mbt check
///|
test "pattern match over Array, struct and StringView" {
  let arr : Array[Int] = [10, 20, 25, 30]
  match arr {
    [] => fail("expected a nonempty array")
    [single] => inspect(single, content="10")
    [first, .. middle, rest] => {
      let _ : ArrayView[Int] = middle // middle is ArrayView[Int]  
      assert_true(first is 10 && middle is [20, 25] && rest is 30)
    }
  }
  fn process_point(point : Point) -> String {
    match point {
      { x: 0, y: 0, } => "origin"
      { x, y, } if x == y => "diagonal"
      { x, .. } if x < 0 => "negative x"
      _ => "other"
    }
  }
  /// StringView pattern matching for parsing
  fn is_palindrome(s : StringView) -> Bool {
    for remaining = s {
      match remaining {
        [] | [_] => break true
        [a, .. rest, b] if a == b => continue rest
        // a is of type Char, rest is of type StringView
        _ => break false
      }
    }
  }
  inspect(process_point(Point::{ x: 0, y: 0, }), content="origin")
  assert_true(is_palindrome("radar"))
  assert_false(is_palindrome("hello"))
}
```

## Functional `for` control flow

Use explicit loop binders for state, `continue` to update it, and `break` to return a result. The older `loop ... { ... }` syntax is deprecated.

```mbt check
///|
/// Functional loop with pattern matching on loop variables
/// @list.List is from the standard library
fn sum_list(list : @list.List[Int]) -> Int {
  for remaining = list, acc = 0 {
    match remaining {
      Empty => break acc
      More(x, tail=rest) => continue rest, x + acc
    }
  }
}

///|
///  Multiple loop variables with complex control flow
fn find_pair(arr : Array[Int], target : Int) -> (Int, Int)? {
  for i = 0, j = arr.length() - 1 {
    if i >= j {
      break None
    } else {
      let sum = arr[i] + arr[j]
      if sum == target {
        break Some((i, j)) // Found pair
      } else if sum < target {
        continue i + 1, j // Move left pointer
      } else {
        continue i, j - 1 // Move right pointer
      }
    }
  }
}

///|
test "functional loop results" {
  inspect(sum_list(@list.List([1, 2, 3])), content="6")
  debug_inspect(find_pair([1, 2, 4, 8], 6), content="Some((1, 2))")
  debug_inspect(find_pair([1, 2, 4, 8], 20), content="None")
}
```

For an infinite loop without state, use `for ;; { ... }` or `while true { ... }`.


## Methods and Traits

Methods use `Type::method_name` syntax, traits require explicit implementation:

```mbt check
///|
struct Rectangle {
  width : Double
  height : Double
}

///|
// Methods are prefixed with Type::
fn Rectangle::area(self : Rectangle) -> Double {
  self.width * self.height
}

///|
/// Static methods don't need self
fn Rectangle::new(w : Double, h : Double) -> Rectangle {
  { width: w, height: h, }
}

///|
/// Show trait now uses output(self, logger) for custom formatting
/// to_string() is automatically derived from this
pub impl Show for Rectangle with fn output(self, logger) {
  logger.write_string("Rectangle(\{self.width}x\{self.height})")
}

///|
/// Trait methods can take Self as a receiver
trait Named {
  fn name(Self) -> String
}

///|
/// Trait bounds in generics
fn[T : Show + Named] describe(value : T) -> String {
  "\{Named::name(value)}: \{Show::to_string(value)}"
}

///|
///  Trait implementation
pub impl Hash for Rectangle with fn hash_combine(self, hasher) {
  hasher.combine(self.width)
  hasher.combine(self.height)
}

///|
impl Named for Rectangle with fn name(_self) {
  "rectangle"
}

///|
test "methods and trait calls" {
  let rect = Rectangle::new(3, 4)
  inspect(rect.area(), content="12")
  inspect(describe(rect), content="rectangle: Rectangle(3x4)")
}
```

## Operator Overloading

MoonBit supports operator overloading through traits:

```mbt check
///|
struct Vector(Int, Int)

///|
/// Implement arithmetic operators
pub impl Add for Vector with fn add(self, other) {
  Vector(self.0 + other.0, self.1 + other.1)
}

///|
struct Person {
  age : Int
} derive(Eq)

///|
/// Comparison operators
pub impl Compare for Person with fn compare(self, other) {
  self.age.compare(other.age)
}

///|
test "overloading" {
  let v1 : Vector = Vector(1, 2)
  let v2 : Vector = Vector(3, 4)
  let _v3 : Vector = v1 + v2
  assert_true(Person::{ age: 20, } < Person::{ age: 30, })
}
```

## Access Control Modifiers

MoonBit has fine-grained visibility control:

```mbt check
///|
/// `fn` defaults to Private - only visible in current package
fn internal_helper() -> Unit {
  ()
}

///|
pub fn get_value() -> Int {
  internal_helper()
  42
}

///|
// Struct (default) - type visible, implementation hidden
struct DataStructure {}

///|
/// `pub struct` defaults to readonly - can read, pattern match, but not create
pub struct Config {}

///|
///  Public all - full access
pub(all) struct Config2 {}

///|
/// Abstract trait (default) - cannot be implemented by
/// types outside this package
pub trait MyTrait {}

///|
///  Open for extension
pub(open) trait Extendable {}

///|
test "construct types inside their defining package" {
  let _data = DataStructure::{ }
  let _config = Config::{ }
  inspect(get_value(), content="42")
}
```
