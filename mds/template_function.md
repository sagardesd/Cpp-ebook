# C++ Templates Tutorial

## Table of Contents

1. [What Are Templates?](#what-are-templates)
2. [Visualize the Problem](#visualize-the-problem)
3. [Function Templates](#function-templates)
4. [How Function Templates Work](#how-function-templates-work)
5. [How to Call Template Functions](#how-to-call-template-functions)
   - [Option 1: Implicit Instantiation](#option-1-implicit-instantiation)
   - [Option 2: Explicit Instantiation](#option-2-explicit-instantiation)
6. [Templates vs Functions](#templates-vs-functions)
7. [Key Takeaways and Summary](#key-takeaways-and-summary)

---

## What Are Templates?

**Templates are C++'s way of writing generic code that can work on any data type (built-in and user-defined).**

In essence, templates automate code generation. You write the function logic once, and the compiler generates the necessary versions for each data type you use.

[↑ Back to Table of Contents](#table-of-contents)

---

## Visualize the Problem

Consider a function that returns the smallest of two values, let's say two integers:

```cpp
int min(int a, int b) { 
    return a < b ? a : b; 
}
```

The `min` function makes sense for more than just integers. What if we want to find the smallest of two doubles, or two strings?

```cpp
min(106, 107);           // int, returns 106
min(1.2, 3.4);           // double, returns 1.2
min("Thomas", "Rachel"); // string, returns "Rachel" (alphabetically first)
```

### Attempted Solution: Function Overloading

```cpp
int min(int a, int b) { 
    return a < b ? a : b; 
}

double min(double a, double b) { 
    return a < b ? a : b; 
}

std::string min(std::string a, std::string b) { 
    return a < b ? a : b; 
}
```

**Problem:** The function logic doesn't change, but we keep duplicating code. What if we need to support more types in the future, including custom classes? We can't keep adding functions manually.

**This problem can be solved using function templates.**

[↑ Back to Table of Contents](#table-of-contents)

---

## Function Templates

Templates let us write the function once and let the compiler generate the necessary versions automatically:

```cpp
template <typename T>
T min(T a, T b) { 
    return a < b ? a : b; 
}
```

### Syntax

```cpp
template <typename T>  // or template <class T>
return-type functionName(T parameter1, T parameter2, ...) {
    // Function logic
}
```

**Note:** `typename` and `class` are interchangeable in template declarations. Using `typename` is generally preferred and will make more sense when exploring advanced C++20 features.

[↑ Back to Table of Contents](#table-of-contents)

---

## How Function Templates Work

When you call a template function, the compiler generates the specific version of the code for the type you're using:

```cpp
min(106, 107);   // Compiler generates: int min(int a, int b)
min(1.2, 3.4);   // Compiler generates: double min(double a, double b)
```

Behind the scenes, the compiler generates:

```cpp
// Compiler-generated code
int min(int a, int b) { 
    return a < b ? a : b; 
}

double min(double a, double b) { 
    return a < b ? a : b; 
}
```

This happens at **compile-time**, so there's no runtime overhead.

[↑ Back to Table of Contents](#table-of-contents)

---

## How to Call Template Functions

### Option 1: Implicit Instantiation

Let the compiler infer the types automatically:

```cpp
min(106, 107);   // int, returns 106
min(1.2, 3.4);   // double, returns 1.2
```

**Advantage:** Clean, concise syntax.

**Disadvantage:** Can lead to unexpected behavior in ambiguous cases.

#### Problem 1: String Literals

```cpp
template <typename T>
T min(T a, T b) { 
    return a < b ? a : b; 
}

min("Thomas", "Rachel");  // Dangerous!
```

String literals (`"Thomas"`, `"Rachel"`) are passed as `const char*`, so the compiler generates:

```cpp
const char* min(const char* a, const char* b) { 
    return a < b ? a : b; 
}
```

**Problem:** This performs pointer comparison, not string comparison! ❌

#### Problem 2: Mismatched Parameter Types

```cpp
min(106, 3.14);  // int and double - doesn't compile!
```

Since the parameters are different types (`int` and `double`), the compiler cannot deduce a single type `T`. Implicit instantiation fails.

[↑ Back to Table of Contents](#table-of-contents)

---

### Option 2: Explicit Instantiation

Explicitly specify the type to avoid ambiguity:

```cpp
min<std::string>("Thomas", "Rachel");  // Specify type explicitly
min<double>(106, 3.14);                // Specify type explicitly
```

#### Solution to Problem 1: String Comparison

```cpp
template <typename T>
T min(const T& a, const T& b) { 
    return a < b ? a : b; 
}

min<std::string>("Thomas", "Rachel");  // ✅ Correct!
```

Here, `const char*` is converted to `std::string`, giving us proper string comparison.

#### Solution to Problem 2: Mismatched Types

When parameters have different types, use explicit instantiation to specify which type to use:

```cpp
min<double>(106, 3.14);  // ✅ Converts 106 to double, returns 3.14
min<int>(106, 3.14);     // ✅ Converts 3.14 to int, returns 3
```

The compiler will perform necessary type conversions based on your explicit type specification.

[↑ Back to Table of Contents](#table-of-contents)

---

## Templates vs Functions

It's important to understand the distinction:

| Concept | What It Is |
|---------|------------|
| `template<typename T> T min(T a, T b)` | **This is a TEMPLATE** - Not a function, but a blueprint for generating functions |
| `min<int>` | **This is a FUNCTION** - Also known as a template instantiation |

**Key Point:** When you write a template, you're creating a pattern. When the compiler instantiates it with a specific type (like `min<int>`), that's when an actual function is generated.

[↑ Back to Table of Contents](#table-of-contents)

---

## Key Takeaways and Summary

### Key Takeaways

- **Templates automate code generation** - write once, use for any type
- **Implicit instantiation** is convenient but can be ambiguous
- **Explicit instantiation** gives you control when types don't match exactly
- Templates are resolved at **compile-time** with no runtime overhead
- Works with both built-in types and user-defined classes

### Summary

Templates allow you to:

1. **Define behavior once** - Write the logic a single time
2. **Let the compiler generate type-specific implementations** - Automatic code generation
3. **Write generic, reusable code** - Works with any type
4. **Maintain type safety without code duplication** - No manual overloading needed

**Think of it as:** You provide the blueprint (template), the compiler builds the specific versions you need!

[↑ Back to Table of Contents](#table-of-contents)