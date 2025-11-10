# How I Built a 65 Million Item Array in PHP... Kind Of

## But Why?

Arrays in PHP are great, flexible, dynamic, easy to work with, but I've always wondered how far they could really go. Whether it's Laravel Collections, plain arrays, or objects, I often find myself bumping into performance or memory limits.

So I decided to find out how far I could _really_ push them.
I'm an avid C developer (without _much_ C experience, you know how that goes), and I wanted to peek under the hood to learn how PHP handles its internal data structures.

Spoiler alert: I didn't use PHP arrays.  
I used C structs.

## Why Are PHP Arrays So Heavy?

PHP arrays are wonderful to use, but they're **not** memory efficient.  
That's because, under the hood, arrays in PHP are implemented as **hash tables**.

A hash table isn't just a flat block of memory like in C. It's a complex data structure that maps keys to values by computing a _hash_ (a deterministic number) for each key. This makes associative arrays possible, so we can do `$user['id']` instead of `$user[0]`.

But that flexibility comes at a cost: every key/value pair involves extra bookkeeping, memory for hashes, pointers, type metadata, and reference counts. It's great for ergonomics, not for density.

Objects in PHP are similar: they're also built around hash tables internally, though the Zend Engine optimises access by simulating a "struct-like" layout for class properties.

If you're curious, you can see this in the Zend source code (warning: requires reading C):
- [zend_hash.c](https://github.com/php/php-src/blob/master/Zend/zend_hash.c)
- [zend_objects.c](https://github.com/php/php-src/blob/master/Zend/zend_objects.c)


## Enter FFI - The Bridge Between PHP and C

In C, the language PHP itself is built on, you have **structs**, which are contiguous blocks of memory containing fixed, typed fields. This layout is incredibly efficient, especially for large datasets.

```c
#include <stdio.h>
#include <string.h>

// Define a struct to represent a person
typedef struct {
    int id;
    char name[50];
    float height;
} Person;

int main(void) {
    // Declare a struct variable
    Person alice;

    // Assign values to its fields
    alice.id = 1;
    strcpy(alice.name, "Alice");
    alice.height = 1.68f;

    // Access and print fields
    printf("ID: %d\n", alice.id);
    printf("Name: %s\n", alice.name);
    printf("Height: %.2f m\n", alice.height);

    return 0;
}
```
_Example of implementing structs within C_

Wouldn't it be great if PHP could use those same C structs directly?

It turns out, it can.  
Enter **FFI**, _Foreign Function Interface_, which allows PHP to call directly into C code and allocate native memory.

FFI is powerful, but it's also risky. You can poke directly into memory, forget to free it, or crash PHP entirely if you're not careful. In other words: great power, great responsibility (and maybe a segfault or two).

``` php
<?php

// Define the struct layout using C syntax
$typedef = "
typedef struct {
    int id;
    float value;
} Record;
";

// Create the FFI binding for this definition
$ffi = FFI::cdef($typedef);

// Allocate a new struct instance
$record = $ffi->new("Record");

// Assign values to its fields
$record->id = 42;
$record->value = 3.14;

// Access and print the values
echo "ID: {$record->id}\n";        // ID: 42
echo "Value: {$record->value}\n";  // Value: 3.14

```
_Example of implementing C-like structs within PHP_

This is a great start, but let's see if we can extend it further!

You can read more about FFI here: https://www.php.net/manual/en/book.ffi.php

## Building a `Struct` Class

I wanted a way to use C structs safely from PHP, so I wrote a small library that wraps FFI in a familiar object-oriented interface.

The idea:
- Define your C `typedef struct` in a string
- Create an array of those structs
- Interact with it using `set()` and `get()` methods

Here's what that looks like in practice:

```php
$typedef = "
typedef struct {
    int id;
    float value;
} Record;
";

$count = 100_000;
$records = new Struct($typedef, "Record", $count);

for ($i = 0; $i < $count; $i++) {
    $records->set($i, ['id' => $i, 'value' => (float)$i]);
}

$record = $records->get(0);
```
_Example of using homebrewed `Struct` class_

Under the hood, the `Struct` class:
- Calls `FFI::cdef()` to define the struct
- Allocates a contiguous array (`new Type[N]`)
- Uses PHP's garbage collector for cleanup (you _can_ manage memory manually, but proceed at your own risk)
- Exposes safe methods like `set()`, `get()`, iteration helpers, and size reporting

You can find the code on GitHub as a Composer package:
https://github.com/rayblair06/php-struct

## Results: Structs vs Arrays vs Objects

For this benchmark, I used a simple data structure with two fields: `id` and `value`.

With a 512 MB memory limit, I was able to generate roughly **65 million struct entries**.  
Trying to achieve the same with arrays or objects? PHP taps out far earlier.

**Approximate limits:**

| Type                   | Items        | Notes                                 |
| ---------------------- | ------------ | ------------------------------------- |
| `Struct` (FFI)         | ~65 million  | contiguous C memory                   |
| Object (`RecordClass`) | ~4.1 million | each has its own zval/object overhead |
| Array                  | ~1.4 million | hash table storage                    |
| Object (`stdClass`)    | ~1 million   | dynamic property storage              |

For a smaller test with __500,000__ items:

| Type        | Memory Used |
| ----------- | ----------- |
| Struct      | ~4 MB       |
| RecordClass | ~52 MB      |
| Array       | ~197 MB     |
| `stdClass`  | ~253 MB     |

All this raw power comes with trade-offs: C structs are fixed-type (no strings), require manual field handling, and can't represent complex data structures.

In other words:
PHP arrays give you flexibility, structs give you raw performance. You can't have both... at least, not yet.

You can see the full benchmark implementation here: https://github.com/rayblair06/php-struct/blob/main/bin/benchmarks.php

## Why Does This Matter?

Honestly? It doesn't.  
At least not for your average application. You're probably not handling **65 million** array items, and if you are, you're likely using lazy iteration, streams, buffers, or another architecture designed for that scale. But if you can't, knowing this might just help.

This was mostly a for-science experiment to see how PHP's internals behave when you cut out the abstraction layers. You lose many of PHP's conveniences, type flexibility, automatic resizing, string safety, and gain a lot of sharp edges (like floating-point precision differences and endian quirks).

Still, it's a fascinating way to explore how PHP memory really works, and it shows that PHP's FFI gives you the keys to drive right into C territory.

## What I Learned
- How PHP's memory model and arrays work under the hood
- How FFI bridges PHP and C in a surprisingly powerful way
- Why PHP arrays are magical but expensive
- How fun it is to (responsibly) abuse FFI 😄

I learned a whole lot about low-level concepts and the internal workings of the PHP language, but this only feels like scratching the surface.  

It's worth saying these aren't new problems, and there are extensions that expose lower-level behaviour without needing to dive into C yourself.

For example:
- **`ext/spl` (Standard PHP Library)** offers data structures like `SplFixedArray`, `SplHeap`, and `SplObjectStorage` that provide more predictable memory usage than regular arrays.
- **`ext/ds` (Data Structures)** goes a step further, adding efficient collections such as `Vector`, `Map`, and `Deque` — all written in C for performance.
- And of course, **FFI** itself lets you bridge directly into C, opening the door to everything from native libraries to raw memory access.

Exploring these extensions shows how PHP gives developers different levels of abstraction depending on what they need, from high-level collections for everyday work, to FFI and Zend internals for those who want to tinker with memory and performance.

## Where This Could Go Next

This experiment opens a few interesting doors:
- A **native PHP extension** for faster struct handling
- **Shared-memory structs** for interprocess data
- **File-mapped buffers** for huge datasets
- **Typed arrays** that blend PHP syntax with native memory performance

Or, like many great side projects, it might just rot in my GitHub, never to see the light of day again.

## Closing

You can think of this as PHP meeting C halfway, bringing the predictability and memory efficiency of C structs into a high-level, dynamic environment.

I didn't make PHP faster.  
I didn't break it (mostly).  
But I did build a 65 million-item array... kind of.
