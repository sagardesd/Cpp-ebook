# C++ Concepts: Constraining Templates (C++20)

## The Problem: Unclear Template Requirements

Consider this simple template function:

```cpp
template <typename T>
T min(const T& a, const T& b) {
    return a < b ? a : b;
}
```

**Question:** What must be true of type `T` for us to be able to use `min`?

**Answer:** `T` must have an `operator<` defined that returns something convertible to `bool`.

### When Things Go Wrong

```cpp
struct StudentId {
    std::string name;
    std::string id;
};

int main() {
    StudentId thomas { "Thomas", "S001" };
    StudentId rachel { "Rachel", "S002" };
    
    min<StudentId>(thomas, rachel);  // Compiler error!
}
```

### The Confusing Error Message

```
$ g++ main.cpp --std=c++20
main.cpp:9:12: error: invalid operands to binary expression
('const StudentId' and 'const StudentId')
    return a < b ? a : b;
           ~ ^ ~
main.cpp:20:3: note: in instantiation of function template
specialization 'min<StudentId>' requested here
    min<StudentId>(thomas, rachel);
    ^
1 error generated.
```

### What Happened? Understanding Template Instantiation

Here's the critical timeline:

**Step 1: You write the call**
```cpp
min<StudentId>(thomas, rachel);
```

**Step 2: Compiler sees the template**
```cpp
template <typename T>
T min(const T& a, const T& b) {
    return a < b ? a : b;
}
```

At this point, the compiler thinks: *"min for StudentIds, coming right up! The template looks fine, let me instantiate it..."*

**Step 3: Compiler instantiates the template** (creates a concrete function)
```cpp
StudentId min(const StudentId& a, const StudentId& b) {
    return a < b ? a : b;  // NOW it tries to compile this line
}
```

**Step 4: Compiler discovers the problem**
*"AHHH what do I do here! I don't know how to compare two StudentIds with <"*

### The Critical Problem: Late Error Detection

**The compiler CANNOT check if `StudentId` has `operator<` until it actually instantiates the template!**

Why? Because templates are NOT compiled when they're defined—they're only compiled when they're instantiated with specific types.

```cpp
// When you write this, the compiler does NOT check if T has operator<
template <typename T>
T min(const T& a, const T& b) {
    return a < b ? a : b;  // No error yet!
}

// The compiler only checks when you USE it with a specific type
min<StudentId>(thomas, rachel);  // NOW the error appears!
```

This creates several problems:

1. **Errors appear far from the actual mistake**
   - You made the mistake at the call site: `min<StudentId>(...)`
   - But the error points to line 9 inside the template definition: `return a < b ? a : b;`

2. **Confusing error messages**
   - The error talks about template internals, not your code
   - "in instantiation of function template specialization" - what does that even mean?

3. **No way to know requirements upfront**
   - How do you know `min` requires `operator<`? 
   - You have to read the implementation or documentation
   - The compiler can't help you until it's too late

4. **Bad templates can produce really confusing compiler errors**
   - Imagine a template with 50 lines of code
   - The error could be buried deep in that implementation
   - You see errors about code you didn't even write!

**Big Question:** How do we put constraints on templates so the compiler can check them BEFORE instantiation?

## The Solution: C++20 Concepts

### What is a Concept?

**A concept is a named set of constraints on template parameters introduced in C++20.**

In simple terms:
- A concept defines **requirements** that a type must satisfy
- It allows you to specify **what operations a type must support** to be used with a template
- The compiler checks these requirements **before instantiating** the template
- If the requirements aren't met, you get a clear error message at the call site

Think of concepts as "compile-time interfaces" or "type constraints" for templates.

### How Concepts Solve the Instantiation Problem

Concepts solve the instantiation problem by checking constraints **before** the compiler tries to instantiate the template.

### Key Benefit: Early Error Detection

**Without concepts:**
1. Compiler tries to instantiate `min<StudentId>`
2. Compiler generates the function body
3. Compiler tries to compile `a < b`
4. **Error discovered!** (too late)

**With concepts:**
1. Compiler checks: "Does `StudentId` satisfy `Comparable`?"
2. **Error discovered immediately!** (before instantiation)
3. Compiler never even tries to instantiate the template
4. You get a clear error at the call site

Concepts allow us to:
- **Check constraints BEFORE instantiation** (most important!)
- Be explicit about what we require of a template type
- Prevent template instantiation unless all constraints are met
- Get much better compiler error messages

### Concept Syntax

The general syntax for defining a concept is:

```cpp
template <typename T>
concept ConceptName = constraint_expression;
```

Where `constraint_expression` can be:
- A `requires` expression (most common)
- A conjunction of concepts using `&&`
- A disjunction of concepts using `||`
- A simple type trait like `std::is_integral_v<T>`

#### Requires Expression Syntax

```cpp
requires(parameter_list) {
    requirement1;
    requirement2;
    ...
}
```

**Types of requirements:**

1. **Simple requirement** - Expression must be valid
   ```cpp
   a + b;           // a + b must compile
   a.size();        // a must have a size() method
   ```

2. **Type requirement** - Type must exist
   ```cpp
   typename T::value_type;      // T must have a value_type member
   typename T::iterator;         // T must have an iterator member
   ```

3. **Compound requirement** - Expression must be valid and return specific type
   ```cpp
   { expression } -> concept<args>;
   { a < b } -> std::convertible_to<bool>;     // a < b must return bool-like
   { a.begin() } -> std::same_as<typename T::iterator>;
   ```

4. **Nested requirement** - Another constraint must be satisfied
   ```cpp
   requires std::is_copy_constructible_v<T>;
   ```

### Breaking Down the Comparable Concept

```cpp
template <typename T>
concept Comparable = requires(T a, T b) {
    { a < b } -> std::convertible_to<bool>;
};
```

Let's break this down:

```cpp
concept Comparable = ...
```
**Concept:** A named set of constraints

```cpp
requires(T a, T b) { ... }
```
**Requires clause:** "Given two T's, I expect the following to hold"

```cpp
{ a < b } -> std::convertible_to<bool>;
```
**Constraint 1:** Anything inside the `{ }` must compile without error (i.e., `a < b` must be valid)

**Constraint 2:** The result must be convertible to `bool` (note: `std::convertible_to` is itself a concept!)

### Using the Comparable Concept

There are two syntaxes for applying concepts to templates:

#### Syntax 1: Using `requires` clause

```cpp
template <typename T> requires Comparable<T>
T min(const T& a, const T& b) {
    return a < b ? a : b;
}
```

#### Syntax 2: Super slick shorthand (preferred)

```cpp
template <Comparable T>
T min(const T& a, const T& b) {
    return a < b ? a : b;
}
```

This reads naturally: "T must be Comparable"

### Concepts Greatly Improve Compiler Errors

Now when you try to use `min` with `StudentId`:

```cpp
template <Comparable T>
T min(const T& a, const T& b) {
    return a < b ? a : b;
}

StudentId thomas { "Thomas", "S001" };
StudentId rachel { "Rachel", "S002" };

min<StudentId>(thomas, rachel);  // Much clearer error!
```

**New error message:**
```
error: no matching function for call to 'min'
note: candidate template ignored: constraints not satisfied
note: because 'StudentId' does not satisfy 'Comparable'
```

Much better! The error now clearly states:
- The problem is at the **call site** (where you used it)
- `StudentId` doesn't satisfy the `Comparable` concept
- **No template instantiation attempted!**
- No confusing template instantiation details

### The Game Changer: Constraint Checking Before Instantiation

This is the crucial difference:

| Without Concepts | With Concepts |
|------------------|---------------|
| ❌ Try to instantiate template | ✅ Check constraints first |
| ❌ Generate function code | ✅ If constraints fail, STOP |
| ❌ Try to compile generated code | ✅ Never instantiate bad templates |
| ❌ Error deep in template code | ✅ Error at call site |
| ❌ "invalid operands to binary expression" | ✅ "does not satisfy Comparable" |

## More Concept Examples

### Example 1: Requiring Multiple Operations

```cpp
template <typename T>
concept Arithmetic = requires(T a, T b) {
    { a + b } -> std::convertible_to<T>;
    { a - b } -> std::convertible_to<T>;
    { a * b } -> std::convertible_to<T>;
    { a / b } -> std::convertible_to<T>;
};

template <Arithmetic T>
T average(const T& a, const T& b) {
    return (a + b) / T(2);
}
```

### Example 2: Requiring Member Functions

```cpp
template <typename T>
concept Printable = requires(T obj) {
    { obj.toString() } -> std::convertible_to<std::string>;
};

template <Printable T>
void display(const T& obj) {
    std::cout << obj.toString() << std::endl;
}
```

### Example 3: Requiring Type Members

```cpp
template <typename T>
concept Container = requires(T container) {
    typename T::value_type;           // Must have value_type member
    typename T::iterator;             // Must have iterator member
    { container.begin() } -> std::same_as<typename T::iterator>;
    { container.end() } -> std::same_as<typename T::iterator>;
    { container.size() } -> std::convertible_to<std::size_t>;
};

template <Container C>
void printSize(const C& container) {
    std::cout << "Size: " << container.size() << std::endl;
}
```

### Example 4: Combining Concepts

```cpp
template <typename T>
concept Sortable = Comparable<T> && std::copyable<T>;

template <Sortable T>
void sort(std::vector<T>& vec) {
    // Sort implementation
}
```

## Common Standard Library Concepts (C++20)

The STL provides many built-in concepts in `<concepts>`:


| Concept | Meaning |
|---------|---------|
| `std::same_as<T, U>` | T and U are the same type |
| `std::convertible_to<From, To>` | From is convertible to To |
| `std::integral<T>` | T is an integral type |
| `std::floating_point<T>` | T is a floating point type |
| `std::copyable<T>` | T can be copied |
| `std::movable<T>` | T can be moved |
| `std::default_initializable<T>` | T can be default constructed |

All the available builtin concepts can be found here: https://en.cppreference.com/w/cpp/concepts.html

## Concepts with Iterators

```cpp
template <typename It, typename T>
concept SearchableIterator = requires(It it, T value) {
    { *it } -> std::convertible_to<T>;  // Can dereference
    { ++it } -> std::same_as<It&>;      // Can increment
    { it != it } -> std::convertible_to<bool>;  // Can compare
};

template <SearchableIterator<T> It, typename T>
It find(It begin, It end, const T& value) {
    for (It it = begin; it != end; ++it) {
        if (*it == value) {
            return it;
        }
    }
    return end;
}
```

## Concepts Recap

### Two Main Reasons to Use Concepts

1. **Better compiler error messages**
   - Errors caught at the constraint level, not deep in template code
   - Clear indication of which requirements aren't met
   - Errors appear at the call site where they're most useful

2. **Better IDE support**
   - Improved Intellisense/autocomplete
   - IDEs can show which types satisfy which concepts
   - Better code navigation and refactoring

### Current Limitations

- Concepts are still a relatively new feature (C++20)
- The STL does not yet support them fully across all libraries
- Many older codebases still use older constraint techniques (SFINAE, `std::enable_if`)
- Compiler support is still maturing

## Quick Reference: Concept Syntax

```cpp
// Define a concept
template <typename T>
concept ConceptName = requires(T obj) {
    // constraints go here
};

// Use concept - Method 1
template <typename T> requires ConceptName<T>
void function(T param);

// Use concept - Method 2 (preferred)
template <ConceptName T>
void function(T param);

// Use concept with auto parameters (C++20)
void function(ConceptName auto param);
```

## Summary

| Before Concepts | With Concepts |
|----------------|---------------|
| Template errors deep in instantiation | Errors at call site |
| Unclear requirements | Explicit, named requirements |
| Cryptic error messages | Clear, understandable errors |
| No IDE help | Better IDE support |
| Requirements in documentation only | Requirements in code |

**Note:** Concepts make templates safer, clearer, and much easier to use correctly!
