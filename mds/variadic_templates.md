# Variadic Templates

## Table of Contents
1. [The Problem: Variable Number of Arguments](#the-problem-variable-number-of-arguments)
2. [Solution 1: Manual Function Overloading](#solution-1-manual-function-overloading)
3. [Solution 2: Using std::vector](#solution-2-using-stdvector)
4. [Variadic Templates](#solution-3-variadic-templates)
    - [What is a Variadic Template?](#what-is-a-variadic-template)
    - [Basic Syntax](#basic-syntax)
    - [Complete Example: Variadic min](#complete-example-variadic-min)
    - [How It Works](#how-it-works)
5. [Parameter Packs Deep Dive](#parameter-packs-deep-dive)
    - [Template Parameter Packs](#template-parameter-packs)
    - [Function Parameter Packs](#function-parameter-packs)
    - [Pack Expansion](#pack-expansion)
6. [Modern C++17: Fold Expressions](#modern-c17-fold-expressions)
7. [Common Variadic Patterns](#common-variadic-patterns)
8. [Summary](#summary)

---

# Variadic Templates: From Problem to Solution

## The Problem: Variable Number of Arguments

Suppose we want to write a `min` function that finds the minimum of any number of values:
The following example is taken from the `Concepts` chapter.
Here in this example `Comparable` is a concept, so dont get confused.

```cpp
template <Comparable T>
T min(const T& a, const T& b) {
    return a < b ? a : b;
}

min(2.4, 7.5);              // ✓ This works
min(2.4, 7.5, 5.3);         // ✗ ERROR: No matching function
min(2.4, 7.5, 5.3, 1.2);    // ✗ ERROR: No matching function
```

How do we make `min` accept a variable number of parameters?

## Solution 1: Function Overloading (The Manual Way)

We could manually write overloads for different numbers of parameters:

```cpp
// 2 parameters
template <Comparable T>
T min(const T& a, const T& b) { 
    return a < b ? a : b; 
}

// 3 parameters
template <Comparable T>
T min(const T& a, const T& b, const T& c) {
    auto m = min(b, c);          // Calls 2-parameter version
    return a < m ? a : m;
}

// 4 parameters
template <Comparable T>
T min(const T& a, const T& b, const T& c, const T& d) {
    auto m = min(b, c, d);       // Calls 3-parameter version
    return a < m ? a : m;
}
```

**Results:**
```cpp
min(2.4, 7.5);              // ✓ Works
min(2.4, 7.5, 5.3);         // ✓ Works now
min(2.4, 7.5, 5.3, 1.2);    // ✓ Works too!
min(2.4, 7.5, 5.3, 1.2, 3.4, 6.7, 8.9, 9.1); // ✗ Need to write more overloads...
```

### Problems with This Approach

**Tedious**: Need to write many overloads manually  
**Limited**: Only works up to the number of overloads you write  
**Not scalable**: What if someone needs 10 or 20 parameters?  
**Repetitive**: Notice the pattern? The compiler should handle this!

## Solution 2: Using std::vector (The Dynamic Way)

Can we use a vector to hold variable numbers of arguments?

```cpp
template <Comparable T>
T min(const std::vector<T>& values) {
    if (values.size() == 1) return values[0];
    
    const auto& first = values[0];
    std::vector<T> rest(++values.begin(), values.end());
    auto m = min(rest);              // Recursive call
    return first < m ? first : m;
}

// Usage with brace initialization
min({2.4, 7.5});
min({2.4, 7.5, 5.3});
min({2.4, 7.5, 5.3, 1.2});
```

### Problems with This Approach

**Runtime overhead**: Must allocate a vector for every call  
**Recursive copying**: Each recursive call copies the remaining elements  
**Memory allocation**: Dynamic memory allocation is expensive  
**Awkward syntax**: Requires braces `{}` around arguments  
**No compile-time optimization**: Cannot be fully optimized away

**Key insight**: We need the compiler to generate the code at compile-time, not handle it at runtime!

So how can we ask compiler to generate these underlying functions for us ?
Here comes the Variadic Templates.

## Variadic Templates ✨

### Enter C++11: A Game-Changing Feature

C++11 introduced **variadic templates**, a powerful feature that allows templates to accept a variable number of arguments. Instead of manually writing overloads or relying on runtime containers, variadic templates let the **compiler** automatically generate all the code we need at compile-time!

### What is a Variadic Template?

A **variadic template** is a template that can accept any number (zero or more) of template arguments. It uses a special construct called a **parameter pack** to capture these arguments.

**Definition:**
> A variadic template uses parameter packs (`...`) to accept and work with a variable number of types or values, enabling type-safe, compile-time generation of code for any number of arguments.

### Basic Syntax

```cpp
// Template with parameter pack
template <typename... Args>
//                  ^^^^^^
//                  Parameter pack (captures 0 or more types)
void function(Args... args) {
//            ^^^^^^^^^^^^
//            Function parameter pack (captures 0 or more values)
    // Use args... here
}
```

**Key syntax elements:**
- `typename... Args` - Declares a **template parameter pack** (types)
- `Args... args` - Declares a **function parameter pack** (values)
- `args...` - **Expands** the parameter pack

### The Complete Solution

```cpp
// Base case: single value (stops recursion)
template <Comparable T>
T min(const T& value) { 
    return value; 
}

// Recursive case: 2 or more values (variadic template)
template <Comparable T, Comparable... Args>
//                      ^^^^^^^^^^^^^^^^^^
//                      Parameter pack: accepts 0+ types
T min(const T& first, const Args&... rest) {
//                    ^^^^^^^^^^^^^^^^^
//                    Function parameter pack: 0+ arguments
    auto min_rest = min(rest...);     // Recursive call with remaining args
//                      ^^^^^^^       // Pack expansion: expands rest...
    return first < min_rest ? first : min_rest;
}
```

**Usage:**
```cpp
min(2.4, 7.5);                    // ✓ Works!
min(2.4, 7.5, 5.3);               // ✓ Works!
min(2.4, 7.5, 5.3, 1.2);          // ✓ Works!
min(2.4, 7.5, 5.3, 1.2, 3.4);     // ✓ Works!
// Works with ANY number of arguments!
```

### How It Works: Step-by-Step

Let's trace `min(5, 2, 8, 1)` and see what template functions the compiler generates:

```
═══════════════════════════════════════════════════════════════════════════════
                    COMPILER GENERATES THESE FUNCTIONS FOR US!
═══════════════════════════════════════════════════════════════════════════════

╔═══════════════════════════════════════════════════════════════════════════╗
║ CALL 1: min(5, 2, 8, 1)                                                   ║
║                                                                           ║
║ Template Deduction:  T = int,  Args = [int, int, int]                     ║
║ ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓   ║
║ ┃ GENERATED FUNCTION:                                                 ┃   ║
║ ┃ int min(const int& v, const int& a0, const int& a1, const int& a2) {┃   ║
║ ┃     auto m = min(a0, a1, a2);  // Calls next instantiation          ┃   ║
║ ┃     return v < m ? v : m;                                           ┃   ║
║ ┃ }                                                                   ┃   ║
║ ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛   ║
║                                                                           ║
║ Runtime Values:  v=5, a0=2, a1=8, a2=1                                    ║
║ Calls: min(2, 8, 1)  ────────────────────────────────────────────────────→║
╚═══════════════════════════════════════════════════════════════════════════╝
                                      ↓

╔═══════════════════════════════════════════════════════════════════════════╗
║ CALL 2: min(2, 8, 1)                                                      ║
║                                                                           ║
║ Template Deduction:  T = int,  Args = [int, int]                          ║
║ ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓           ║
║ ┃ GENERATED FUNCTION:                                         ┃           ║
║ ┃ int min(const int& v, const int& a0, const int& a1) {       ┃           ║
║ ┃     auto m = min(a0, a1);  // Calls next instantiation      ┃           ║
║ ┃     return v < m ? v : m;                                   ┃           ║
║ ┃ }                                                           ┃           ║
║ ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛           ║
║                                                                           ║
║ Runtime Values:  v=2, a0=8, a1=1                                          ║
║ Calls: min(8, 1)  ──────────────────────────────────────────────────────→ ║
╚═══════════════════════════════════════════════════════════════════════════╝
                                      ↓

╔═══════════════════════════════════════════════════════════════════════════╗
║ CALL 3: min(8, 1)                                                         ║
║                                                                           ║
║ Template Deduction:  T = int,  Args = [int]                               ║
║ ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓           ║
║ ┃ GENERATED FUNCTION:                                         ┃           ║
║ ┃ int min(const int& v, const int& a0) {                      ┃           ║
║ ┃     auto m = min(a0);  // Calls base case                   ┃           ║
║ ┃     return v < m ? v : m;                                   ┃           ║
║ ┃ }                                                           ┃           ║
║ ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛           ║
║                                                                           ║
║ Runtime Values:  v=8, a0=1                                                ║
║ Calls: min(1)  ─────────────────────────────────────────────────────────→ ║
╚═══════════════════════════════════════════════════════════════════════════╝
                                      ↓

╔═══════════════════════════════════════════════════════════════════════════╗
║ CALL 4: min(1)                                     ★ BASE CASE ★          ║
║                                                                           ║
║ Template Deduction:  T = int,  Args = [] (empty!)                         ║
║ ┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓           ║
║ ┃ MATCHED BASE CASE FUNCTION:                                 ┃           ║
║ ┃ int min(const int& v) {                                     ┃           ║
║ ┃     return v;  // No recursion!                             ┃           ║
║ ┃ }                                                           ┃           ║
║ ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛           ║
║                                                                           ║
║ Runtime Values:  v=1                                                      ║
║ Returns: 1  ← Recursion stops!                                            ║
╚═══════════════════════════════════════════════════════════════════════════╝

═══════════════════════════════════════════════════════════════════════════════
                         RETURN VALUES (Unwinding)
═══════════════════════════════════════════════════════════════════════════════

                     min(1) returns → 1
                              ↑
┌───────────────────────────────────────────────────────────────────┐
│ min(8, 1):  m=1,  return 8 < 1 ? 8 : 1  →  returns 1              │
└───────────────────────────────────────────────────────────────────┘
                              ↑
┌───────────────────────────────────────────────────────────────────┐
│ min(2, 8, 1):  m=1,  return 2 < 1 ? 2 : 1  →  returns 1           │
└───────────────────────────────────────────────────────────────────┘
                              ↑
┌───────────────────────────────────────────────────────────────────┐
│ min(5, 2, 8, 1):  m=1,  return 5 < 1 ? 5 : 1  →  returns 1        │
└───────────────────────────────────────────────────────────────────┘

═══════════════════════════════════════════════════════════════════════════════
                             FINAL RESULT: 1 ✓
═══════════════════════════════════════════════════════════════════════════════

💡 KEY INSIGHT: The compiler generated 4 complete functions for us!
   ✓ min(int, int, int, int) - 4 parameters
   ✓ min(int, int, int)      - 3 parameters  
   ✓ min(int, int)           - 2 parameters
   ✓ min(int)                - 1 parameter (base case)

   All at COMPILE TIME with ZERO runtime overhead!
═══════════════════════════════════════════════════════════════════════════════
```

## Understanding the Syntax: Parameter Packs Deep Dive

### What is a Parameter Pack?

A **parameter pack** is a template parameter that accepts zero or more template arguments. Think of it as a compile-time container that holds a variable number of elements (types or values).

There are three types of parameter packs:

1. **Template Parameter Pack** - holds types
2. **Function Parameter Pack** - holds function arguments (values)
3. **Template Template Parameter Pack** - holds template templates

### 1. Template Parameter Pack Declaration

```cpp
template <Comparable T, Comparable... Args>
//                      ^^^^^^^^^^^^^^^^^
//                      Template parameter pack
```

**Syntax breakdown:**
- `...Args` declares a **template parameter pack** named `Args`
- The `...` comes **before** the identifier when capturing
- It can match **zero or more** types
- Each type must satisfy the `Comparable` concept

**Examples:**
```cpp
// Type parameter pack
template<typename... Types>
struct Container {};

Container<int, double, char> c1;        // Types = [int, double, char]
Container<std::string> c2;              // Types = [std::string]
Container<> c3;                         // Types = [] (empty!)

// Non-type parameter pack
template<int... Values>
struct IntList {};

IntList<1, 2, 3, 4> list1;              // Values = [1, 2, 3, 4]
IntList<42> list2;                       // Values = [42]
IntList<> list3;                         // Values = [] (empty!)

// Template template parameter pack
template<template<typename> typename... Templates>
struct TemplateList {};

TemplateList<std::vector, std::list, std::deque> tl;
```

### 2. Function Parameter Pack Declaration

```cpp
T min(const T& first, const Args&... rest)
//                    ^^^^^^^^^^^^^^^^^
//                    Function parameter pack
```

**Syntax breakdown:**
- `const Args&... rest` declares a function parameter pack named `rest`
- The `...` comes **before** the identifier when capturing
- `Args` is expanded first (it's a template parameter pack)
- Then `&...` creates references to each expanded type
- Finally, `rest` names the entire pack

**What this expands to:**

If `Args = [int, double, char]`, then:
```cpp
const Args&... rest
    ↓
const int& r0, const double& r1, const char& r2
```

**More examples:**
```cpp
// By value
void func1(Types... args);              // Takes copies
// Expands to: Type1 arg1, Type2 arg2, ...

// By const reference
void func2(const Types&... args);       // Takes const refs
// Expands to: const Type1& arg1, const Type2& arg2, ...

// By forwarding reference (perfect forwarding)
void func3(Types&&... args);            // Universal references
// Expands to: Type1&& arg1, Type2&& arg2, ...
```

### 3. Pack Expansion: The Magic Happens

Pack expansion is where the compiler replaces the pattern with actual elements. The `...` comes **after** the pattern when expanding.

```cpp
auto min_rest = min(rest...);
//                  ^^^^^^^^
//                  Pack expansion
```

**How it works:**

If `rest` contains `[a, b, c]`, then:
```cpp
min(rest...)
    ↓
min(a, b, c)
```

### Pack Expansion Contexts

Parameter packs can be expanded in many contexts:

#### A. Function Call Arguments
```cpp
template<typename... Args>
void forward_to_func(Args&&... args) {
    some_function(std::forward<Args>(args)...);
    //            ^^^^^^^^^^^^^^^^^^^^^^^^^^^^
    //            Pattern: std::forward<Args>(args)
    //            Expands to: std::forward<Arg1>(arg1), std::forward<Arg2>(arg2), ...
}
```

#### B. Initializer Lists
```cpp
template<typename... Args>
void example(Args... args) {
    std::vector<int> vec{args...};           // Expands to: {arg1, arg2, arg3, ...}
    auto tuple = std::tuple{args...};        // Expands to: tuple{arg1, arg2, arg3, ...}
    int array[] = {args...};                 // Expands to: {arg1, arg2, arg3, ...}
}
```

#### C. Template Arguments
```cpp
template<typename... Args>
void example(Args... args) {
    // Pattern: decltype(args)
    std::tuple<decltype(args)...> tpl;       
    // Expands to: std::tuple<decltype(arg1), decltype(arg2), ...>
    
    // Pattern: std::vector<Args>
    std::tuple<std::vector<Args>...> vec_tuple;
    // Expands to: std::tuple<std::vector<Arg1>, std::vector<Arg2>, ...>
}
```

#### D. Base Class Lists
```cpp
template<typename... Bases>
struct MultiInherit : Bases... {
    //                ^^^^^^^^^
    //                Expands to: Base1, Base2, Base3, ...
    
    using Bases::foo...;  // Bring all foo() methods into scope
    //            ^^^^^
    //            Expands to: using Base1::foo; using Base2::foo; ...
};
```

#### E. Lambda Captures
```cpp
template<typename... Args>
void example(Args... args) {
    // Capture by copy
    auto lambda1 = [args...] { return process(args...); };
    //             ^^^^^^^^                   ^^^^^^^^
    //             Capture pack                Expand pack
    
    // Capture by reference
    auto lambda2 = [&args...] { return process(args...); };
    
    // Init-capture with move (C++20)
    auto lambda3 = [...args = std::move(args)] { 
        return process(args...); 
    };
}
```

### Complex Pack Expansion Patterns

The pattern can be arbitrarily complex:

```cpp
template<typename... Args>
void complex_example(Args... args) {
    // Simple pattern: just the pack
    func(args...);
    // Expands to: func(arg1, arg2, arg3, ...)
    
    // Pattern with function call
    func(transform(args)...);
    // Expands to: func(transform(arg1), transform(arg2), ...)
    
    // Pattern with template
    func(std::make_unique<Args>(args)...);
    // Expands to: func(std::make_unique<Arg1>(arg1), 
    //                  std::make_unique<Arg2>(arg2), ...)
    
    // Pattern with multiple operations
    func((args * 2 + 1)...);
    // Expands to: func((arg1 * 2 + 1), (arg2 * 2 + 1), ...)
}
```

### Multiple Parameter Packs in One Expression

When expanding multiple packs simultaneously, they must have the **same length**:

```cpp
template<typename... Ts, typename... Us>
void zip(Ts... ts, Us... us) {
    // Both packs must have same length
    auto pairs = std::tuple{std::pair(ts, us)...};
    // Expands to: std::tuple{std::pair(t1, u1), std::pair(t2, u2), ...}
}

zip(1, 2, 3, "a", "b", "c");  // OK: both have 3 elements
zip(1, 2, "a");               // ERROR: first has 2, second has 1
```

### sizeof...() Operator

Get the number of elements in a parameter pack:

```cpp
template<typename... Args>
void example(Args... args) {
    constexpr size_t type_count = sizeof...(Args);   // Number of types
    constexpr size_t arg_count = sizeof...(args);    // Number of arguments
    
    static_assert(sizeof...(Args) == sizeof...(args)); // Always true
    
    std::cout << "Received " << sizeof...(args) << " arguments\n";
}

example(1, 2.5, "hello");  // Prints: Received 3 arguments
```

### Nested Pack Expansion

Packs can be expanded inside other packs:

```cpp
template<typename... Outer>
void nested(Outer... outer) {
    // Inner pack expansion inside outer pack expansion
    auto result = std::tuple{
        std::vector{outer, outer, outer}...
        // For each outer element, create a vector with 3 copies
    };
    
    // If outer = [1, 2, 3], creates:
    // std::tuple{std::vector{1, 1, 1}, 
    //            std::vector{2, 2, 2}, 
    //            std::vector{3, 3, 3}}
}
```

### Pack Expansion in sizeof

```cpp
template<typename... Args>
void example(Args... args) {
    // Get total size of all arguments
    size_t total_size = (sizeof(args) + ...);  // Fold expression
    
    // Or create array of sizes
    size_t sizes[] = {sizeof(args)...};
    // Expands to: {sizeof(arg1), sizeof(arg2), sizeof(arg3), ...}
}
```

### Key Rules for Pack Expansion

1. **Ellipsis position matters:**
   - `...Name` = **capture** a pack
   - `Pattern...` = **expand** a pack

2. **Expansion must be in valid context** (see contexts above)

3. **Cannot expand outside valid context:**
   ```cpp
   // ERROR: Can't expand in arbitrary expression
   template<typename... Args>
   void bad(Args... args) {
       int x = args...;  // ERROR! Not a valid expansion context
   }
   ```

4. **Multiple packs in one expansion must have same length**

5. **Empty packs are valid:**
   ```cpp
   template<typename... Args>
   void func(Args... args) {}
   
   func();  // OK! Args and args are both empty
   ```

### Visual Summary: Capture vs Expansion

```cpp
template<typename... Args>              // Capture template pack
void example(Args... args) {            // Capture function pack
    //       ^^^^^^^^^^^^
    //       This is expansion! Args... becomes Arg1 arg1, Arg2 arg2, ...
    
    func(args...);                      // Expand pack
    //   ^^^^^^^
    //   Pattern = args, Expansion = args...
    
    func((args + 1)...);                // Expand with pattern
    //   ^^^^^^^^^^^^^
    //   Pattern = (args + 1), Expansion = (args + 1)...
}
```

**Remember:** 
- Ellipsis **before** = Capture (`...name`)
- Ellipsis **after** = Expand (`pattern...`)

## The Pattern: Base Case + Recursive Case

Variadic templates typically follow this pattern:

```cpp
// BASE CASE: Handles the "stop condition"
template <typename T>
ReturnType function(T value) {
    // Handle single value
    return /* something */;
}

// RECURSIVE CASE: Handles 2+ arguments
template <typename T, typename... Args>
ReturnType function(T first, Args... rest) {
    auto result = function(rest...);  // Recurse with remaining args
    // Combine first with result
    return /* combined result */;
}
```

## More Examples

### Example 1: Print All Arguments

```cpp
// Base case
void print() {
    std::cout << std::endl;
}

// Recursive case
template <typename T, typename... Args>
void print(const T& first, const Args&... rest) {
    std::cout << first << " ";
    print(rest...);  // Recursive call
}

// Usage
print(1, 2.5, "hello", 'x');
// Output: 1 2.5 hello x
```

### Example 2: Sum All Arguments

```cpp
// Base case
template <typename T>
T sum(T value) {
    return value;
}

// Recursive case
template <typename T, typename... Args>
T sum(T first, Args... rest) {
    return first + sum(rest...);
}

// Usage
auto result = sum(1, 2, 3, 4, 5);  // result = 15
```

### Example 3: Check All Conditions

```cpp
// Base case
bool all_true(bool value) {
    return value;
}

// Recursive case
template <typename... Args>
bool all_true(bool first, Args... rest) {
    return first && all_true(rest...);
}

// Usage
bool result = all_true(true, true, false, true);  // result = false
```

## Modern C++17 Alternative: Fold Expressions

C++17 introduced **fold expressions**, which provide a more concise syntax:

```cpp
// Using fold expression (C++17)
template <Comparable T, Comparable... Args>
T min(T first, Args... rest) {
    return (first < ... < rest) ? first : min(rest...);
}

// Even simpler with fold
template <typename... Args>
void print(const Args&... args) {
    ((std::cout << args << " "), ...);  // Fold over comma operator
    std::cout << std::endl;
}

template <typename... Args>
auto sum(Args... args) {
    return (... + args);  // Fold over + operator
}

template <typename... Args>
bool all_true(Args... args) {
    return (... && args);  // Fold over && operator
}
```
Fold expression will be covered in a separate section in more details.

## Comparison: All Three Approaches

| Approach | Code Size | Runtime Cost | Flexibility | Syntax |
|----------|-----------|--------------|-------------|---------|
| **Manual Overloading** | Large (N functions for N args) | None | Limited | Simple |
| **Vector** | Small | High (allocation, copying) | Unlimited | Awkward braces |
| **Variadic Templates** | Generated at compile-time | None (fully inlined) | Unlimited | Clean |

## Key Advantages of Variadic Templates

**Zero runtime overhead**: All code generated at compile-time  
**Type-safe**: Compiler checks all types  
**Unlimited flexibility**: Works with any number of arguments  
**Clean syntax**: No braces or wrappers needed  
**Fully optimizable**: Compiler can inline everything  
**Compile-time errors**: Problems caught during compilation  

## Common Patterns

### Pattern 1: Process First, Recurse on Rest

```cpp
template <typename T>
void process_all(T value) {
    process(value);  // Base case
}

template <typename T, typename... Args>
void process_all(T first, Args... rest) {
    process(first);        // Process first
    process_all(rest...);  // Recurse on rest
}
```

### Pattern 2: Accumulate Result

```cpp
template <typename T>
T accumulate(T value) {
    return value;  // Base case
}

template <typename T, typename... Args>
T accumulate(T first, Args... rest) {
    return combine(first, accumulate(rest...));  // Combine with result
}
```

### Pattern 3: Check All Elements

```cpp
template <typename T>
bool check_all(T value) {
    return check(value);  // Base case
}

template <typename T, typename... Args>
bool check_all(T first, Args... rest) {
    return check(first) && check_all(rest...);  // Short-circuit on false
}
```

## Important Notes

### 1. Base Case is Essential

Without a base case, recursion never stops:

```cpp
// WRONG: No base case!
template <typename T, typename... Args>
void print(T first, Args... rest) {
    std::cout << first << " ";
    print(rest...);  // Infinite recursion when rest is empty!
}
```

### 2. Parameter Packs Must Be Last

```cpp
// CORRECT
template <typename First, typename... Rest>
void func(First f, Rest... r);

// WRONG: Can't have parameters after pack
template <typename... Args, typename Last>  // ERROR!
void func(Args... args, Last l);
```

### 3. Empty Packs Are Valid

```cpp
template <typename... Args>
void func(Args... args) {
    std::cout << "Number of args: " << sizeof...(args) << "\n";
}

func();        // Valid! sizeof...(args) = 0
func(1);       // sizeof...(args) = 1
func(1, 2, 3); // sizeof...(args) = 3
```

## Summary

**Variadic templates** solve the problem of writing functions that accept a variable number of arguments:

- **Syntax**: `template <typename... Args>` for parameter packs
- **Pattern**: Base case + recursive case
- **Expansion**: `args...` expands the pack
- **Benefits**: Zero runtime cost, type-safe, unlimited flexibility
- **Modern C++17**: Fold expressions provide even more concise syntax

Variadic templates are a powerful tool that combines:
- The flexibility of runtime solutions (like vectors)
- The performance of compile-time code generation
- The elegance of recursive algorithms

They're essential for modern C++ generic programming!


