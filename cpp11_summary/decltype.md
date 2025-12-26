# decltype (C++11 to C++20)

## Table of Contents
1. [What is decltype?](#what-is-decltype)
2. [Why decltype is Needed](#why-decltype-is-needed)
3. [How decltype Works in C++11](#how-decltype-works-in-c11)
4. [Type Deduction Rules](#type-deduction-rules)
5. [Evolution in C++14](#evolution-in-c14)
6. [Evolution in C++17](#evolution-in-c17)
7. [Evolution in C++20](#evolution-in-c20)
8. [Common Pitfalls](#common-pitfalls)
9. [Best Practices](#best-practices)

---

## What is decltype?

`decltype` is a **compile-time** type specifier introduced in C++11 that inspects the declared type of an entity or deduces **both the type and value category** of an expression **without evaluating it**. The name stands for "declared type".

### Key Characteristics

- **Compile-time only**: Type deduction happens during compilation, producing zero runtime cost
- **Non-evaluating**: Expressions inside `decltype` are never executed, only analyzed for their type
- **Value category preservation**: `decltype` preserves whether an expression is an lvalue, xvalue, or prvalue, encoding this information in the resulting type (through references)

### Basic Syntax
```cpp
decltype(expression)
```

### Simple Example
```cpp
int x = 42;
decltype(x) y = 10;  // y has type int

const int& z = x;
decltype(z) w = x;   // w has type const int&
```

---

## Why decltype is Needed

Before C++11, there was no way to determine the exact type of an expression at compile time. This created several problems:

### Problem 1: Template Return Type Deduction
```cpp
// Before C++11 - impossible to write correctly for all types
template<typename T, typename U>
??? multiply(T a, U b) {
    return a * b;  // What's the return type?
}
```

### Problem 2: Complex Type Expressions
```cpp
// Hard to maintain - if container type changes, code breaks
std::vector<int> vec;
std::vector<int>::iterator it = vec.begin();
```

### Problem 3: Perfect Forwarding Return Types
```cpp
// How do we preserve the exact return type?
template<typename Func, typename... Args>
??? wrapper(Func f, Args&&... args) {
    return f(std::forward<Args>(args)...);
}
```

### Solutions with decltype
```cpp
// Solution 1: Template return type
template<typename T, typename U>
auto multiply(T a, U b) -> decltype(a * b) {
    return a * b;
}

// Solution 2: Type inference
auto it = vec.begin();  // Type automatically deduced

// Solution 3: Perfect forwarding
template<typename Func, typename... Args>
auto wrapper(Func f, Args&&... args) -> decltype(f(std::forward<Args>(args)...)) {
    return f(std::forward<Args>(args)...);
}
```

---

## How decltype Works in C++11

In C++11, `decltype` has **two completely different behaviors** depending on whether the argument is parenthesized or not.

### The Two Forms

#### Form 1: Variable decltype (unparenthesized id-expression)
Returns the **exact declared type** of a variable, including references.

```cpp
int x = 5;
int& rx = x;
int&& rrx = std::move(x);

decltype(x)    // int
decltype(rx)   // int&
decltype(rrx)  // int&&
```

#### Form 2: Expression decltype (anything else, including parenthesized)
Returns type based on **value category**:
- **prvalue** → `T`
- **lvalue** → `T&`
- **xvalue** → `T&&`

```cpp
int x = 5;

decltype((x))     // int&  (lvalue)
decltype(x + 1)   // int   (prvalue)
decltype(std::move(x))  // int&& (xvalue)
```

### Critical Difference Example
```cpp
int i = 42;

// Safe: returns int (copy)
decltype(auto) fn_A(int i) {
    return i;      // decltype(i) = int
}

// DANGEROUS: returns int& (reference to local variable!)
decltype(auto) fn_B(int i) {
    return (i);    // decltype((i)) = int&
}

int main() {
    int a = fn_A(10);  // OK
    int& b = fn_B(10); // Undefined behavior - dangling reference!
}
```

---

## Type Deduction Rules

### Rule 1: Unparenthesized Variables
```cpp
int x;
const int cx = x;
int& rx = x;
const int& crx = x;

decltype(x)    // int
decltype(cx)   // const int
decltype(rx)   // int&
decltype(crx)  // const int&
```

### Rule 2: Parenthesized Variables
```cpp
int x;

decltype((x))   // int& (always lvalue reference for variables)
```

### Rule 3: Member Access
```cpp
struct S {
    int member;
};

S s;
S f();

decltype(s.member)       // int&  (lvalue)
decltype(f().member)     // int&& (xvalue - temporary object)
decltype(S::member)      // int&  (even outside class context)
```

### Rule 4: Function Calls
Function call expressions take the return type of the function:

```cpp
int func();
int& func_ref();
int&& func_rref();

decltype(func())       // int
decltype(func_ref())   // int&
decltype(func_rref())  // int&&
```

### Rule 5: Operators
```cpp
int a = 5, b = 10;

decltype(a + b)   // int (prvalue)
decltype(a = b)   // int& (assignment returns lvalue reference)
decltype(++a)     // int& (pre-increment returns lvalue reference)
decltype(a++)     // int (post-increment returns prvalue)
decltype(a > b)   // bool (prvalue)
```

### Rule 6: Literals and Constants
```cpp
decltype(42)        // int
decltype(3.14)      // double
decltype("hello")   // const char(&)[6] (array reference)
decltype(nullptr)   // std::nullptr_t
```

### Value Categories Summary

Based on the Stanford article, here's how value categories relate to decltype:

| Value Category | decltype Result | Example |
|----------------|-----------------|---------|
| **prvalue** (pure rvalue) | `T` | `42`, `func()` returning by value |
| **lvalue** | `T&` | Variables, `(x)`, pre-increment |
| **xvalue** (expiring value) | `T&&` | `std::move(x)`, `f().member` |

---

## Evolution in C++14

C++14 introduced significant improvements to make `decltype` easier to use.

### decltype(auto)

The biggest addition was `decltype(auto)`, which combines `auto` type deduction with `decltype` rules.

#### Without decltype(auto) (C++11)
```cpp
template<typename Container>
auto getElement(Container& c, int index) -> decltype(c[index]) {
    return c[index];
}
```

#### With decltype(auto) (C++14)
```cpp
template<typename Container>
decltype(auto) getElement(Container& c, int index) {
    return c[index];  // Preserves reference if c[index] returns reference
}
```

### Key Benefits

1. **Preserves Value Category**
```cpp
std::vector<int> vec = {1, 2, 3};

decltype(auto) elem = vec[0];  // int&, can modify
elem = 42;  // Modifies vec[0]

auto elem2 = vec[0];  // int, copy
elem2 = 42;  // Does NOT modify vec[0]
```

2. **Simpler Return Type Deduction**
```cpp
// C++11
template<typename F, typename... Args>
auto wrapper(F f, Args&&... args) -> decltype(f(std::forward<Args>(args)...)) {
    return f(std::forward<Args>(args)...);
}

// C++14 - much cleaner!
template<typename F, typename... Args>
decltype(auto) wrapper(F f, Args&&... args) {
    return f(std::forward<Args>(args)...);
}
```

3. **Variable Initialization**
```cpp
int x = 5;
int& rx = x;

decltype(auto) y = rx;   // y is int&
auto z = rx;              // z is int (copy)
```

### Return Type Rules in C++14

```cpp
decltype(auto) f1() { return 5; }        // Returns int
decltype(auto) f2() { int x = 5; return x; }   // Returns int
decltype(auto) f3() { int x = 5; return (x); } // Returns int& - DANGEROUS!
```

---

## Evolution in C++17

C++17 brought conceptual changes to how prvalues work, affecting `decltype` indirectly.

### Guaranteed Copy Elision

C++17 changed prvalues to be initialization expressions rather than temporary objects.

```cpp
struct S {
    S() { std::cout << "Constructor\n"; }
    S(const S&) { std::cout << "Copy\n"; }
};

S factory() { return S(); }

// C++14: Constructor, Copy (maybe elided)
// C++17: Constructor only (guaranteed)
S s = factory();

decltype(factory())  // Still S (prvalue), but semantic change
```

### Structured Bindings with decltype

C++17 introduced structured bindings, which work well with `decltype`:

```cpp
std::pair<int, double> getPair() {
    return {42, 3.14};
}

auto [i, d] = getPair();

decltype(i)  // int
decltype(d)  // double

// With references
auto& [ri, rd] = getPair();  // Error: can't bind to temporary

std::pair<int, double> p = getPair();
auto& [ri, rd] = p;  // OK
decltype(ri)  // int&
```

### Template Argument Deduction for Class Templates

```cpp
// C++17
std::pair p{1, 2.0};  // std::pair<int, double>
decltype(p)  // std::pair<int, double>

// Works with complex expressions
decltype(std::pair{1, 2.0})  // std::pair<int, double>
```

---

## Evolution in C++20

C++20 introduced concepts and constraints, which heavily use `decltype` in requires expressions.

### Requires Expressions

```cpp
#include <concepts>

template<typename T>
concept Addable = requires(T a, T b) {
    { a + b } -> std::same_as<T>;  // decltype((a + b)) must be T
};

template<typename T>
concept HasSize = requires(T t) {
    { t.size() } -> std::convertible_to<std::size_t>;
};
```

### Common Mistake in Requires Expressions

```cpp
template<typename TA, typename TB>
auto add(TA a, TB b)
    requires requires {
        { a + b } -> std::same_as<TA>;
        { b } -> std::same_as<int>;  // WRONG! decltype((b)) is int&, not int
    }
{
    return a += b;
}

// Correct version
template<typename TA, typename TB>
auto add(TA a, TB b)
    requires requires {
        { a + b } -> std::same_as<TA>;
        { b } -> std::same_as<int&>;  // Correct!
    }
{
    return a += b;
}
```

### decltype in Abbreviated Function Templates

```cpp
// C++20 abbreviated function template
void process(auto x) {
    using T = decltype(x);
    T copy = x;
    // ...
}

// Equivalent to:
template<typename T>
void process(T x) {
    T copy = x;
    // ...
}
```

### Concepts with decltype

```cpp
template<typename T>
concept Container = requires(T t) {
    typename T::value_type;
    { t.begin() } -> std::same_as<typename T::iterator>;
    { t.size() } -> std::same_as<typename T::size_type>;
};

template<Container C>
decltype(auto) getFirst(C& c) {
    return *c.begin();  // Preserves reference type
}
```

---

## Common Pitfalls

### Pitfall 1: Parentheses Matter!

```cpp
int x = 5;

decltype(x) a = x;    // int
decltype((x)) b = x;  // int&

// Dangerous in return statements
decltype(auto) bad() {
    int x = 42;
    return (x);  // Returns int& to local variable!
}
```

### Pitfall 2: Temporary Object Member Access

```cpp
struct S {
    int member = 0;
};

S f() { return S{}; }

decltype(f().member)  // int&& (xvalue)

// Dangerous!
decltype(auto) getMember() {
    return S{}.member;  // Returns int&& to destroyed temporary!
}
```

### Pitfall 3: Reference Collapsing Confusion

```cpp
int x = 5;
int& rx = x;

decltype(rx) y = x;     // int&
decltype((rx)) z = x;   // int& (parentheses don't add another reference)
```

### Pitfall 4: Conditional Operator Surprises

```cpp
int a = 1, b = 2;

decltype(a > b ? a : b)  // int& (both operands are lvalues)
decltype(true ? 0 : 1)   // int (both operands are prvalues)
decltype(a > b ? a : 0)  // int (mixed: unifies to prvalue)
```

---

## Best Practices

### 1. Use decltype(auto) for Perfect Return Type Forwarding

```cpp
// Good: Preserves exact return type
template<typename Func, typename... Args>
decltype(auto) invoke(Func&& f, Args&&... args) {
    return std::forward<Func>(f)(std::forward<Args>(args)...);
}
```

### 2. Avoid Parentheses in Return Statements

```cpp
// Bad
decltype(auto) bad(int x) {
    return (x);  // int& - dangerous!
}

// Good
decltype(auto) good(int x) {
    return x;    // int - safe
}
```

### 3. Use Macros for Safe decltype Usage

```cpp
// Prevent accidental expression decltype
#define exprtype(E) decltype((E))
#define vartype(v) decltype(v)

int x = 5;
vartype(x) y = 10;     // Clear intent: copy variable type
exprtype(x) z = x;     // Clear intent: get expression type (lvalue ref)
```

### 4. Prefer auto for Variable Declarations

```cpp
// Usually prefer this
auto x = someFunction();

// Use decltype(auto) only when you need to preserve references
decltype(auto) y = someFunction();  // If someFunction returns a reference
```

### 5. Use Trailing Return Types for Clarity

```cpp
// Clear and readable
template<typename T, typename U>
auto multiply(T a, U b) -> decltype(a * b) {
    return a * b;
}
```

### 6. Test Value Categories at Compile Time

```cpp
template<typename T> constexpr const char* category = "prvalue";
template<typename T> constexpr const char* category<T&> = "lvalue";
template<typename T> constexpr const char* category<T&&> = "xvalue";

#define SHOW_CATEGORY(E) \
    std::cout << #E << ": " << category<decltype((E))> << '\n'

int x = 5;
SHOW_CATEGORY(x);        // lvalue
SHOW_CATEGORY(x + 1);    // prvalue
SHOW_CATEGORY(std::move(x));  // xvalue
```

### 7. Document Intent with Type Aliases

```cpp
template<typename T>
using RemoveRef = std::remove_reference_t<T>;

template<typename Func>
auto wrapper(Func&& f) -> RemoveRef<decltype(f())> {
    return f();  // Always returns by value
}
```

---

## Summary Table

| Feature | C++11 | C++14 | C++17 | C++20 |
|---------|-------|-------|-------|-------|
| Basic `decltype` | ✓ | ✓ | ✓ | ✓ |
| `decltype(auto)` | ✗ | ✓ | ✓ | ✓ |
| Trailing return types | ✓ | ✓ | ✓ | ✓ |
| Guaranteed copy elision | ✗ | ✗ | ✓ | ✓ |
| Requires expressions | ✗ | ✗ | ✗ | ✓ |
| Abbreviated templates | ✗ | ✗ | ✗ | ✓ |

---

## Conclusion

`decltype` is a powerful feature that enables:
- **Type introspection** at compile time
- **Perfect forwarding** of return types
- **Generic programming** with exact type preservation
- **Metaprogramming** with type computations

Understanding the two forms of `decltype` (variable vs expression) and value categories is crucial for avoiding bugs. The evolution from C++11 through C++20 has made `decltype` progressively more powerful and easier to use, especially with `decltype(auto)` in C++14 and concepts in C++20.

Remember: **Parentheses matter!** `decltype(x)` and `decltype((x))` can be completely different types.
