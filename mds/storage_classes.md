# Understanding Memory Layout and Storage Classes in C++

C++ programs are organized in memory into several **sections** or
**segments**. Understanding these helps us know where variables are
stored, how they persist, and their lifetimes.

------------------------------------------------------------------------

## Sections of a C++ Program in Memory

A typical C++ program's memory layout looks like this:

    +---------------------------+
    |        Stack              |
    |   (local variables)       |
    +---------------------------+
    |        Heap               |
    | (dynamic allocations)     |
    +---------------------------+
    |   Uninitialized Data (.bss)|
    | (global/static = 0)       |
    +---------------------------+
    |   Initialized Data (.data) |
    | (global/static ≠ 0)       |
    +---------------------------+
    |         Code (.text)       |
    | (compiled instructions)    |
    +---------------------------+

### 1. **Code Section (.text)**

-   Contains the **compiled instructions** of your program.
-   Read-only to prevent accidental modification of executable code.
-   Example: function bodies.

``` cpp
void greet() { 
    std::cout << "Hello, World!"; 
}
```

### 2. **Initialized Data Section (.data)**

-   Stores **global** and **static** variables **initialized** with a
    non-zero value.
-   Exists throughout the program lifetime.

``` cpp
int global_var = 10;  // Stored in .data
```

### 3. **Uninitialized Data Section (.bss)**

-   Stores **global** and **static** variables **initialized to zero**
    or **not initialized**.
-   Allocated at runtime, initialized to zero automatically.

``` cpp
static int counter;   // Stored in .bss (default 0)
```

### 4. **Heap Section**

-   Used for **dynamic memory allocation** via `new`, `malloc`, etc.
-   Managed manually by the programmer.
-   Grows upward.

``` cpp
int* ptr = new int(5); // Stored in heap
```

### 5. **Stack Section**

-   Used for **function calls** and **local variables**.
-   Memory is automatically managed (pushed and popped).
-   Grows downward.

``` cpp
void foo() {
    int local = 42; // Stored in stack
}
```

------------------------------------------------------------------------

## Storage Classes in C++

Storage classes define the **scope**, **lifetime**, and **visibility**
of variables.

| Storage Class     | Keyword     | Default Value | Scope       | Lifetime      | Memory Section      |
|------------------|------------|---------------|------------|---------------|-------------------|
| Automatic         | `auto` (default) | Garbage       | Local      | Until function returns | Stack              |
| Register          | `register` | Garbage       | Local      | Until function returns | CPU Register / Stack |
| Static (local)    | `static`   | Zero          | Local      | Entire program | `.data` or `.bss`  |
| Static (global)   | `static`   | Zero          | Global     | Entire program | `.data` or `.bss`  |
| Extern            | `extern`   | Depends       | Global     | Entire program | `.data` or `.bss`  |
| Mutable           | `mutable`  | Depends       | Class member | Until object destroyed | Heap/Stack depending on object |


------------------------------------------------------------------------

## Mapping Storage Classes to Memory Sections

| Example                      | Storage Class            | Memory Section     |
|------------------------------|--------------------------|--------------------|
| `int x = 5;` (inside main)   | auto                     | Stack              |
| `static int count;`          | static                   | .bss               |
| `int global = 10;`           | extern/global            | .data              |
| `int* p = new int(3);`       | auto + heap allocation   | Heap               |
| `register int r = 5;`        | register                 | Register / Stack   |


## Example Program

``` cpp
#include <iostream>
using namespace std;

int global_var = 10;        // .data
static int static_global;   // .bss

void demo() {
    int local = 5;          // stack
    static int static_local = 7; // .data
    int* heap_ptr = new int(42); // heap
    cout << "Local: " << local << ", Heap: " << *heap_ptr << endl;
    delete heap_ptr;
}

int main() {
    demo();
    return 0;
}
```

------------------------------------------------------------------------

## Diagram: Complete Memory Layout

            +----------------------------------+
            |           Stack                  |
            |   - Function call frames         |
            |   - Local variables              |
            +----------------------------------+
            |           Heap                   |
            |   - Dynamic memory               |
            +----------------------------------+
            |   Uninitialized (.bss)           |
            |   - static int x;                |
            |   - int global_uninit;           |
            +----------------------------------+
            |   Initialized (.data)            |
            |   - int global_init = 5;         |
            |   - static int local_init = 7;   |
            +----------------------------------+
            |           Code (.text)           |
            |   - main(), demo(), etc.         |
            +----------------------------------+

------------------------------------------------------------------------

## Summary

-   **Stack:** Local and temporary data.
-   **Heap:** Dynamic runtime allocations.
-   **.data:** Initialized globals/statics.
-   **.bss:** Zero-initialized globals/statics.
-   **.text:** Program instructions.

------------------------------------------------------------------------

## Understanding Static Variables in Depth

### What Makes `static` Special?

-   A **static variable** inside a function is **initialized only
    once**, not every time the function is called.
-   It **retains its value** between function calls.
-   It has **local scope** (not visible outside the function) but
    **global lifetime**.

### Key Points:

-   Initialized only once at program startup (if not explicitly
    initialized, it defaults to zero).
-   Memory is allocated in the **.data** (if initialized) or **.bss**
    (if uninitialized) section.
-   Value persists across multiple calls to the same function.

### Example:

``` cpp
#include <iostream>
using namespace std;

void counterFunction() {
    static int count = 0; // initialized once
    count++;
    cout << "Count = " << count << endl;
}

int main() {
    counterFunction();  // Output: Count = 1
    counterFunction();  // Output: Count = 2
    counterFunction();  // Output: Count = 3
    return 0;
}
```

### How It Works Internally:

1.  The first time `counterFunction()` is called, `count` is initialized
    to `0`.
2.  On subsequent calls, `count` retains its last value instead of
    reinitializing.
3.  This behavior makes static variables ideal for maintaining **state**
    between function calls.

### Visual Representation:

    +---------------------------------------------+
    | Function Call Stack                         |
    |   local variables -> destroyed after return  |
    +---------------------------------------------+
    | .data section                               |
    |   static int count = 0;  ← persists forever |
    +---------------------------------------------+

This shows that even though `count` is declared inside a function, its
memory **does not live on the stack**.\
Instead, it resides in the **data segment**, making it available
throughout the program's execution.

------------------------------------------------------------------------

### Summary Table for `static`

| Property        | Local Static                   | Global Static                     |
|-----------------|-------------------------------|-----------------------------------|
| Scope           | Within function               | Within translation unit (.cpp file) |
| Lifetime        | Entire program                | Entire program                     |
| Initialization  | Once only                     | Once only                          |
| Memory Section  | .data / .bss                  | .data / .bss                       |
| Typical Use     | Retain value between function calls | Hide variable/function from other files |


------------------------------------------------------------------------

Static variables are often misunderstood in C++, but mastering them
helps in writing efficient and predictable code that maintains internal
state without global exposure.

# Note on `register` Variables in C++

- Declaring a variable with the `register` keyword:

```cpp
register int counter = 0;
```

- **Does NOT guarantee** that the variable will reside in a CPU register.
- It is only a **compiler optimization hint**.
- Modern compilers often ignore this keyword and manage registers automatically.
- Reasons it might not be placed in a register:
  1. Limited number of CPU registers.
  2. Compiler optimization strategies determine better storage location.

- Therefore, `register` mainly serves as historical or readability guidance rather than a strict directive.
