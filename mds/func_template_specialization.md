# C++ Function Template Specialization

## Table of Contents

1. [What is Function Template Specialization?](#what-is-function-template-specialization)
2. [Full Function Template Specialization](#full-function-template-specialization)
3. [Partial Function Template Specialization](#partial-function-template-specialization)
4. [Understanding ODR (One Definition Rule)](#understanding-odr-one-definition-rule)
5. [Inline Requirements Summary](#inline-requirements-summary)
6. [Function Template Overloading vs Specialization](#function-template-overloading-vs-specialization)
7. [Practical Examples](#practical-examples)
8. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
9. [Key Takeaways](#key-takeaways)

---

## What is Function Template Specialization?

Function template specialization allows you to provide a custom implementation of a template function for specific template arguments. This is useful when the generic algorithm doesn't work well for certain types or when you need optimized behavior for specific types.

**Important distinction from class templates:**
- **Full specialization**: Supported for function templates
- **Partial specialization**: **NOT supported** for function templates (use overloading instead)

[↑ Back to Table of Contents](#table-of-contents)

---

## Full Function Template Specialization

### What is Full Function Template Specialization?

Full function template specialization provides a complete alternative implementation when **all** template parameters are specified with concrete types.

### Syntax and Examples

```cpp
// Primary template
template <typename T>
void print(T value) {
    std::cout << "Generic: " << value << "\n";
}

// Full specialization for const char*
template <>
void print<const char*>(const char* value) {
    std::cout << "String: " << value << "\n";
}

// Full specialization for bool
template <>
void print<bool>(bool value) {
    std::cout << "Boolean: " << (value ? "true" : "false") << "\n";
}

// Usage
int main() {
    print(42);           // Uses primary template
    print("hello");      // Uses const char* specialization
    print(true);         // Uses bool specialization
}
```

### Template Argument Deduction

You can often omit the template arguments in the specialization if they can be deduced:

```cpp
// Primary template
template <typename T>
T max(T a, T b) {
    return (a > b) ? a : b;
}

// Full specialization - explicit template argument
template <>
const char* max<const char*>(const char* a, const char* b) {
    return (strcmp(a, b) > 0) ? a : b;
}

// Alternative: Let compiler deduce (cleaner syntax)
template <>
const char* max(const char* a, const char* b) {
    return (strcmp(a, b) > 0) ? a : b;
}
```

### Key Characteristics

- Uses `template <>` syntax (empty template parameter list)
- Specifies **concrete types** for all template parameters
- Creates a **regular function**, not a template
- Must match the primary template's signature exactly (except for type substitution)

**Important Note:** Because full specialization creates a regular function, it behaves like any other regular function definition and is subject to ODR rules.

[↑ Back to Table of Contents](#table-of-contents)

---

## Partial Function Template Specialization

### Why Partial Specialization is NOT Supported

Unlike class templates, **function templates do NOT support partial specialization**. This is a language limitation.

```cpp
// Primary template
template <typename T, typename U>
void process(T a, U b) {
    std::cout << "Generic\n";
}

// ❌ ERROR: Partial specialization not allowed for function templates
template <typename T>
void process<T, int>(T a, int b) {
    std::cout << "Specialized for int\n";
}
```

### The Solution: Function Overloading

Instead of partial specialization, use **function overloading** to achieve similar results:

```cpp
// Primary template
template <typename T, typename U>
void process(T a, U b) {
    std::cout << "Generic: T and U\n";
}

// Overload for when second parameter is int
template <typename T>
void process(T a, int b) {
    std::cout << "Overload: T and int\n";
}

// Overload for pointer types
template <typename T, typename U>
void process(T* a, U* b) {
    std::cout << "Overload: pointers\n";
}

// Usage
int main() {
    process(1.5, 2.5);      // Generic: T and U
    process(1.5, 2);        // Overload: T and int
    int x = 1, y = 2;
    process(&x, &y);        // Overload: pointers
}
```

### Overloading Patterns

Common patterns that would be partial specialization in classes:

```cpp
// Pattern 1: Same type for multiple parameters
template <typename T>
void compare(T a, T b) {
    std::cout << "Same type comparison\n";
}

// Pattern 2: Pointer types
template <typename T>
void process(T* ptr) {
    std::cout << "Pointer processing\n";
}

// Pattern 3: Const types
template <typename T>
void handle(const T& value) {
    std::cout << "Const reference handling\n";
}

// Pattern 4: Array types
template <typename T, size_t N>
void processArray(T (&arr)[N]) {
    std::cout << "Array of size " << N << "\n";
}
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Understanding ODR (One Definition Rule)

Now that we've seen what function template specializations are, let's understand the **One Definition Rule (ODR)**. This rule is the foundation for why full specializations require `inline` when defined in headers.

### What is ODR?

The One Definition Rule states that:

1. **Variables and non-inline functions** can have only **one definition** in the entire program across all translation units
2. **Templates and inline functions** can be defined in multiple translation units, but all definitions must be **identical**
3. **Function templates** are exempt from ODR violations because they're not instantiated until used

### Why ODR Matters for Function Templates

When you `#include` a header file in multiple `.cpp` files, each `.cpp` file becomes a separate **translation unit**. The linker combines all translation units into the final executable.

#### Example of ODR Violation

```cpp
// utils.h
void regularFunction(int x) {  // Regular function definition in header
    std::cout << x << "\n";
}

// file1.cpp
#include "utils.h"  // Translation unit 1 gets a definition

// file2.cpp
#include "utils.h"  // Translation unit 2 gets a definition

// Linker error: multiple definition of 'regularFunction(int)'
```

### How ODR Affects Function Template Specialization

#### Primary Function Templates Are ODR-Safe

Regular function templates don't violate ODR because:
- Templates are **not compiled until instantiated**
- The compiler generates code only when the template is used
- Multiple identical template definitions are expected and merged by the linker

```cpp
// utils.h - This is fine!
template <typename T>
void print(T value) {
    std::cout << value << "\n";
}

// file1.cpp
#include "utils.h"
print(42);  // Instantiates print<int>

// file2.cpp
#include "utils.h"
print(100);  // Same instantiation - compiler merges them
```

**No ODR violation** because this is a template definition, not an actual function definition.

#### Full Specialization Creates Regular Functions (ODR Risk!)

Here's the critical point: **Full function template specialization creates a regular function**, which means it **follows ODR rules for regular code**.

##### ODR Violation Example with Full Specialization

```cpp
// utils.h
template <typename T>
void print(T value) {
    std::cout << value << "\n";
}

// Full specialization - THIS VIOLATES ODR if in header!
template <>
void print<bool>(bool value) {
    // This is a regular function definition now
    std::cout << (value ? "true" : "false") << "\n";
}

// file1.cpp
#include "utils.h"  // Gets definition of print<bool>

// file2.cpp
#include "utils.h"  // Gets ANOTHER definition of print<bool>

// Linker error: multiple definition of 'print<bool>(bool)'
```

**Why ODR is violated:**
- `print<bool>` is a **regular function**, not a template
- Both `file1.cpp` and `file2.cpp` include the header, creating two definitions
- The linker sees two definitions and reports an error

### The `inline` Solution

The `inline` keyword tells the linker: "Multiple identical definitions are allowed; just pick one."

```cpp
// utils.h
template <typename T>
void print(T value) {
    std::cout << value << "\n";
}

// Using 'inline' makes multiple definitions legal
template <>
inline void print<bool>(bool value) {
    std::cout << (value ? "true" : "false") << "\n";
}

// file1.cpp
#include "utils.h"  // Definition 1

// file2.cpp
#include "utils.h"  // Definition 2 - OK with inline!
```

**With `inline`:** The linker recognizes these as intentionally duplicated definitions and merges them.

### Visual Summary: ODR and Function Templates

```
Primary Template:           ┌─────────────────┐
template <typename T>       │   Template      │
void func(T) { }            │  (ODR-exempt)   │
                            └─────────────────┘
                                    ↓
                            Multiple includes OK
                            Compiler handles it


Function Overload:          ┌─────────────────┐
template <typename T>       │   Template      │
void func(T*) { }           │  (ODR-exempt)   │
                            └─────────────────┘
                                    ↓
                            Multiple includes OK
                            Compiler handles it


Full Specialization:        ┌─────────────────┐
template <>                 │ Regular Function│
void func<int>(int) { }     │  (ODR applies!) │
                            └─────────────────┘
                                    ↓
                            ┌──────────┴──────────┐
                            ↓                     ↓
                      In header file          In CPP file
                    (needs 'inline'!)      (no 'inline' needed)
                            ↓                     ↓
                    Would violate ODR      Only one definition
                    without 'inline'
```

### Three Ways to Avoid ODR Violations

#### Option 1: Use `inline` Keyword in Header

```cpp
// utils.h
template <typename T>
void print(T value) {
    std::cout << value << "\n";
}

template <>
inline void print<bool>(bool value) {  // inline required
    std::cout << (value ? "true" : "false") << "\n";
}
```

**No ODR violation:** Explicit `inline` keyword allows multiple definitions.

#### Option 2: Move to CPP File (Single Definition)

```cpp
// utils.h
template <typename T>
void print(T value) {
    std::cout << value << "\n";
}

// Declaration only
template <>
void print<bool>(bool value);

// utils.cpp
template <>
void print<bool>(bool value) {  // No inline needed
    std::cout << (value ? "true" : "false") << "\n";
}
```

**No ODR violation:** Only one translation unit has the definition.

#### Option 3: Use Function Overloading Instead

```cpp
// utils.h - No specialization, just overloading
template <typename T>
void print(T value) {
    std::cout << value << "\n";
}

// Regular overload - still a template
inline void print(bool value) {
    std::cout << (value ? "true" : "false") << "\n";
}
```

**Note:** This is not a specialization but an overload, which may have different resolution rules.

[↑ Back to Table of Contents](#table-of-contents)

---

## Inline Requirements Summary

Now that we understand ODR and how it applies to function template specializations, here's a quick reference for when `inline` is required:

### Full Function Template Specialization

| Location | Inline Required? | Reason |
|----------|------------------|--------|
| Header | **YES** (must use `inline`) | Regular function definition - would violate ODR without `inline` |
| CPP file | No | Only one translation unit has the definition |

### Primary Function Template

| Location | Inline Required? | Reason |
|----------|------------------|--------|
| Header | No | Template definition - ODR doesn't apply to templates |
| CPP file | **Not recommended** | Templates need to be visible at instantiation point - causes linker errors |

### Function Overloads (Alternative to Partial Specialization)

| Location | Inline Required? | Reason |
|----------|------------------|--------|
| Header (template overload) | No | Still a template - ODR doesn't apply to templates |
| Header (non-template overload) | **YES** | Regular function - would violate ODR without `inline` |
| CPP file | Depends | Template overloads not recommended; non-template OK |

**Key Difference from Class Templates:** Function template specializations are always defined in one place (not split between declaration and definition), so the inline requirement is simpler.

[↑ Back to Table of Contents](#table-of-contents)

---

## Function Template Overloading vs Specialization

Understanding when to use overloading versus specialization is crucial for function templates.

### Overload Resolution Order

The compiler selects functions in this order:
1. **Non-template functions** (exact match)
2. **Template overloads** (more specialized)
3. **Primary template** (most generic)
4. **Template specializations** are considered **after** selecting the best template

### Example: Surprising Behavior

```cpp
// Primary template
template <typename T>
void process(T value) {
    std::cout << "Primary template\n";
}

// Overload for pointers
template <typename T>
void process(T* value) {
    std::cout << "Pointer overload\n";
}

// Full specialization of primary template
template <>
void process<int*>(int* value) {
    std::cout << "int* specialization\n";
}

int main() {
    int x = 42;
    int* ptr = &x;
    
    process(ptr);  // What gets called?
    // Answer: "Pointer overload" - NOT the specialization!
    // The overload is more specialized than the primary template,
    // so the specialization of the primary template is never considered
}
```

### Best Practice: Prefer Overloading

```cpp
// ✅ BETTER: Use overloading instead of specialization
template <typename T>
void process(T value) {
    std::cout << "Generic\n";
}

template <typename T>
void process(T* value) {
    std::cout << "Pointer\n";
}

// For specific types, use non-template overload
inline void process(int* value) {
    std::cout << "int pointer\n";
}
```

### When to Use Specialization

Use full specialization when:
- You need to completely replace the implementation for a specific type
- The specialization is for the **exact template signature** being used
- You understand overload resolution and have verified it behaves as expected

Use overloading when:
- You want to handle patterns (pointers, arrays, const, etc.)
- You want more predictable behavior
- You need "partial specialization" behavior (not supported for functions)

[↑ Back to Table of Contents](#table-of-contents)

---

## Practical Examples

### Example 1: String Handling Specialization

```cpp
// utils.h
#include <iostream>
#include <cstring>

// Primary template
template <typename T>
bool isEqual(T a, T b) {
    return a == b;
}

// Full specialization for C-strings (must use inline in header)
template <>
inline bool isEqual<const char*>(const char* a, const char* b) {
    return strcmp(a, b) == 0;
}

// Usage
int main() {
    std::cout << isEqual(5, 5) << "\n";           // Uses primary template
    std::cout << isEqual("hello", "hello") << "\n"; // Uses specialization
}
```

### Example 2: Performance Optimization

```cpp
// algorithm.h
#include <algorithm>
#include <cstring>

// Primary template - element by element
template <typename T>
void copyArray(T* dest, const T* src, size_t count) {
    for (size_t i = 0; i < count; ++i) {
        dest[i] = src[i];
    }
}

// Specialization for trivially copyable types - use memcpy
template <>
inline void copyArray<int>(int* dest, const int* src, size_t count) {
    std::memcpy(dest, src, count * sizeof(int));
}

template <>
inline void copyArray<double>(double* dest, const double* src, size_t count) {
    std::memcpy(dest, src, count * sizeof(double));
}
```

### Example 3: Using Overloading Instead of Specialization

```cpp
// printer.h
#include <iostream>
#include <vector>

// Primary template
template <typename T>
void print(const T& value) {
    std::cout << value << "\n";
}

// Overload for vectors (still a template - no inline needed)
template <typename T>
void print(const std::vector<T>& vec) {
    std::cout << "[";
    for (size_t i = 0; i < vec.size(); ++i) {
        std::cout << vec[i];
        if (i < vec.size() - 1) std::cout << ", ";
    }
    std::cout << "]\n";
}

// Overload for bool (non-template - needs inline)
inline void print(bool value) {
    std::cout << (value ? "true" : "false") << "\n";
}
```

### Example 4: Multiple Template Parameters

```cpp
// comparator.h
#include <iostream>

// Primary template
template <typename T, typename U>
bool areEqual(T a, U b) {
    return false;  // Different types - not equal
}

// Specialization when both types are the same
template <typename T>
inline bool areEqual(T a, T b) {
    return a == b;
}

// Specialization for comparing int and double
template <>
inline bool areEqual<int, double>(int a, double b) {
    return static_cast<double>(a) == b;
}
```

### Example 5: Separating Declaration and Definition

```cpp
// math_utils.h
template <typename T>
T square(T value);

// Specialization declaration
template <>
int square<int>(int value);

// math_utils.cpp
#include "math_utils.h"

template <typename T>
T square(T value) {
    return value * value;
}

// Specialization definition (no inline needed in .cpp)
template <>
int square<int>(int value) {
    std::cout << "Squaring int: " << value << "\n";
    return value * value;
}

// Explicit instantiation for types you want to support
template double square<double>(double);
template float square<float>(float);
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Common Mistakes to Avoid

### Mistake 1: Forgetting `inline` in Header

```cpp
// ❌ WRONG: Full specialization in header without inline
// utils.h
template <typename T>
void func(T value) { }

template <>
void func<int>(int value) { }  // ODR violation!

// ✅ CORRECT: Use inline
template <>
inline void func<int>(int value) { }
```

### Mistake 2: Attempting Partial Specialization

```cpp
// ❌ WRONG: Partial specialization not allowed
template <typename T, typename U>
void process(T a, U b) { }

template <typename T>
void process<T, int>(T a, int b) { }  // Compilation error!

// ✅ CORRECT: Use overloading
template <typename T>
void process(T a, int b) { }
```

### Mistake 3: Specialization After Overload

```cpp
// ❌ PROBLEMATIC: Specialization may not be called
template <typename T>
void func(T value) { std::cout << "Primary\n"; }

template <typename T>
void func(T* value) { std::cout << "Pointer overload\n"; }

template <>
void func<int*>(int* value) { std::cout << "int* spec\n"; }

int x = 0;
func(&x);  // Calls "Pointer overload", not "int* spec"!

// ✅ BETTER: Use overloading consistently
inline void func(int* value) { std::cout << "int* overload\n"; }
```

### Mistake 4: Declaring Specialization Before Primary Template

```cpp
// ❌ WRONG: Specialization declared before primary template
template <>
void func<int>(int value);

template <typename T>
void func(T value);  // Primary template comes too late

// ✅ CORRECT: Primary template first
template <typename T>
void func(T value);

template <>
void func<int>(int value);
```

### Mistake 5: Template Parameter Mismatch

```cpp
// Primary template with default argument
template <typename T = int>
void func(T value) { }

// ❌ WRONG: Specialization must match exactly
template <>
void func<>(int value) { }  // Ambiguous

// ✅ CORRECT: Explicit type
template <>
void func<int>(int value) { }
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Key Takeaways

1. **Full specialization only**: Function templates support full specialization but NOT partial specialization
2. **Use overloading**: For pattern-based behavior, use function overloading instead of attempting partial specialization
3. **Inline in headers**: Full specializations in headers MUST use `inline` to avoid ODR violations
4. **CPP file option**: Full specializations can go in `.cpp` files without `inline`
5. **Overload resolution**: Specializations are considered AFTER overload resolution, which can lead to surprising behavior
6. **Prefer overloading**: In most cases, function overloading is clearer and more predictable than specialization
7. **Primary template first**: Always declare the primary template before any specializations
8. **ODR is the reason**: Full specializations create regular functions that must follow ODR

### Quick Decision Guide

- Need to handle patterns (pointers, const, etc.)? → **Use overloading**
- Need to completely replace implementation for one specific type? → **Use full specialization**
- Putting specialization in header? → **Must use `inline`**
- Want partial specialization behavior? → **Use overloading (or a class template helper)**

[↑ Back to Table of Contents](#table-of-contents)
