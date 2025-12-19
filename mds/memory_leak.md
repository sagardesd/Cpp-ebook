# Understanding Dynamic memory leaks

## Dynamic Memory Allocation in C++

In C++, when you need to allocate memory dynamically (at runtime), you use the `new` operator to get memory from the **heap**. Unlike stack memory, heap memory is **not automatically managed** - you must manually free it using the `delete` operator.

```
Stack Memory                  Heap Memory
┌──────────────┐             ┌──────────────────┐
│ Automatic    │             │ Manual Management│
│ Cleaned up   │             │ YOU must call    │
│ automatically│             │ delete!          │
└──────────────┘             └──────────────────┘
       ↑                              ↑
  int x = 5;                  int* p = new int(5);
  (destroyed when            (YOU must delete p)
   out of scope)
```

C++ does **not** have automatic garbage collection. 

If you allocate memory with `new`, you **must** free it with `delete`. 

If you forget, that memory is **permanently lost** until your program terminates - this is called a **memory leak**.


Let's start with an example where the prgram calls new to allocate memory and delete to free the memory:

```cpp
#include <iostream>

void good_function(int data) {
    int* rawPtr = new int(data);  // 1. Allocate memory from heap
    std::cout << "data: " << *rawPtr << std::endl;
    delete rawPtr;  // 2. Free the memory 
}

int main() {
    good_function(10);
    return 0;
}
```

**Compile and run with Valgrind:**
```bash
g++ -g -O0 good_example.cpp -o good_example
valgrind --leak-check=full ./good_example
```

**Valgrind Report (No Leaks):**
```
==3102717== HEAP SUMMARY:
==3102717==     in use at exit: 0 bytes in 0 blocks
==3102717==   total heap usage: 3 allocs, 3 frees, 73,732 bytes allocated
==3102717== 
==3102717== All heap blocks were freed -- no leaks are possible
==3102717== 
==3102717== ERROR SUMMARY: 0 errors from 0 contexts (suppressed: 0 from 0)
```

**Memory Lifecycle:**
```
Step 1: int* rawPtr = new int(data)

    STACK                           HEAP
┌──────────────┐              ┌──────────────┐
│  Function    │              │              │
│  Stack Frame │              │  [4 bytes]   │
│              │              │  value: 10   │
│  rawPtr ─────┼──────────────┼─>  [int]     │
│  (address)   │              │              │
└──────────────┘              └──────────────┘
    ↑                              ↑
  Lives here               Lives here until
  (automatic)              delete is called


Step 2: Using *rawPtr
    
    STACK                           HEAP
┌──────────────┐              ┌──────────────┐
│              │              │              │
│  rawPtr ─────┼──────────────┼─>  [10]      │
│  (pointer)   │              │              │
└──────────────┘              └──────────────┘
                              Access via pointer


Step 3: delete rawPtr ✅

    STACK                           HEAP
┌──────────────┐              ┌──────────────┐
│              │              │              │
│  rawPtr      │      X       │  [freed]     │
│  (dangling)  │              │              │
└──────────────┘              └──────────────┘
                              Memory returned to OS


Step 4: Function exits

    STACK                           HEAP
┌──────────────┐              ┌──────────────┐
│              │              │              │
│  [destroyed] │              │  [free]      │
│              │              │              │
└──────────────┘              └──────────────┘
    Stack cleaned up          No leak! ✅
```

Now let's see what happens when you forget to call `delete`:

```cpp
#include <iostream>

void bad_function(int data) {
    int* rawPtr = new int(data);  // Allocate memory from heap
    std::cout << "data: " << *rawPtr << std::endl;
    // Forgot to delete! 
}

int main() {
    bad_function(10);
    return 0;
}
```

**Compile and run with Valgrind:**
```bash
g++ -g -O0 forgot_delete.cpp -o forgot_delete
valgrind --leak-check=full ./forgot_delete
```

**Valgrind Report - Observe the Memory Leak:**
```
==3102369== HEAP SUMMARY:
==3102369==     in use at exit: 4 bytes in 1 blocks
==3102369==   total heap usage: 3 allocs, 2 frees, 73,732 bytes allocated
==3102369== 
==3102369== Searching for pointers to 1 not-freed blocks
==3102369== Checked 147,280 bytes
==3102369== 
==3102369== 4 bytes in 1 blocks are definitely lost in loss record 1 of 1
==3102369==    at 0x4849013: operator new(unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==3102369==    by 0x109201: bad_function(int) (memory_leak.cpp:5)
==3102369==    by 0x10925D: main (memory_leak.cpp:10)
==3102369== 
==3102369== LEAK SUMMARY:
==3102369==    definitely lost: 4 bytes in 1 blocks
==3102369==    indirectly lost: 0 bytes in 0 blocks
==3102369==      possibly lost: 0 bytes in 0 blocks
==3102369==    still reachable: 0 bytes in 0 blocks
==3102369==         suppressed: 0 bytes in 0 blocks
==3102369== 
==3102369== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

### Memory Lifecycle:
```
Step 1: int* rawPtr = new int(data)

    STACK                           HEAP
┌──────────────┐              ┌──────────────┐
│  Function    │              │              │
│  Stack Frame │              │  [4 bytes]   │
│              │              │  value: 10   │
│  rawPtr ─────┼──────────────┼─>  [int]     │
│  (address)   │              │              │
└──────────────┘              └──────────────┘
    ↑                              ↑
  Lives here               Lives here until
  (automatic)              delete is called


Step 2: Using *rawPtr
    
    STACK                           HEAP
┌──────────────┐              ┌──────────────┐
│              │              │              │
│  rawPtr ─────┼──────────────┼─>  [10]      │
│  (pointer)   │              │              │
└──────────────┘              └──────────────┘
                              Access via pointer


Step 3: Function exits (NO delete called!) ❌

    STACK                           HEAP
┌──────────────┐              ┌──────────────┐
│              │              │              │
│  [destroyed] │      X       │  [LEAKED!]   │
│              │              │  value: 10   │
└──────────────┘              └──────────────┘
  rawPtr is gone!             Memory orphaned!
  (pointer destroyed)         No way to free it!


Step 4: Program continues

    STACK                           HEAP
┌──────────────┐              ┌──────────────┐
│              │              │              │
│              │              │  [LEAKED]    │
│              │              │  4 bytes     │
└──────────────┘              └──────────────┘
                              Permanently lost! ❌
```

**The Problem:** When `bad_function()` exits:
1. The local variable `rawPtr` is destroyed (it's on the stack)
2. But the memory it pointed to (on the heap) is **not** freed
3. There's now **no way** to access or free that memory
4. The 4 bytes are **permanently lost** until the program terminates

### Why This Matters at Scale:

```cpp
int main() {
    for (int i = 0; i < 1000000; i++) {
        bad_function(i);  // Leaks 4 bytes EVERY call!
    }
    // Total leaked: 4 MB of memory!
    return 0;
}
```

In a long-running application:
- Memory consumption grows continuously
- System performance degrades
- Eventually: out-of-memory crashes


Even if you remember to call `delete`, exceptions can still cause leaks:

```cpp
#include <iostream>
#include <stdexcept>

void some_function() {
    throw std::runtime_error("Something went wrong");
}

void bad_function(int data) {
    int* rawPtr = new int(data);  // Allocate memory
    std::cout << "data: " << *rawPtr << std::endl;
    some_function();  // Exception thrown here! 
    delete rawPtr;    // This line NEVER executes! ❌
}

int main() {
    try {
        bad_function(10);
    } catch (const std::exception& e) {
        std::cerr << "Caught: " << e.what() << '\n';
    }
    return 0;
}
```

**Compile and run with Valgrind:**
```bash
g++ -g -O0 exception_leak.cpp -o exception_leak
valgrind --leak-check=full ./exception_leak
```

### Valgrind Report - Observe the Exception Leak:
```
data: 10
Caught: Something went wrong
==3106542== 
==3106542== HEAP SUMMARY:
==3106542==     in use at exit: 4 bytes in 1 blocks
==3106542==   total heap usage: 4 allocs, 3 frees, 73,804 bytes allocated
==3106542== 
==3106542== 4 bytes in 1 blocks are definitely lost in loss record 1 of 1
==3106542==    at 0x4849013: operator new(unsigned long) (in /usr/libexec/valgrind/vgpreload_memcheck-amd64-linux.so)
==3106542==    by 0x109381: bad_function(int) (memory_leak_exc.cpp:9)
==3106542==    by 0x1093FD: main (memory_leak_exc.cpp:17)
==3106542== 
==3106542== LEAK SUMMARY:
==3106542==    definitely lost: 4 bytes in 1 blocks
==3106542==    indirectly lost: 0 bytes in 0 blocks
==3106542==      possibly lost: 0 bytes in 0 blocks
==3106542==    still reachable: 0 bytes in 0 blocks
==3106542==         suppressed: 0 bytes in 0 blocks
==3106542== 
==3106542== ERROR SUMMARY: 1 errors from 1 contexts (suppressed: 0 from 0)
```

### The Surprising Result:

**Observation:** Even though we have `delete rawPtr` in the code, the memory still leaked!

- **"4 bytes in 1 blocks are definitely lost"** - Memory leaked despite `delete` being present
- The exception caused the function to exit **before** reaching `delete`
- Leak originated at line 9 (the `new` statement)

### Exception Execution Flow:
```
Step 1: int* rawPtr = new int(data)

    STACK                           HEAP
┌──────────────┐              ┌──────────────┐
│  Function    │              │              │
│  Stack Frame │              │  [4 bytes]   │
│              │              │  value: 10   │
│  rawPtr ─────┼──────────────┼─>  [int]     │
│              │              │              │
└──────────────┘              └──────────────┘


Step 2: some_function() throws exception ⚡

    STACK                           HEAP
┌──────────────┐              ┌──────────────┐
│              │              │              │
│  rawPtr ─────┼──────────────┼─>  [10]      │
│              │              │              │
└──────────────┘              └──────────────┘
    ⚡ Exception!               Still allocated!


Step 3: Stack unwinding begins
        (cleaning up local variables)

    STACK                           HEAP
┌──────────────┐              ┌──────────────┐
│              │              │              │
│  rawPtr      │      ?       │  [10]        │
│ (being       │              │              │
│  destroyed)  │              │              │
└──────────────┘              └──────────────┘
  Pointer about to die        Memory still there!
  delete rawPtr NEVER runs!


Step 4: Function exited via exception

    STACK                           HEAP
┌──────────────┐              ┌──────────────┐
│              │              │              │
│  [destroyed] │      X       │  [LEAKED!]   │
│              │              │  value: 10   │
└──────────────┘              └──────────────┘
  rawPtr destroyed!           Memory orphaned!
  delete line skipped!        No way to free it! ❌
```

**What happened:**
1. Memory is allocated with `new`
2. Exception is thrown in `some_function()`
3. **Stack unwinding** begins (C++ cleans up local objects)
4. `rawPtr` (the pointer variable) is destroyed during unwinding
5. The `delete rawPtr` line is **never reached**
6. The allocated memory has **no way to be freed**

## The Problems with Manual Memory Management

Managing heap memory manually with `new`/`delete` is **error-prone** because:

### Problem 1: Easy to Forget
```cpp
void process() {
    int* data = new int[1000];
    // ... lots of code ...
    // Did you remember to delete[]? ❌
}
```
**Issue:** In complex functions with multiple return paths, it's easy to forget `delete`.

### Problem 2: Exception Safety is Hard
```cpp
void process() {
    int* data = new int[1000];
    // Any function here might throw! ⚡
    complexOperation();  // Exception?
    delete[] data;  // Might never execute!
}
```
**Issue:** Any exception between `new` and `delete` causes a leak.

### Problem 3: Multiple Exit Points
```cpp
void process(bool condition) {
    int* data = new int[1000];
    if (condition) {
        return;  // Oops, forgot delete! ❌
    }
    // ... more code ...
    if (errorOccurred) {
        return;  // Oops again! ❌
    }
    delete[] data;  // Only this path is safe
}
```
**Issue:** Each return path needs its own `delete`.

### Problem 4: Ownership is Unclear
```cpp
int* createData();  // Who deletes this?
void process(int* data);  // Does this take ownership?
int* getData();  // Should caller delete?
```
**Issue:** Unclear who is responsible for freeing memory.


### Detection Tools you can use to findout memory leaks
- **Valgrind** - Detects memory leaks at runtime
- **AddressSanitizer** - Fast leak detection during testing
- **Static analyzers** - Clang-Tidy, Cppcheck find potential leaks

**Uhh So many problems, is there anyway these problems can be fixed.**
**Yes the answer to all these problems is Smart pointers introduced in C++11 based on the RAII concept.**
