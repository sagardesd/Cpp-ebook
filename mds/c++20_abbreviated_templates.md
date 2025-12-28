# C++20 Abbreviated Function Templates

## Table of Contents
1. [What is an Abbreviated Function Template?](#what-is-an-abbreviated-function-template)
2. [Traditional Template vs Abbreviated Template](#traditional-template-vs-abbreviated-template)
   - [Traditional Template Function](#traditional-template-function)
   - [C++20 Abbreviated Template Function](#c20-abbreviated-template-function)
3. [Key Differences](#key-differences)
4. [Constrained Auto Abbreviated Functions](#constrained-auto-abbreviated-functions)
   - [The Problem with Unconstrained Auto](#the-problem-with-unconstrained-auto)
   - [Solution: Using Concepts with Constrained Auto](#solution-using-concepts-with-constrained-auto)
   - [Equivalent Traditional Syntax](#equivalent-traditional-syntax-1)
   - [Benefits of Constrained Auto](#benefits-of-constrained-auto)
5. [Important Limitation: No Abbreviated Class Templates](#important-limitation-no-abbreviated-class-templates)
6. [Advantages of Abbreviated Function Templates](#advantages-of-abbreviated-function-templates)
7. [When to Use](#when-to-use)
8. [Compilation](#compilation)

---

## What is an Abbreviated Function Template?

An **abbreviated function template** is a C++20 feature that allows you to write template functions using `auto` as a parameter type instead of explicitly declaring template parameters. This provides a more concise and readable syntax for function templates.

In C++20, when you use `auto` (or a constrained `auto` with concepts) as a function parameter type, the compiler automatically treats it as a template parameter. Each `auto` parameter introduces an independent template type parameter.

**Syntax:**
```cpp
// Unconstrained auto
auto functionName(auto param1, auto param2) {
    // function body
}

// Constrained auto with concepts
auto functionName(ConceptName auto param1, ConceptName auto param2) {
    // function body
}
```

This is equivalent to:
```cpp
// Unconstrained equivalent
template<typename T1, typename T2>
auto functionName(T1 param1, T2 param2) {
    // function body
}

// Constrained equivalent
template<ConceptName T1, ConceptName T2>
auto functionName(T1 param1, T2 param2) {
    // function body
}
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Traditional Template vs Abbreviated Template

### Traditional Template Function

```cpp
#include <iostream>
#include <typeinfo>

template <typename T>
T min(const T& a, const T& b) {
    std::cout << "Type of a: " << typeid(a).name() 
              << " Type of b: " << typeid(b).name() << " Min: ";
    return a < b ? a : b;
}

int main() {
    std::cout << min(1, 2) << std::endl;
    std::cout << min(2.7, 2.5) << std::endl;
    std::cout << min('a', 'b') << std::endl;
    return 0;
}
```

**Output:**
```
Type of a: i Type of b: i Min: 1
Type of a: d Type of b: d Min: 2.5
Type of a: c Type of b: c Min: a
```

**Template Expansion (using `clang++ -std=c++20 -Xclang -ast-print -fsyntax-only`):**
```cpp
template <typename T> T min(const T &a, const T &b) {
    std::cout << "Type of a: " << typeid(a).name() 
              << " Type of b: " << typeid(b).name() << " Min: ";
    return a < b ? a : b;
}

template<> int min<int>(const int &a, const int &b) { /* ... */ }
template<> double min<double>(const double &a, const double &b) { /* ... */ }
template<> char min<char>(const char &a, const char &b) { /* ... */ }
```

---

### C++20 Abbreviated Template Function

```cpp
#include <iostream>
#include <typeinfo>

auto min(const auto& a, const auto& b) {
    std::cout << "Type of a: " << typeid(a).name() 
              << " Type of b: " << typeid(b).name() << " Min: ";
    return a < b ? a : b;
}

int main() {
    std::cout << min(1, 2) << std::endl;
    std::cout << min(2.7, 2.5) << std::endl;
    std::cout << min('a', 'b') << std::endl;
    return 0;
}
```

**Output:**
```
Type of a: i Type of b: i Min: 1
Type of a: d Type of b: d Min: 2.5
Type of a: c Type of b: c Min: a
```

**Template Expansion (using `clang++ -std=c++20 -Xclang -ast-print -fsyntax-only`):**
```cpp
auto min(const auto &a, const auto &b) {
    std::cout << "Type of a: " << typeid(a).name() 
              << " Type of b: " << typeid(b).name() << " Min: ";
    return a < b ? a : b;
}

template<> int min<int, int>(const int &a, const int &b) { /* ... */ }
template<> double min<double, double>(const double &a, const double &b) { /* ... */ }
template<> char min<char, char>(const char &a, const char &b) { /* ... */ }
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Key Differences

1. **Syntax**: The abbreviated form uses `auto` instead of explicit `template<typename T>` declaration
2. **Each `auto` is independent**: Notice in the expansion that the abbreviated version creates `min<int, int>`, `min<double, double>`, etc., meaning each `auto` parameter is a separate template parameter
3. **Readability**: The abbreviated syntax is more concise and resembles regular function syntax

### Equivalent Template Syntax

The abbreviated function:
```cpp
auto min(const auto& a, const auto& b) {
    std::cout << "Type of a: " << typeid(a).name() 
              << " Type of b: " << typeid(b).name() << " Min: ";
    return a < b ? a : b;
}
```

Is exactly equivalent to:
```cpp
template<typename T1, typename T2>
auto min(const T1& a, const T2& b) {
    std::cout << "Type of a: " << typeid(a).name() 
              << " Type of b: " << typeid(b).name() << " Min: ";
    return a < b ? a : b;
}
```

**Important:** Each `auto` parameter becomes an independent template parameter (`T1`, `T2`). This means the function can accept two different types, such as `min(5, 3.14)` where `a` is `int` and `b` is `double`.

[↑ Back to Table of Contents](#table-of-contents)

---

## Constrained Auto Abbreviated Functions

C++20 also allows you to add **constraints** to abbreviated function templates using concepts. This ensures that the template parameters meet certain requirements.

### The Problem with Unconstrained Auto

Consider this example with a custom type:

```cpp
#include <iostream>
#include <typeinfo>
#include <string>

struct StudentId {
    std::string name;
    std::string id;
};

auto min(const auto& a, const auto& b) {
    std::cout << "Type of a: " << typeid(a).name() 
              << " Type of b: " << typeid(b).name() << " Min: ";
    return a < b ? a : b;  // Error! StudentId doesn't have operator<
}

int main() {
    StudentId s1{"Alice", "001"};
    StudentId s2{"Bob", "002"};
    
    std::cout << min(1, 2) << std::endl;           // Works
    std::cout << min(s1, s2) << std::endl;         // Compilation Error!
    return 0;
}
```

**Error:** `StudentId` doesn't have `operator<` defined, so the comparison `a < b` fails.

---

### Solution: Using Concepts with Constrained Auto

We can create a custom concept to constrain our function:

```cpp
#include <iostream>
#include <typeinfo>
#include <string>
#include <concepts>

struct StudentId {
    std::string name;
    std::string id;
    
    // Define comparison operator
    bool operator<(const StudentId& other) const {
        return name < other.name;
    }
};

// Custom concept for types that support < operator
template<typename T>
concept Comparable = requires(T a, T b) {
    { a < b } -> std::convertible_to<bool>;
};

auto min(const Comparable auto& a, const Comparable auto& b) {
    std::cout << "Type of a: " << typeid(a).name() 
              << " Type of b: " << typeid(b).name() << " Min: ";
    return a < b ? a : b;
}

int main() {
    StudentId s1{"Alice", "001"};
    StudentId s2{"Bob", "002"};
    
    std::cout << min(1, 2) << std::endl;
    std::cout << min(2.7, 2.5) << std::endl;
    std::cout << min('a', 'b') << std::endl;
    
    auto result = min(s1, s2);
    std::cout << result.name << " (ID: " << result.id << ")" << std::endl;
    
    return 0;
}
```

**Output:**
```
Type of a: i Type of b: i Min: 1
Type of a: d Type of b: d Min: 2.5
Type of a: c Type of b: c Min: a
Type of a: 9StudentId Type of b: 9StudentId Min: Alice (ID: 001)
```

The `Comparable` concept checks if a type supports the `<` operator:

```cpp
template<typename T>
concept Comparable = requires(T a, T b) {
    { a < b } -> std::convertible_to<bool>;
};
```

This requires that for type `T`, the expression `a < b` must be valid and convertible to `bool`.

### Equivalent Traditional Syntax

The constrained abbreviated function is equivalent to:

```cpp
template<Comparable T1, Comparable T2>
auto min(const T1& a, const T2& b) {
    std::cout << "Type of a: " << typeid(a).name() 
              << " Type of b: " << typeid(b).name() << " Min: ";
    return a < b ? a : b;
}
```

### Benefits of Constrained Auto

1. **Compile-time error checking**: Catch type errors early with clear error messages
2. **Self-documenting code**: The constraint explains what types are acceptable
3. **Better IDE support**: IDEs can provide better autocomplete and hints
4. **Type safety**: Prevents misuse of generic functions

[↑ Back to Table of Contents](#table-of-contents)

---

## Important Limitation: No Abbreviated Class Templates

**C++20 does NOT support abbreviated class templates.** You cannot write:
```cpp
// This is NOT valid C++20
class MyClass<auto T> {  // Error!
    T value;
};
```

**Reason:** Abbreviated function templates work because the compiler can deduce template parameters from function arguments at the call site. Class templates require explicit instantiation (e.g., `MyClass<int>`), so there's no argument deduction context for `auto` to work with.

You must still use traditional template syntax for classes:
```cpp
// Correct way for class templates
template<typename T>
class MyClass {
    T value;
public:
    MyClass(T v) : value(v) {}
    
    // But member functions CAN use abbreviated templates!
    auto add(auto other) {
        return value + other;
    }
    
    auto compare(const auto& other) const {
        return value < other;
    }
};
```

**Example Usage:**
```cpp
int main() {
    MyClass<int> obj(10);           // Class needs explicit type
    
    std::cout << obj.add(5) << std::endl;      // Member function: auto deduces int
    std::cout << obj.add(3.14) << std::endl;   // Member function: auto deduces double
    std::cout << obj.compare(20) << std::endl; // Member function: auto deduces int
    
    return 0;
}
```

**Output:**
```
15
13.14
1
```

This demonstrates that while class templates must use traditional syntax, their member functions can freely use abbreviated function template syntax.

[↑ Back to Table of Contents](#table-of-contents)

---

## Advantages of Abbreviated Function Templates

1. **Conciseness**: Less boilerplate code
2. **Readability**: Easier to read and understand at a glance
3. **Flexibility**: Each `auto` can deduce to a different type
4. **Modern**: Aligns with modern C++ practices

[↑ Back to Table of Contents](#table-of-contents)

---

## When to Use

Abbreviated function templates are ideal for:
- Simple generic functions
- Lambda expressions
- Functions where the template nature is obvious from context
- Reducing syntactic noise in template-heavy code

For complex templates with constraints, explicit template syntax or C++20 concepts may be more appropriate for clarity.

[↑ Back to Table of Contents](#table-of-contents)

---

## Compilation

To compile code using abbreviated function templates:
```bash
g++ -std=c++20 program.cpp -o program
clang++ -std=c++20 program.cpp -o program
```

[↑ Back to Table of Contents](#table-of-contents)
