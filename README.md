
![wax](assets/wax.svg)

# wax

**wax** is a tiny language designed to transpile to other languages easily. Currently supported backends: **C**, **Java** and **TypeScript**. (WIP: WebAssembly, Planned: C++, Python, ...)

### [Playground]() | [Quickstart]() | [Examples]()

The goal of wax is to be a "common subset" of most major imperative programming languages. By lacking the unique fancy features in each of these languages and being as boring as possible, wax transpiles to all of them seamlessly, producing outputs that are:

- Readable: The output code looks just like the input code.
- Editable: A human programmer should be able to work from the output even if the orignal wax source is unavailable.
- Integratable: The output code can be imported as libraries to use with the rest of the target language (in addition to just being runnable alone).

These of course, from the programmers' perspective, come at the cost of losing some of the interesting features offered by other languages. Nevertheless, wax retains the crucial bits for a programming language to be productive.

The syntax of wax is inspired by WebAssembly Text Format (wat), hence the name. Though it uses S-expressions reminiscent of the Lisp family, it is actually quite **imperative** and most resemblant of C in its design. The idea of transpiling to many languages is inspired by Haxe.

wax is currently experimental, so there might be bugs as well as aspects to be improved, in which case PR and Issues are very much appreciated.


## Hello World

```scheme
(func main (result int)
  (print "hello world!")
  (return 0)
)
```

Newlines and indentations are entirely cosmetic. You can use any type of brackets anywhere (`()` `[]` `{}`). You can mix different types of brackets if you somehow prefer that.

```scheme
[func main [result int] [print "hello world!"] [return 0]]
{func main {result int} {print "hello world!"} {return 0}}
{func main [result int] (print "hello world!") (return 0)}
```

Here's an in-place quicksort to get a quick taste of the language:

```scheme
;; sort array in-place for index range [lo,hi] inclusive
(func qksort_inplace (param A (arr float)) (param lo int) (param hi int)
	(if (>= lo hi) (then
		(return)
	))
	(let pivot float (get A lo))
	(let left  int lo)
	(let right int hi)
	(while (<= left right) (do
		(while (< (get A left) pivot) (do
			(set left (+ left 1))
		))
		(while (> (get A right) pivot) (do
			(set right (- right 1))
		))
		(if (<= left right) (then
			(let tmp float (get A left))
			(set A left (get A right))
			(set A right tmp)
			(set left  (+ left 1))
			(set right (- right 1))
		))
	))
	(call qksort_inplace A lo right)
	(call qksort_inplace A left hi)
)

(func qksort (param A (arr float))
	(if (! (# A)) (then
		(return)
	))
	(call qksort_inplace A 0 (- (# A) 1))
)
```

As you might have noticed, writing in wax is pretty much like writing an abstract syntax tree directly!


## Overview

- wax is strongly statically typed.
- wax has built-in support for arrays, hashtables and structs.
- wax supports C-like macros, allowing specifying different behavior/branches for each compile target, as well as embedding target code directly.
- syntax is simple: an expression is always a list of tokens encolsed in parenthesis `()`, and the first token is always a keyword/operator. There're 50 keywords in total.
- wax does not support OOP or functional programming.
- wax does not have a boolean: zero is false, non-zero is true.
- wax is not garbage-collected. However, it does have constructs to facilitate memory management and make leaky bugs less likely. On compile targets that do support garbage collection (e.g. Java, JS), explicit freeing of resources is not required, and theoratically you can ignore memory management altogether if you intend to compile to these targets only. Check out the Memory Management section for details.

## The Compiler

This repo contains a reference implementation of wax called `waxc`, written from scratch in C99.

- Compiles from wax to C, Java and TypeScript.
- It seems pretty fast. Compiling a 700 lines file takes 0.015 seconds on Macbook Pro 2015. Comparison: the output TypeScript, which is also 700 lines long, took `tsc` 1.5 seconds to compile. 
- It can print the tokenization and the abstract syntax tree.
- Usage:

```
 _____                                          
|||'  |                                         
|''   |                                         
|_WAX_| Compiler                                

built Oct 18 2020                              

USAGE: waxc [options] code.wax                  

OPTIONS:                                        
--c    path/out.c     transpile to c            
--java path/out.java  transpile to java         
--ts   path/out.ts    transpile to typescript   
--tokens              print tokenization        
--ast                 print abstract syntax tree
--silent              don't print info          
--help                print this message   
```

### Example

To compile the `fib.wax` example included in the example folder to C, and print the abstract sytax tree to terminal:

```bash
./waxc examples/fib.wax --c fib.c --ast
```

Now compile the C output with gcc and run the example:

```bash
gcc fib.c
./a.out
```

Compile to all targets and compile all outputs with target languages' compilers and run all outputs of target languages' compilers:

```bash
./waxc examples/fib.wax --c fib.c   --java fib.java   --ts fib.ts;
                        gcc fib.c;    javac fib.java;   tsc fib.ts;
                        ./a.out;      java fib;         node fib.js; 
```


### Compiling the Compiler

You need:

- A C compiler that supports C99. e.g. `gcc` or `clang`.

To compile:

```bash
gcc src/wax.c -o waxc
```

That's it, no dependencies.

Alternatively you can run the Makefile:

- `make c`. Compile it.
- `make co`. Compile it with `-std=c99 -O3 -std=c99 -pedantic -Wall`.
- `make em`. Compile it with emscripten as a node.js app. (You might need to edit the rule based on how/when/where you installed emscripten.)
- `make emlib`. Compile it as a javascript library with emscripten, without filesystem dependencies. This is what powers the [online playground]().


# Quick Tour

A quick tour of the wax language to get started.

## Variables & types

There are only 7 types in wax. 

- `int`: integer
- `float`: floating-point number
- `string`: string
- `vec`: fixed size array
- `arr`: dynamically resizable array
- `map`: hashtables
- `struct` : user defined data structures


### Variable Declaration


```scheme
(let x int)

```

Shorthand for initializing to a value.

```scheme
(let x int 42)

```

Declaring a compound type, array of floats:

```scheme
(let x (arr float))

```

An array of 3D vectors:

```scheme
(let x (arr (vec 3 float)))

```

Variables default to the zero value of their type when an initial value is not specified. `int` and `float` default to `0`. Other types default to null. Null is not a type itself in wax, but nullable objects can be nullified with expression `(null x)`. To check if a variable is NOT null, use `(?? x)` (equivalent to `x!=null` in other languages).

You can also use `local` in place of `let`, to declare variables that gets automatically freed when it goes out of scope. See next section for details.

### Varaible Assignment

```scheme
(set x 42)
```

## Math

Math in wax uses prefix notation like the rest of the language. e.g.:

```scheme
(+ 1 1)
```

When nested, `(1 + 2) *3` is:

```scheme
(* (+ 1 2) 3)
```

`+` `*` `&&` `||` can take multiple parameters, which is a shorthand that gets expanded by the compiler.

e.g., `1 + 2 + 3 + 4` is:

```scheme
(+ 1 2 3 4)
```
`a && b && c` is:

```scheme
(&& a b c)
```
which the compiler will read as:

```scheme
(&& (&& a b) c)
```

### Ternary operator

`?` is the ternary operator in wax. e.g., `y = (x==0) ? 1 : 2` is:

```scheme
(set y (? (= x 0) 1 2))
```


### List of operators

These operators work just like their namesakes in other languages.

```
<< >> = && || >= <= <>
+ - * / ^ % & | ! ~ < >
```

Note: `<>` is `!=`. `=` is `==`. `^` is xor.

## Arrays and Vectors

### Initializing

To allocate a `vec` or an `arr`, use `(alloc <type>)`

```scheme
(let x (vec 3 float) (alloc (vec 3 float)))
```

To free it, use

```scheme
(free x)
```

**Important:** Freeing the container does not free the individual elements in it if the elements of the array is not a primitive type (`int`/`float`). Simple rule: if you `alloc`'ed something, you need to `free` it yourself. The container is more like a management service that helps you arrange data; it does not take ownership.


To allocate something that is automatically freed when it goes out of scope, use the `local` keyword to declare it.

```scheme
(local x (vec 3 float) (alloc (vec 3 float)))

```

The memory will be freed at the end of the block the variable belongs to, or immediately before any `return` statement. `local` variables cannot be returned or accessed out of its scope.

You can also use a literal to initialize the `vec` or `arr`, by listing the elements in `alloc` expression.


```scheme
(let x (arr float) (alloc (arr float) 1.0 2.0 3.0))

```

Think of it as

```java
float[] x = new float[] {1.0, 2.0, 3.0};
```

### Accessing

To get the `i`th element of `arr` `x`:

```scheme
(get x i)
```

To set the `i`th element of `arr` `x` to `v`;

```scheme
(set x i v)
```

`get` supports a "chaining" shorthand when you're accessing nested containers. For example, if `x` is a `(arr (arr int))` (2D array),

```scheme
(get x i j)
```
is equivalent to

```scheme
(get (get x i) j)
```

To set the same element to `v`, you'll need

```scheme
(set (get x i) j v)
```

If the array is 3D, then `get` will be:

```scheme
(get x i j k)
```
or (if you enjoy typing):

```scheme
(get (get (get x i) j) k)
```
and `set` will be:

```scheme
(set (get x i j) k v)
```

### Operations

To find out the length of an array `x`, use `#` operator:

```scheme
(# x)
```

`vec`'s length is already burnt into its type, so `#` is not needed.

To insert `v` into a an array `x` at index `i`, use

```scheme
(insert x v)
```

So say to push to the end of the array, you might use

```scheme
(insert x (# x) v)
```

To remove `n` values starting from index `i` from array `x`, use

```scheme
(remove x i n)
```

To produce a new array that contains a range of values from an array `x`, starting from index `i` and with length `n`, use

```scheme
(set y (arr int) (slice x i n))
```

Note that if the result of `slice` operation is neither assigned to anything nor returned, it would be a memory leak since `slice` allocates a new array.

These four are the only operations with syntax level support (`#`, `insert` `remove` and `slice` are keywords). Other methods can be implemented as function calls derived from these fundemental operations.


## Maps

```scheme
(let m (map str int) (alloc (map str int)))

(set m "xyz" 123)

(insert m "abc" 456) ; exactly same as 'set'

(print (get m "xyz"))

(remove m "xyz")

(print (get m "xyz")) 
;^ if a value is not there, the "zero" value of the element type is returned
; for numbers, 0; for compound types, null.

```

Map key type can be `int` `float` or `str`. Map value type can be anything.

## Structs

```scheme
(struct point
    (let x float)
    (let y float)
)
```
Structs are declared with `struct` keyword. In it, fields are listed with `let` expessions, though initial values cannot be specified (they'll be set to zero values of respective types when the struct gets allocated).

Another example: structs used for implementing linked lists might look something like this:

```scheme
(struct list
	(let len int)
	(let head (struct node))
	(let tail (struct node))
)
(struct node
	(let prev (struct node)) 
	(let next (struct node))
	(let data int)
)
```

Structs fields of struct type are always references. Think of them as pointers in C:

```c
struct node {
    struct node * prev;
    struct node * next;
    int data;
};
```

However the notion of "pointers" is hidden in wax; From user's perspective, all non-primitives (`arr`,`vec`,`map`,`str`,`struct`) are manipulated as references.

### Instantiating

```scheme
(let p (struct point) (alloc (struct point)))
```

To free:

```scheme
(free p)
```

The `local` keyword works for structs the same way it does for arrays and vectors.

### Accessing

The `get` and `set` keywords are overloaded for structs too.

To get field `x` of a `(struct point)` instance `p`:

```scheme
(get p x)
```

To set field `x` of struct `point` to `42`:

```scheme
(set p x 42.0)
```

If the struct is an element of an array, say the `j`th point of the `i`th polyline in the `arr` of `polylines`:

```scheme
(get polylines i j x)
(set (get polylines i j) x 42.0)
```

## Strings

In wax, string is an object similar to an array.

To initialize a new string:

```scheme
(let s str (alloc str "hello"))
```

To free the string:

```scheme
(free s)
```

To append to a string, use `<<` operator.

```scheme
(<< s " world!")
```

Now `s` is modified in place, to become `"hello world!"`.

Note that a string does not always need to be allocated to be used, but it needs to be allocated if it needs to:

- be modified
- persist outside of its block

E.g. if all you want is to just print a string:

```scheme
(let s str "hello world")
(print s)
```

is OK. (And so is `(print "hello world")`)

The right-hand-side of string `<<` operator does not have to be allocated, while the left-hand-side must be.

If the function returns a string, it needs to be allocated.

To add a character to a string, `<<` can also be used:

```scheme
(<< s 'a')
```
Note that `'a'` is just an integer (ASCII of `a` is 97). It's the same as:

```scheme
(<< s 97)
```

To add the string `"97"` instead, `cast` expression can be used (see more about casting in next section):

```scheme
(<< s (cast 97 str))
```

Strings can be compared with `=` and `<>` equality tests. They actually check if the strings contain the same content, NOT just checking if they're the exact same object.

```scheme
(let s str (alloc str "hello"))
(<< s "!!")
(print (= s "hello!!"))
;; prints 1
```

## Casting

Ints and Floats can be cast to each other implicitly. You can also use `(cast var to_type)` to do so explicitly:

```scheme
(let x float 3.14)
(let y int (cast x int))
```

Numbers can be cast to and from string with the same `cast` keyword.

```scheme
(let x float (cast "3.14" float))
(let y str (cast x str))
```

In other words, `cast` is also responsible for doing `parseInt` `parseFloat` `toString` present in other languages.

Types other than `int` `float` `str` cannot be `cast`ed. You can define and call custom functions to do the job.

## Control Flow


### If statement

```scheme
(if (= x 42) (then
    (print "the answer is 42")
))
```

with else:

```scheme
(if (= x 42) (then
    (print "the answer is 42")
)(else
    (print "what?")
))
```

else if:

```scheme
(if (= x 42) (then
    (print "the answer is 42")
)(else (if (< x 42) (then
    (print "too small!")
)(else (if (> x 42) (then
    (print "too large!")
)(else    
    (print "impossible!")
))))))
```
with `&&` and `||` and `!`:

```scheme
(if (|| (! (&& (= x 42) (= y 666))) (< z 0)) (then
    (print "complicated logic evaluates to true")
))

```


### For loops


```scheme
(for i 0 (< i 100) 1 (do
    (print "i is:")
    (print i)
))
```

The first argument is the looper variable name, the second is starting value, the third is stopping condition, the fourth is the increment, the fifth is a `(do ...)` expression containing the body of the loop. It is equivalent to:

```java
for (int i = 0; i < 100; i+= 1){

}
```
Looping backwards (99, 98, 97, ..., 2, 1, 0), iterating over the same set of numbers as above:

```scheme
(for i 99 (>= i 0) -1 (do

))
```
Looping with a step of 3 (0, 3, 6, 9, ...):

```scheme
(for i 0 (< i 100) 3 (do

))
```

### Iterate over a Map

```scheme
(let m (map str int) (alloc (map str int)))

; populate ...

(for k v m (do
    (print "key is")
    (print k)
    (print "val is")
    (print v)
))
```

### While loops

```scheme
(while (<> x 0) (do

))
```
which is equivalent to

```java
while (x != 0){

}
```

### Break

Break works for both 

```scheme
(while (<> x 0) (do
    (if (= y 0) (then
        (break)
    ))
))
```

## Functions

A minimal function:

```scheme
(func foo 
    (return)
)
```

A simple function that adds to integers, returning an int:

```scheme
(func add_ints (param x int) (param y int) (result int)
    (return (+ x y))
)
```
A function that adds 1 to each element in an array of floats, in-place:

```scheme
(func add_one (param a (arr float))
    (for i 0 (< i (# a)) 1 (do
        (set a i (+ (get a i) 1.0))
    ))
)
```

Fibonacci:

```scheme
(func fib (param i int) (result int)
	(if (<= i 1) (then
		(return i)
	))
	(return (+
		(call fib (- i 1))
		(call fib (- i 2))
	))
)
```

### Calling

To call a function, use `call` keyword, followed by function name, and the list of arguments.

```scheme
(call foo 1 2)
```

### The main function

The main function is optional. If you're making a library, then you probably don't want to include a main function. If you do, the main function will map to the main function of the target language, (if the target language has the notion of main function, that is).

The main function has 2 signatures, similar to C. One that takes no argument, and returns an int that is the exit code. The other one takes one argument, which is an array of strings containing the commandline arguments, and returns the exit code.

```scheme
(func main (result int)
    (return 0)
)

(func main (param args (arr str)) (result int)
    (for i 0 (< i (# args)) 1 (do
        (print (get args i))
    ))
    (return 0)
)
```

### Function Signature

Functions need to be defined before they're called. Therefore for mutually recursive functions, (or for organizing code), it is useful to declare a signature first.

```scheme
(func foo (param x int) (param y int) (result int))
```

It looks just like a function without body. Therefore, to instead define a  void function that actually does nothing at all, a single `return` needs to be the body to make it not turn into a function signature.

```scheme
(func do_nothing
    (return)
)
```

### File Layout

Functions and structures can only be defined on the top level. So your source code file might looks something like this:

```scheme
;; constants
(let XYZ int 1)
(let ZYX float 2.0)

;; data structures
(struct Foo
    (let x int 3)
)

;; implementation
(func foo
    (return)
)
(func bar
    (return)
)

;; optional main function
(func main (result int)
    (call foo)
    (call bar)
)
```

## Macros

wax supports C-like preprocessor directives. In wax, macros look like other expressions, but the keywords are prefixed with `@`.


```scheme
(@define MY_CONSTANT 5)

(@if MY_CONSTANT 5
    (print "yes, it's")
    (print @MY_CONSTANT)
)
```

after it goes through the preprocessor, the above becomes:

```scheme
(print "yes, it's")
(print 5)
```

Note the `@` in `@MY_CONSTANT` when it is used outside of a macro expression.

If a macro is not defined, and is tested in an `@if` macro, the value defaults to `0`:

```scheme
(@if IM_NOT_DEFINED 1
    (print "never gets printed")
)
(@if IM_NOT_DEFINED 0
    (print "always gets printed")
)

```
The second argument to `@define` can be omitted, which makes it default to `1`:

```scheme
(@define IMPLICITLY_ONE)

(print "printing '1' now:")
(print @IMPLICITLY_ONE)
```

### Including Files

To include another source code, use:

```scheme
(@include "path/to/file.wax")
```
The content of the included file gets dumped into exactly where this `@include` line is. To make sure a file doesn't get included multiple times, use:


```scheme
(@pragma once)
```

### Target-specific behaviors

These macros are pre-defined to be `1` when the wax compiler is asked to compile to a specific language, so the user can specify different behavior for different lanauges:

```c
TARGET_C
TARGET_JAVA
TARGET_TS
...
```

For example:

```scheme
(@if TARGET_C 1
    (print "hello from C")
)
(@if TARGET_JAVA 1
    (print "hello from Java")
)
```

## External Functions

To call functions written in the target language, you can describe their signature with `extern` keyword so that the compiler doesn't yell at you for referring to undefined things.

For example:

```scheme
(extern sin (param x float) (result float))
```

Then in your code, you can write:

```scheme
(call sin 3.14)
```

## Inline "Assembly"

You can embed fragments of the target language into wax, similar to embedding assembly in C, using the `(asm "...")` expression. For example:

```java
(@if TARGET_C 1
    (asm "printf(\"hello from C\n\");")
)
(@if TARGET_JAVA 1
    (asm "System.out.println(\"hello from Java\n\");")
)
(@if TARGET_TS 1
    (asm "console.log(\"hello from TypeScript\n\");")
)
```



