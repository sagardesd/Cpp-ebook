# Why C++ Templates Must Be in Headers

## Table of Contents

1. [The Problem](#the-problem)
2. [Understanding Two-Phase Translation](#understanding-two-phase-translation)
3. [Why the Linker Error Occurs](#why-the-linker-error-occurs)
4. [The Solution](#the-solution)
5. [Alternative Solutions](#alternative-solutions)
6. [Common Errors and How to Fix Them](#common-errors-and-how-to-fix-them)
7. [Best Practices](#best-practices)

---

## The Problem

Let's start with a real-world example that causes a linker error:

```cpp
// vector.h
template<typename T>
class vector {
public:
    T& at(int);
};
```

```cpp
// vector.cpp
#include "vector.h"

template <typename T>
T& vector<T>::at(int i) {
    // some code
}
```

```cpp
// main.cpp
#include "vector.h"

int main() {
    vector<int> a;
    a.at(5);
}
```

When you compile this:

```bash
g++ vector.cpp main.cpp
```

**You get a linker error:**

```
/usr/bin/ld: /tmp/cc6PAyEd.o: in function `main':
main.cpp:(.text+0x28): undefined reference to `vector<int>::at(int)'
collect2: error: ld returned 1 exit status
```

**Why does this happen?** The answer lies in how C++ compiles templates.

[↑ Back to Table of Contents](#table-of-contents)

---

## Understanding Two-Phase Translation

C++ templates use a **two-phase translation model**. Understanding this is crucial to understanding why templates must be in headers.

### Phase 1: Template Parsing (Definition Point)

When the compiler first encounters a template definition, it performs **Phase 1** processing:

```
┌─────────────────────────────────────────────┐
│ Phase 1: Template Parsing                   │
├─────────────────────────────────────────────┤
│ ✓ Parse the syntax                          │
│ ✓ Check template structure                  │
│ ✓ Store the template body                   │
│ ✓ Resolve non-dependent names               │
│ ✗ Does NOT generate actual code             │
│ ✗ Does NOT substitute template parameters   │
└─────────────────────────────────────────────┘
```

**In our example (vector.cpp):**

```cpp
template <typename T>
T& vector<T>::at(int i) {
    // some code
}
```

During Phase 1:
- The compiler parses this template
- Checks that the syntax is valid
- Stores the template definition internally
- **No actual code is generated yet**
- The compiler doesn't know what `T` will be

### Phase 2: Template Instantiation (Usage Point)

**Phase 2** happens when you **use** the template with a specific type:

```
┌─────────────────────────────────────────────┐
│ Phase 2: Template Instantiation             │
├─────────────────────────────────────────────┤
│ ✓ Triggered when template is USED           │
│ ✓ Substitute template parameters (T = int)  │
│ ✓ Resolve all type-dependent operations     │
│ ✓ Check if operations are valid for type    │
│ ✓ Generate actual machine code              │
└─────────────────────────────────────────────┘
```

**In our example (main.cpp):**

```cpp
vector<int> a;
a.at(5);
```

When the compiler sees `a.at(5)`, it needs to:
1. Substitute `T = int`
2. Generate the actual code for `vector<int>::at(int)`
3. Check if all operations are valid for `int`

**Critical Point:** To do Phase 2, the compiler **must see the complete template definition**!

### The Complete Flow

```
Template Definition (vector.cpp)
        │
        ▼
┌───────────────────┐
│   Phase 1:        │
│   Parse & Store   │  ← Compiler stores template
└────────┬──────────┘     but generates NO code
         │
         │ (Template remains dormant)
         │
         ▼
Template Usage (main.cpp)
vector<int> a;
a.at(5);
        │
        ▼
┌───────────────────┐
│   Phase 2:        │
│   Instantiate     │  ← Needs template definition!
│   - Substitute T  │     But it's in vector.cpp
│   - Generate code │     which is NOT visible here
└────────┬──────────┘
         │
         ▼
    ❌ ERROR!
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Why the Linker Error Occurs

Let's trace exactly what happens during compilation:

### Step 1: Compile vector.cpp

```bash
g++ -c vector.cpp -o vector.o
```

```
┌─────────────────────────────────────────────┐
│ Compiling vector.cpp                        │
├─────────────────────────────────────────────┤
│ • Compiler sees template definition         │
│ • Phase 1: Parses and stores template       │
│ • Phase 2: NOT TRIGGERED                    │
│   (template is never used in this file)     │
│ • Result: vector.o contains NO code         │
│           for vector<int>::at(int)          │
└─────────────────────────────────────────────┘
```

**Key point:** Even though `vector.cpp` has the template definition, no `vector<int>::at(int)` code is generated because the template is never instantiated in this file.

### Step 2: Compile main.cpp

```bash
g++ -c main.cpp -o main.o
```

```
┌─────────────────────────────────────────────┐
│ Compiling main.cpp                          │
├─────────────────────────────────────────────┤
│ • #include "vector.h" → only declaration    │
│ • Compiler sees: a.at(5)                    │
│ • Needs to instantiate vector<int>::at(int) │
│ • Phase 2 TRIGGERED                         │
│ • Problem: Can only see declaration         │
│            NOT the definition!              │
│ • Cannot generate code without definition   │
│ • Assumes definition exists elsewhere       │
│ • Creates undefined reference               │
└─────────────────────────────────────────────┘
```

**Key point:** `main.cpp` only sees the declaration from `vector.h`:

```cpp
template<typename T>
class vector {
public:
    T& at(int);  // ← Only this is visible
};
```

It **cannot** see the definition in `vector.cpp`:

```cpp
template <typename T>
T& vector<T>::at(int i) {  // ← This is INVISIBLE
    // some code
}
```

### Step 3: Linking

```bash
g++ vector.o main.o -o program
```

```
┌─────────────────────────────────────────────┐
│ Linking                                     │
├─────────────────────────────────────────────┤
│ • Linker examines main.o                    │
│ • Finds: needs vector<int>::at(int)         │
│ • Searches vector.o for this symbol         │
│ • vector.o has NO such symbol               │
│   (wasn't instantiated there)               │
│ • ❌ LINKER ERROR                            │
│   "undefined reference to                   │
│    vector<int>::at(int)"                    │
└─────────────────────────────────────────────┘
```

### Visualization of the Problem

```
vector.cpp               main.cpp
    │                        │
    ▼                        ▼
┌─────────┐            ┌─────────┐
│Phase 1: │            │Includes │
│Parse    │            │vector.h │
│template │            │(decl    │
│         │            │ only)   │
│No code  │            │         │
│generated│            │Uses:    │
│         │            │a.at(5)  │
└────┬────┘            │         │
     │                 │Phase 2: │
     │                 │needs    │
     │                 │def! ❌  │
     │                 └────┬────┘
     ▼                      ▼
┌─────────┐            ┌─────────┐
│vector.o │            │main.o   │
│         │            │         │
│No       │            │Undefined│
│vector   │            │reference│
│<int>::  │            │to       │
│at(int)  │            │vector   │
│         │            │<int>::  │
│         │            │at(int)  │
└────┬────┘            └────┬────┘
     │                      │
     └──────────┬───────────┘
                ▼
           ┌─────────┐
           │ Linker  │
           │  ❌     │
           │ Error!  │
           └─────────┘
```

### Why Separate Compilation is the Issue

Each `.cpp` file is a **separate translation unit**:

```
Project Structure:
├── vector.cpp  → Compiled independently → vector.o
├── main.cpp    → Compiled independently → main.o
└── vector.h    → Included in both files
```

- `vector.cpp` and `main.cpp` **don't see each other** during compilation
- They are compiled completely separately
- The compiler cannot "look ahead" to see what's in other files
- When compiling `main.cpp`, the compiler has **no idea** that `vector.cpp` exists

[↑ Back to Table of Contents](#table-of-contents)

---

## The Solution

The solution is simple: **Put the template definition in the header file.**

### Understanding the Key Insight

> **Templates don't emit code until instantiated, so include the .cpp in the .h instead of the other way around!**

This is the opposite of what you do with regular C++ code:
- **Regular code:** Declare in `.h`, define in `.cpp`, include the `.h`
- **Template code:** Declare in `.h`, define in `.cpp`, **include the `.cpp` at the end of the `.h`**

Or better yet: Just put everything in the `.h` file!

### Corrected Code

```cpp
// vector.h
template<typename T>
class vector {
public:
    T& at(int i) {
        // some code - DEFINITION IN HEADER
    }
};
```

**Or if you prefer to separate declaration and definition:**

```cpp
// vector.h
template<typename T>
class vector {
public:
    T& at(int);
};

// Definition still in the header
template<typename T>
T& vector<T>::at(int i) {
    // some code
}
```

```cpp
// main.cpp
#include "vector.h"

int main() {
    vector<int> a;
    a.at(5);  // ✓ Works!
}
```

### Why This Works

```
┌─────────────────────────────────────────────┐
│ Compiling main.cpp                          │
├─────────────────────────────────────────────┤
│ • #include "vector.h"                       │
│   → Full definition is included             │
│ • Compiler sees: a.at(5)                    │
│ • Phase 2: Instantiate vector<int>::at(int) │
│ • Template definition IS VISIBLE            │
│ • Compiler substitutes T = int              │
│ • Generates actual code                     │
│ • Code is placed in main.o                  │
│ • ✓ SUCCESS - No linker error               │
└─────────────────────────────────────────────┘
```

**Now delete `vector.cpp` entirely** - you don't need it!

```bash
g++ main.cpp -o program  # ✓ Works!
```

### Key Principle

> **Template definitions must be visible at the point of instantiation.**

Since instantiation happens wherever the template is **used** (not where it's **defined**), the definition must be in a header file that can be included everywhere.

[↑ Back to Table of Contents](#table-of-contents)

---

## Alternative Solutions

While putting definitions in headers is the standard approach, there are alternatives:

### Solution 1: Explicit Template Instantiation

If you know **exactly** which types will be used, you can explicitly instantiate them in the `.cpp` file:

```cpp
// vector.h
template<typename T>
class vector {
public:
    T& at(int);
};
```

```cpp
// vector.cpp
#include "vector.h"

template <typename T>
T& vector<T>::at(int i) {
    // some code
}

// Explicit instantiation for specific types
template class vector<int>;     // Generate vector<int>
template class vector<double>;  // Generate vector<double>
template class vector<std::string>; // Generate vector<string>
```

```cpp
// main.cpp
#include "vector.h"

int main() {
    vector<int> a;
    a.at(5);  // ✓ Works! vector<int> was explicitly instantiated
}
```

**Compilation:**
```bash
g++ vector.cpp main.cpp  # ✓ Works!
```

**Pros:**
- Definitions can stay in `.cpp` files
- Faster compilation for large projects
- Reduces code bloat

**Cons:**
- Must know all types in advance
- Users cannot use the template with new types
- Less flexible - not truly generic

### Solution 2: Include Implementation at End of Header

```cpp
// vector.h
template<typename T>
class vector {
public:
    T& at(int);
};

#include "vector.tpp"  // or "vector_impl.h"
```

```cpp
// vector.tpp (or vector_impl.h)
template <typename T>
T& vector<T>::at(int i) {
    // some code
}
```

**Pros:**
- Separates interface from implementation (for readability)
- Still makes definition visible

**Cons:**
- Confusing naming conventions
- More files to manage
- Not commonly used in practice

[↑ Back to Table of Contents](#table-of-contents)

---

## Common Errors and How to Fix Them

### Error 1: Undefined Reference (Most Common)

```
undefined reference to `vector<int>::at(int)'
```

**Cause:** Template definition in `.cpp` file, not visible at instantiation point

**Fix:** Move template definition to header file

---

### Error 2: Multiple Definition Error

```
multiple definition of `vector<int>::at(int)'
```

**Cause:** Template accidentally instantiated explicitly in multiple `.cpp` files

**Fix:** 
- Remove explicit instantiations
- Keep definition in header (implicit instantiation handles duplicates automatically)
- If using explicit instantiation, only instantiate in ONE `.cpp` file

---

### Error 3: Incomplete Type

```cpp
template<typename T>
class Container {
    void process();  // Declaration only
};

// main.cpp
Container<int> c;
c.process();  // Error: incomplete type
```

**Error:**
```
error: invalid use of incomplete type 'class Container<int>'
```

**Fix:** Include the full definition in the header

---

### Error 4: Circular Dependencies

```cpp
// a.h
#include "b.h"
template<typename T>
class A {
    B<T> b;
};

// b.h
#include "a.h"
template<typename T>
class B {
    A<T> a;
};
```

**Error:** Circular inclusion

**Fix:** Use forward declarations and pointers/references:

```cpp
// a.h
template<typename T> class B;  // Forward declaration

template<typename T>
class A {
    B<T>* b;  // Pointer instead of value
};
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Best Practices

### ✅ DO

1. **Put template definitions in header files**
   ```cpp
   // vector.h
   template<typename T>
   T& vector<T>::at(int i) {
       // definition here
   }
   ```

2. **Use include guards or `#pragma once`**
   ```cpp
   #ifndef VECTOR_H
   #define VECTOR_H
   // template code
   #endif
   ```

3. **Use meaningful file extensions**
   - `.h` or `.hpp` for headers
   - `.tpp` or `_impl.h` for template implementations (if separating)

4. **Consider explicit instantiation for large templates with known types**

5. **Document which types are supported** (if using explicit instantiation)

### ❌ DON'T

1. **Don't put template definitions in `.cpp` files** (unless using explicit instantiation)

2. **Don't forget that templates need complete visibility**

3. **Don't mix implicit and explicit instantiation carelessly**

4. **Don't assume the linker will "figure it out"**

### Quick Decision Guide

```
Are you writing a generic template library?
    └─ Yes → Put definitions in headers

Do you know ALL types that will be used?
    ├─ Yes → Consider explicit instantiation
    └─ No  → Put definitions in headers

Is compilation time a major concern?
    └─ Yes → Use explicit instantiation for known types
              Put definitions in headers for flexibility
```

### Summary: The Golden Rule

> **Template code must be visible where it's instantiated, not where it's defined.**

Since instantiation happens at the **point of use**, and `.cpp` files are compiled separately, template definitions must be in **headers** that can be included wherever needed.

[↑ Back to Table of Contents](#table-of-contents)
