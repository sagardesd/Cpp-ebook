# C++ Class Template Specialization

## Table of Contents

1. [What is Class Template Specialization?](#what-is-class-template-specialization)
2. [Full Template Specialization](#full-template-specialization)
3. [Partial Template Specialization](#partial-template-specialization)
4. [Understanding ODR (One Definition Rule)](#understanding-odr-one-definition-rule)
5. [Inline Requirements Summary](#inline-requirements-summary)
6. [Specializing a Single Member Function](#specializing-a-single-member-function)
7. [Practical Examples](#practical-examples)
8. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
9. [Key Takeaways](#key-takeaways)

---

## What is Class Template Specialization?

Class template specialization allows you to provide a custom implementation of a template class for specific template arguments. This is useful when the generic implementation doesn't work well for certain types or when you need optimized behavior for specific types.

There are two types of specialization:
- **Full (Explicit) Specialization**: Specialize for all template parameters
- **Partial Specialization**: Specialize for some template parameters or patterns

[↑ Back to Table of Contents](#table-of-contents)

---

## Full Template Specialization

### What is Full Template Specialization?

Full template specialization provides a complete alternative implementation when **all** template parameters are specified with concrete types.

### Syntax and Example

```cpp
// Primary template
template <typename T>
class Storage {
public:
    void store(T value) {
        data = value;
        std::cout << "Storing generic type\n";
    }
private:
    T data;
};

// Full specialization for bool
template <>  // Empty angle brackets - all parameters specified
class Storage<bool> {
public:
    void store(bool value) {
        data = value;
        std::cout << "Storing bool efficiently\n";
    }
private:
    unsigned char data; // More efficient storage for bool
};
```

### Key Characteristics

- Uses `template <>` syntax (empty template parameter list)
- Specifies **concrete types** for all template parameters: `Storage<bool>`
- Creates a **completely separate class** - it's not a template anymore
- Can have a completely different implementation from the primary template

**Important Note:** Because full specialization creates a regular (non-template) class, it behaves like any other regular class definition. This has implications for the One Definition Rule, which we'll explore next.

[↑ Back to Table of Contents](#table-of-contents)

---

## Partial Template Specialization

### What is Partial Template Specialization?

Partial specialization allows you to specialize a template for a **pattern** or **subset** of possible template arguments while keeping some template parameters generic.

### Syntax and Examples

```cpp
// Primary template with two parameters
template <typename T, typename U>
class Pair {
public:
    void display() {
        std::cout << "Generic pair\n";
    }
private:
    T first;
    U second;
};

// Partial specialization: both types are the same
template <typename T>  // Still has template parameter
class Pair<T, T> {     // Pattern: same type for both parameters
public:
    void display() {
        std::cout << "Same-type pair\n";
    }
private:
    T first;
    T second;
};

// Partial specialization: second type is int
template <typename T>  // Still has template parameter
class Pair<T, int> {   // Pattern: any type with int
public:
    void display() {
        std::cout << "Pair with int as second\n";
    }
private:
    T first;
    int second;
};

// Partial specialization: pointer types
template <typename T, typename U>  // Still has template parameters
class Pair<T*, U*> {                // Pattern: both are pointers
public:
    void display() {
        std::cout << "Pointer pair\n";
    }
private:
    T* first;
    U* second;
};
```

### Common Partial Specialization Patterns

```cpp
// Original template
template <typename T, typename U, int N>
class Container { };

// Specialize for pointer types
template <typename T, typename U, int N>
class Container<T*, U, N> { };

// Specialize when both types are the same
template <typename T, int N>
class Container<T, T, N> { };

// Specialize for arrays
template <typename T, typename U, int N>
class Container<T[], U, N> { };

// Specialize for const types
template <typename T, typename U, int N>
class Container<const T, U, N> { };
```

### Key Characteristics

- Uses `template <...>` with remaining template parameters
- Specifies a **pattern** using template parameters: `Pair<T, T>`, `Pair<T*, U*>`
- Still a **template** - not a concrete class
- Gets instantiated by the compiler like any template

**Important Note:** Because partial specialization is still a template, it behaves like regular templates and doesn't have the same ODR concerns as full specialization.

[↑ Back to Table of Contents](#table-of-contents)

---

## Understanding ODR (One Definition Rule)

Now that we've seen what full and partial template specializations are, let's understand the **One Definition Rule (ODR)**. This rule is the foundation for why full specializations require special handling with `inline` while partial specializations don't.

### What is ODR?

The One Definition Rule states that:

1. **Variables and non-inline functions** can have only **one definition** in the entire program across all translation units
2. **Classes, templates, and inline functions** can be defined in multiple translation units, but all definitions must be **identical**
3. **Templates** (both class and function templates) are exempt from ODR violations because they're not instantiated until used

### Why ODR Matters

When you `#include` a header file in multiple `.cpp` files, each `.cpp` file becomes a separate **translation unit**. The linker combines all translation units into the final executable.

#### Example of ODR Violation

```cpp
// header.h
void regularFunction() {  // Definition in header
    std::cout << "Hello\n";
}

// file1.cpp
#include "header.h"  // Translation unit 1 gets a definition

// file2.cpp
#include "header.h"  // Translation unit 2 gets a definition

// Linker error: multiple definition of 'regularFunction'
```

**What happens:** The linker sees two identical definitions of `regularFunction` (one from file1.o and one from file2.o) and doesn't know which one to use.

### How ODR Affects Template Specialization

#### Templates Are Naturally ODR-Safe

Regular templates (both primary and partial specializations) don't violate ODR because:
- Templates are **not compiled until instantiated**
- The compiler generates code only when the template is used
- Multiple identical template definitions are expected and merged by the linker

```cpp
// header.h - This is fine!
template <typename T>
class MyClass {
public:
    void method() { }  // OK - template definition
};

template <typename T>
void MyClass<T>::method() { }  // OK - template definition

// file1.cpp
#include "header.h"
MyClass<int> obj1;  // Instantiates template

// file2.cpp
#include "header.h"
MyClass<int> obj2;  // Same instantiation - compiler merges them
```

**No ODR violation** because these are template definitions, not actual function/class definitions.

#### Full Specialization Creates Regular Definitions (ODR Risk!)

Here's the critical point: **Full template specialization creates a regular (non-template) class or function**, which means it **follows ODR rules for regular code**.

##### ODR Violation Example with Full Specialization

```cpp
// header.h
template <typename T>
class Storage {
public:
    void store(T value);
};

// Full specialization - this is now a REGULAR class, not a template!
template <>
class Storage<bool> {
public:
    void store(bool value);
};

// Definition outside class - THIS VIOLATES ODR if in header!
template <>
void Storage<bool>::store(bool value) {
    // This is a regular function definition now
    std::cout << "Storing bool\n";
}

// file1.cpp
#include "header.h"  // Gets definition of Storage<bool>::store

// file2.cpp
#include "header.h"  // Gets ANOTHER definition of Storage<bool>::store

// Linker error: multiple definition of 'Storage<bool>::store(bool)'
```

**Why ODR is violated:**
- `Storage<bool>::store` is a **regular member function**, not a template
- Both `file1.cpp` and `file2.cpp` include the header, creating two definitions
- The linker sees two definitions and reports an error

### The `inline` Solution

The `inline` keyword tells the linker: "Multiple identical definitions are allowed; just pick one."

```cpp
// header.h
template <>
class Storage<bool> {
public:
    void store(bool value);
};

// Using 'inline' makes multiple definitions legal
inline void Storage<bool>::store(bool value) {
    std::cout << "Storing bool\n";
}

// file1.cpp
#include "header.h"  // Definition 1

// file2.cpp
#include "header.h"  // Definition 2 - OK with inline!
```

**With `inline`:** The linker recognizes these as intentionally duplicated definitions and merges them.

### Visual Summary: ODR and Templates

```
Primary Template:           ┌─────────────────┐
template <typename T>       │   Template      │
class MyClass { };          │  (ODR-exempt)   │
                            └─────────────────┘
                                    ↓
                            Multiple includes OK
                            Compiler handles it


Partial Specialization:     ┌─────────────────┐
template <typename T>       │   Template      │
class MyClass<T*> { };      │  (ODR-exempt)   │
                            └─────────────────┘
                                    ↓
                            Multiple includes OK
                            Compiler handles it


Full Specialization:        ┌─────────────────┐
template <>                 │  Regular Class  │
class MyClass<int> { };     │  (ODR applies!) │
                            └─────────────────┘
                                    ↓
                            ┌──────────┴──────────┐
                            ↓                     ↓
                    Inside class body      Outside class body
                    (implicitly inline)    (needs 'inline'!)
                            ↓                     ↓
                    Multiple includes OK   Would violate ODR
                                          without 'inline'
```

### Three Ways to Avoid ODR Violations

#### Option 1: Define Inside Class Body (Implicit Inline)

```cpp
// header.h
template <>
class Storage<bool> {
public:
    void store(bool value) {  // Implicitly inline
        std::cout << "Storing bool\n";
    }
};
```

**No ODR violation:** Functions defined inside class bodies are implicitly `inline`.

#### Option 2: Use Explicit `inline` Keyword

```cpp
// header.h
template <>
class Storage<bool> {
public:
    void store(bool value);
};

inline void Storage<bool>::store(bool value) {
    std::cout << "Storing bool\n";
}
```

**No ODR violation:** Explicit `inline` keyword allows multiple definitions.

#### Option 3: Move to CPP File (Single Definition)

```cpp
// header.h
template <>
class Storage<bool> {
public:
    void store(bool value);  // Declaration only
};

// storage.cpp
void Storage<bool>::store(bool value) {
    std::cout << "Storing bool\n";
}
```

**No ODR violation:** Only one translation unit has the definition.

[↑ Back to Table of Contents](#table-of-contents)

---

## Inline Requirements Summary

Now that we understand ODR and how it applies to template specializations, here's a quick reference for when `inline` is required:

### Full Template Specialization

| Location | Definition | Inline Required? | Reason |
|----------|-----------|------------------|--------|
| Header | Inside class body | No (implicitly inline) | Functions defined in class body are always implicitly inline |
| Header | Outside class body | **YES** (must use `inline`) | Regular class definition - would violate ODR without `inline` |
| CPP file | Outside class body | No | Only one translation unit has the definition |

### Partial Template Specialization

| Location | Definition | Inline Required? | Reason |
|----------|-----------|------------------|--------|
| Header | Inside class body | No (implicitly inline) | Functions defined in class body are always implicitly inline |
| Header | Outside class body | No | Still a template - ODR doesn't apply to templates |
| CPP file | Outside class body | **Not recommended** | Templates need to be visible at instantiation point - causes linker errors |

### Primary Template (for comparison)

| Location | Definition | Inline Required? | Reason |
|----------|-----------|------------------|--------|
| Header | Inside class body | No (implicitly inline) | Functions defined in class body are always implicitly inline |
| Header | Outside class body | No | Template definition - ODR doesn't apply to templates |
| CPP file | Outside class body | **Not recommended** | Templates need to be visible at instantiation point - causes linker errors |

[↑ Back to Table of Contents](#table-of-contents)

---

## Specializing a Single Member Function

You can specialize individual member functions without specializing the entire class. However, **you must fully specialize the class first**, then specialize the member.

### Example: Specializing a Member Function

```cpp
// Primary template
template <typename T>
class Calculator {
public:
    T add(T a, T b);
    T multiply(T a, T b);
};

// Generic implementation
template <typename T>
T Calculator<T>::add(T a, T b) {
    return a + b;
}

template <typename T>
T Calculator<T>::multiply(T a, T b) {
    return a * b;
}

// Specialize only the add() function for std::string
template <>
std::string Calculator<std::string>::add(std::string a, std::string b) {
    return a + " " + b; // Add space between strings
}
// multiply() still uses the generic implementation
```

**Important:** You cannot partially specialize individual member functions. You can only fully specialize them for a specific type.

[↑ Back to Table of Contents](#table-of-contents)

---

## Practical Examples

### Example 1: Full Specialization in Header (Methods Outside)

```cpp
// vector_wrapper.h
#include <vector>
#include <iostream>

template <typename T>
class VectorWrapper {
public:
    void add(T value);
    void print() const;
private:
    std::vector<T> data;
};

// Primary template definitions
template <typename T>
void VectorWrapper<T>::add(T value) {
    data.push_back(value);
}

template <typename T>
void VectorWrapper<T>::print() const {
    for (const auto& item : data) {
        std::cout << item << " ";
    }
    std::cout << "\n";
}

// Full specialization for bool
template <>
class VectorWrapper<bool> {
public:
    void add(bool value);
    void print() const;
private:
    std::vector<bool> data;
};

// MUST use inline when defined outside in header
inline void VectorWrapper<bool>::add(bool value) {
    data.push_back(value);
    std::cout << "Added bool\n";
}

inline void VectorWrapper<bool>::print() const {
    for (bool b : data) {
        std::cout << (b ? "true" : "false") << " ";
    }
    std::cout << "\n";
}
```

### Example 2: Partial Specialization for Pointers

```cpp
// smart_container.h
template <typename T>
class SmartContainer {
public:
    void process(T value);
};

// Primary template
template <typename T>
void SmartContainer<T>::process(T value) {
    std::cout << "Processing value: " << value << "\n";
}

// Partial specialization for pointer types
template <typename T>
class SmartContainer<T*> {
public:
    void process(T* ptr);
};

// No inline needed - still a template
template <typename T>
void SmartContainer<T*>::process(T* ptr) {
    if (ptr) {
        std::cout << "Processing pointer to: " << *ptr << "\n";
    } else {
        std::cout << "Null pointer\n";
    }
}
```

### Example 3: Mixed Definitions (Inside and Outside)

```cpp
// config.h
template <typename T>
class Config {
public:
    // Defined inside - implicitly inline
    void setDefault(T value) {
        defaultValue = value;
    }
    
    T getDefault() const;
private:
    T defaultValue;
};

// Defined outside - no inline needed (template)
template <typename T>
T Config<T>::getDefault() const {
    return defaultValue;
}

// Full specialization for const char*
template <>
class Config<const char*> {
public:
    // Defined inside - implicitly inline
    void setDefault(const char* value) {
        defaultValue = value ? value : "";
    }
    
    const char* getDefault() const;
private:
    std::string defaultValue;
};

// MUST use inline (full specialization in header)
inline const char* Config<const char*>::getDefault() const {
    return defaultValue.c_str();
}
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Common Mistakes to Avoid

```cpp
// ❌ WRONG: Partial specialization in CPP file
// partial_spec.cpp
template <typename T>
void MyClass<T*>::method() { } // Linker error!

// ❌ WRONG: Full specialization without inline in header
// full_spec.h
template <>
void MyClass<int>::method() { } // Multiple definition error!

// ✅ CORRECT: Full specialization with inline in header
// full_spec.h
template <>
class MyClass<int> {
    void method();
};

inline void MyClass<int>::method() { } // OK

// ✅ CORRECT: Partial specialization in header
// partial_spec.h
template <typename T>
void MyClass<T*>::method() { } // OK - still a template
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Key Takeaways

1. **Full specialization** = regular class, follows regular inline rules
2. **Partial specialization** = still a template, follows template rules
3. **Inside class body** = always implicitly inline
4. **Outside in header**:
   - Full specialization → needs `inline`
   - Partial specialization → no `inline` needed
5. **CPP files**: Only full specializations should go there (without `inline`)
6. **Member function specialization**: Only full specialization possible, must specialize entire class type first
7. **ODR is the reason**: Full specializations create regular code that must follow ODR, while templates are ODR-exempt

[↑ Back to Table of Contents](#table-of-contents)