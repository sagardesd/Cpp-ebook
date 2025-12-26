# From Specific to General: A Guide to Predicates, Functors, and Lambda Functions in C++

## Starting Point: A Specific Find Algorithm

Let's begin with a simple `find` algorithm that searches for a specific value:

```cpp
template <typename It, typename T>
It find(It first, It last, const T& value) {
    for (auto it = first; it != last; ++it) {
        if (*it == value)  // This condition is too specific!
            return it;
    }
    return last;
}
```

This works well for finding exact values.
The line of code that does this is:
```cpp
if (*it == value)
```
What if i want to find the 1st element that is a prime number or based on some different criteria ?

The condition `*it == value` is too restrictive. 

Instead of hardcoding the comparison, what if we could pass the condition itself as a parameter?

Let's replace the specific condition with a general **predicate** function:

```cpp
template <typename It, typename Pred>
It find_if(It first, It last, Pred pred) {
    for (auto it = first; it != last; ++it) {
        if (pred(*it))  // Call the predicate on each element
            return it;
    }
    return last;
}
```

**What changed?**
- `Pred`: The type of our predicate (the compiler figures this out via template deduction)
- `pred`: Our predicate parameter - a function we can call on each element
- `pred(*it)`: We call the predicate to test each element

Lets rename it to `find_if` to distinguish it from the original `find` function.


Finding Prime Numbers `predicate` function.

```cpp
bool isPrime(size_t n) {
    if (n < 2) return false;
    for (size_t i = 2; i <= std::sqrt(n); i++)
        if (n % i == 0) return false;
    return true;
}

std::vector<int> ints = {1, 0, 6};
auto it = find_if(ints.begin(), ints.end(), isPrime);
assert(it == ints.end());  // No primes found!
```

So the observation here is by passing functions as parameters allows us to generalize algorithms with user-defined behavior !

## The Problem: What About Runtime Values?

Suppose we want to find a number less than N, where N is determined at runtime:

```cpp
int n;
std::cin >> n;
find_if(begin, end, /* lessThan... what? */);
```

The Naive Approach, We might try creating multiple functions:

```cpp
bool lessThan5(int x) { return x < 5; }
bool lessThan6(int x) { return x < 6; }
bool lessThan7(int x) { return x < 7; }

find_if(begin, end, lessThan5);
find_if(begin, end, lessThan6);
find_if(begin, end, lessThan7);
```

**Problem:** We can't create a function for every possible value of N at compile time!

### Can We Add Another Parameter?

```cpp
bool isLessThan(int elem, int n) {
    return elem < n;
}
```

**Problem:** This won't work with `find_if`! Look at our algorithm:

```cpp
template <typename It, typename Pred>
It find_if(It first, It last, Pred pred) {
    for (auto it = first; it != last; ++it) {
        if (pred(*it))  // We only pass ONE parameter to pred!
            return it;
    }
    return last;
}
```

The predicate `pred` is called with only one parameter (`*it`), so we can't pass the threshold value N here.

**The Challenge:** We need to give our function extra state (the value N) without adding another parameter to the predicate call.

So how can we add a state to the predicate.

The answer is a feature called **Functors (Function Objects)**

A **functor** is an object that can be called like a function. We create this by overloading the `operator()` in a class.

### What Makes Something Callable?

In `find_if`, we write `pred(*it)`. For this to work, `pred` needs to be **callable**. Three things in C++ are callable:
1. Regular functions
2. **Functors (objects with `operator()` overloaded)**
3. Lambda functions (we'll get to these!)

### Creating a Functor

```cpp
class LessThanN {
private:
    int threshold;
    
public:
    LessThanN(int n) : threshold(n) {}
    
    bool operator()(int x) const {
        return x < threshold;
    }
};
```

**How it works:**
- `LessThanN` is a class that stores the threshold value as member data
- The constructor allows us to set the threshold at runtime
- `operator()` makes objects of this class callable like a function
- The `const` means this doesn't modify the object's state

### Using the Functor

```cpp
int n;
std::cin >> n;

LessThanN lessThanN(n);  // Create a functor object with threshold n
find_if(begin, end, lessThanN);  // Pass the functor to the algorithm
```

Or more concisely:

```cpp
int n;
std::cin >> n;
find_if(begin, end, LessThanN(n));  // Create and pass in one line
```

### Why This Works

When `find_if` calls `pred(*it)`, it's actually calling `lessThanN.operator()(*it)`:

```cpp
// Inside find_if:
if (pred(*it))  // This becomes: lessThanN.operator()(*it)
```

The functor has **state** (the `threshold` member variable) that persists across multiple calls!

### Lets see some more examples of Functors

#### Example 1: Checking if a String is Longer Than N

```cpp
class LongerThan {
private:
    size_t minLength;
    
public:
    LongerThan(size_t len) : minLength(len) {}
    
    bool operator()(const std::string& str) const {
        return str.length() > minLength;
    }
};

std::vector<std::string> words = {"hi", "hello", "hey", "goodbye"};
auto it = find_if(words.begin(), words.end(), LongerThan(4));
// Finds "hello"
```

#### Example 2: Counting Occurrences (Stateful Functor)

```cpp
class Counter {
private:
    mutable int count;  // mutable allows modification in const function
    
public:
    Counter() : count(0) {}
    
    bool operator()(int x) const {
        count++;
        return false;  // Never actually "finds" anything
    }
    
    int getCount() const { return count; }
};

std::vector<int> nums = {1, 2, 3, 4, 5};
Counter counter;
find_if(nums.begin(), nums.end(), counter);
std::cout << "Checked " << counter.getCount() << " elements\n";
```

### Advantages of Functors

1. **State preservation**: Can store data between calls
2. **Type safety**: Each functor is its own type
3. **Optimization**: Compiler can inline the `operator()` calls
4. **Flexibility**: Can have multiple member functions and complex state

### Disadvantages of Functors

1. **Verbose**: Requires writing an entire class
2. **Boilerplate**: Lots of code for simple predicates
3. **Readability**: The logic is separated from where it's used

So in C++11 a wonderful feature has been introdcued named **Lamda functions**.

## Lambda Functions - The Modern Way

Lambda functions give us the benefits of functors with much cleaner syntax:

```cpp
int n;
std::cin >> n;

auto lessThanN = [n](int x) {
    return x < n;
};

find_if(begin, end, lessThanN);
```

### Lambda Syntax Breakdown

```cpp
[capture](parameters) { body }
```

- **Capture clause `[n]`**: What variables from the outer scope to "remember" (like member variables in a functor or state)
- **Parameters `(int x)`**: What gets passed when the lambda is called (like the parameters to `operator()`)
- **Body `{ return x < n; }`**: The code to execute (like the body of `operator()`)

**Lambdas are syntactic sugar for functors. They give us the power of function objects with the convenience of inline code!**

### Capture Modes
The parameters from the outerscope can be captured in varius modes.
Below are the modes.

```cpp
int x = 10, y = 20;

[x]        // Capture x by value (Variables captured by value are const by default (read-only))
[&x]       // Capture x by reference (Variables captured by reference can be modified)
[x, &y]    // Capture x by value, y by reference
[=]        // Capture all used variables by value
[&]        // Capture all used variables by reference
[=, &y]    // Capture all by value except y (by reference)
[&, x]     // Capture all by reference except x (by value)
```

Example:
```cpp
#include <iostream>
using namespace std;

int main() {
    int x = 10, y = 20;
    
    // [x] - Capture x by value (Read only)
    auto lambda1 = [x]() {
        cout << "x = " << x << endl;
        // x = 15; // ERROR: cannot modify x (captured by value is const)
    };
    lambda1();
    
    
    // [&x] - Capture x by reference (Can modify)
    auto lambda2 = [&x]() {
        cout << "Original x = " << x << endl;
        x = 15; // OK: can modify x
        cout << "Modified x = " << x << endl;
    };
    lambda2();
    cout << "x after lambda2: " << x << endl << endl;
    
    
    // [x, &y] - Capture x by value, y by reference
    x = 10; // Reset x
    auto lambda3 = [x, &y]() {
        cout << "x = " << x << ", y = " << y << endl;
        // x = 100; // ERROR: cannot modify x (captured by value)
        y = 25; // OK: can modify y (captured by reference)
    };
    lambda3();
    cout << "y after lambda3: " << y << endl << endl;
    
    
    // [=] - Capture all used variables by value (Read only)
    auto lambda4 = [=]() {
        cout << "x = " << x << ", y = " << y << endl;
        // x = 50; // ERROR: cannot modify x (captured by value)
        // y = 50; // ERROR: cannot modify y (captured by value)
    };
    lambda4();
    
    
    // [&] - Capture all used variables by reference (Can modify)
    auto lambda5 = [&]() {
        cout << "Before: x = " << x << ", y = " << y << endl;
        x = 30; // OK: can modify x
        y = 40; // OK: can modify y
        cout << "After: x = " << x << ", y = " << y << endl;
    };
    lambda5();
    cout << "After lambda5: x = " << x << ", y = " << y << endl << endl;
    
    
    // [=, &y] - Capture all by value except y (by reference)
    auto lambda6 = [=, &y]() {
        cout << "x = " << x << ", y = " << y << endl;
        // x = 100; // ERROR: cannot modify x (captured by value)
        y = 50; // OK: can modify y (captured by reference)
    };
    lambda6();
    cout << "y after lambda6: " << y << endl << endl;
    
    
    // [&, x] - Capture all by reference except x (by value)
    auto lambda7 = [&, x]() {
        cout << "x = " << x << ", y = " << y << endl;
        // x = 200; // ERROR: cannot modify x (captured by value)
        y = 60; // OK: can modify y (captured by reference)
    };
    lambda7();
    cout << "After lambda7: x = " << x << ", y = " << y << endl;
    
    return 0;
}
```

**Output:**
```
x = 10
Original x = 10
Modified x = 15
x after lambda2: 15

x = 10, y = 20
y after lambda3: 25

x = 10, y = 25

Before: x = 10, y = 25
After: x = 30, y = 40
After lambda5: x = 30, y = 40

x = 30, y = 40
y after lambda6: 50

x = 30, y = 50
After lambda7: x = 30, y = 60
```

## Lambda Capture with `mutable`

By default, variables captured **by value** in a lambda are **read-only** (const). 
If you need to modify the captured variable inside the lambda, use the `mutable` keyword.

However, `mutable` only allows you to modify a **local read-write copy** of the variable 
inside the lambda. Any changes made are **local to the lambda** and do not affect the 
original variable outside.

If you want to modify the **original variable**, you must capture it **by reference** 
using `&`.

### Example:
```cpp
int x = 10;

// Without mutable - Read only
auto lambda1 = [x]() {
    // x = 20; // ERROR: cannot modify
};

// With mutable - Local read-write copy
auto lambda2 = [x]() mutable {
    x = 20; // OK: modifies LOCAL copy only
};
lambda2();
cout << x; // Output: 10 (original unchanged)

// By reference - Modifies original
auto lambda3 = [&x]() {
    x = 30; // Modifies original x
};
lambda3();
cout << x; // Output: 30 (original changed)
```

**Note:** `mutable` gives you a read-write copy, but changes stay inside the lambda. 
Use `&` (reference) if you need to modify the actual variable.

### What Lambdas Really Are ?

Here is the fun part.
**Behind the scenes, the compiler turns a lambda into a functor!**

When you write:
```cpp
auto lessThanN = [n](int x) { return x < n; };
auto output = lessThanN(20);
```

The compiler generates something like the below:
```cpp

  class __lambda_6_22 // Compiler-generated name
  {
    public: 
    inline /*constexpr */ bool operator()(int x) const
    {
      return x < n;
    }
    
    private: 
    int n;
    
    public:
    __lambda_6_22(int & _n)
    : n{_n}
    {}
    
  };
  
__lambda_6_22 lessThanN = __lambda_6_22{n};
bool output = lessThanN.operator()(20);
```

## Passing Lambdas to Functions
One of the key advatage of `lamdas` is you can pass them to functions as paramters.
This feature is very useful for usecase like callback systems, Eventing etc.

Below are various ways you can accept `lamdas` as function parameters:

### Method 1: Using `std::function` (Most Flexible)
`std::function` is a general-purpose wrapper that can hold any callable object (lambda, function pointer, functor).

```cpp
#include <iostream>
#include <functional>
using namespace std;

// Function accepting lambda via std::function
void executeOperation(int a, int b, function<int(int, int)> operation) {
    int result = operation(a, b);
    cout << "Result: " << result << endl;
}

int main() {
    // Pass different lambdas
    executeOperation(10, 5, [](int x, int y) { return x + y; });     // 15
    executeOperation(10, 5, [](int x, int y) { return x - y; });     // 5
    executeOperation(10, 5, [](int x, int y) { return x * y; });     // 50
    
    return 0;
}
```

**Pros:** Flexible, can store lambdas with different captures  
**Cons:** Slight performance overhead (type erasure, heap allocation)

---

### Method 2: Using Template (Best Performance)
Templates allow the compiler to optimize the lambda call directly.

```cpp
#include <iostream>
using namespace std;

// Template function - accepts any callable
template<typename Func>
void executeOperation(int a, int b, Func operation) {
    int result = operation(a, b);
    cout << "Result: " << result << endl;
}

int main() {
    executeOperation(10, 5, [](int x, int y) { return x + y; });
    executeOperation(10, 5, [](int x, int y) { return x * y; });
    
    // Works with captures too
    int multiplier = 2;
    executeOperation(10, 5, [multiplier](int x, int y) { 
        return (x + y) * multiplier; 
    });
    
    return 0;
}
```

**Pros:** Zero overhead, compiler optimizations, works with any callable  
**Cons:** Template code in header files, longer compile times

---

### Method 3: Using Function Pointer (C-Style, Limited)

Only works with lambdas that **don't capture** anything (stateless).

```cpp
#include <iostream>
using namespace std;

// Function pointer for int(int, int) signature
void executeOperation(int a, int b, int (*operation)(int, int)) {
    int result = operation(a, b);
    cout << "Result: " << result << endl;
}

int main() {
    // Works - no capture
    executeOperation(10, 5, [](int x, int y) { return x + y; });
    
    // ERROR - cannot convert lambda with capture to function pointer
    int multiplier = 2;
    // executeOperation(10, 5, [multiplier](int x, int y) { return x * y; });
    
    return 0;
}
```

**Pros:** Lightweight, C-compatible  
**Cons:** Only works with non-capturing lambdas

---

### Method 4: Using `auto` (C++14+, Generic)

Perfect for generic code where you don't care about the exact type.

```cpp
#include <iostream>
using namespace std;

// Generic function using auto
auto executeOperation(int a, int b, auto operation) {
    return operation(a, b);
}

int main() {
    auto result1 = executeOperation(10, 5, [](int x, int y) { return x + y; });
    auto result2 = executeOperation(10, 5, [](int x, int y) { return x * y; });
    
    cout << "Result1: " << result1 << endl; // 15
    cout << "Result2: " << result2 << endl; // 50
    
    return 0;
}
```

**Note:** `auto` parameters require C++20, but templates work in C++11+.

---

## Lambda Evolution: C++11 to C++20

Lambdas have evolved significantly since their introduction in C++11.
Let's explore the incremental improvements across C++ standards.

### C++11: Lambda Introduction

C++11 introduced lambdas with basic functionality:

```cpp
// Basic lambda syntax
auto add = [](int a, int b) { return a + b; };

// Capture by value and reference
int x = 10;
auto byValue = [x]() { return x; };      // Captures copy of x
auto byRef = [&x]() { return x; };       // Captures reference to x

// Capture all
auto captureAll = [=]() { return x; };   // Capture all by value
auto captureAllRef = [&]() { return x; }; // Capture all by reference

// Mutable lambdas (can modify captured values)
auto counter = [count = 0]() mutable {
    return ++count;
};

// Explicit return type
auto divide = [](int a, int b) -> double {
    return static_cast<double>(a) / b;
};
```

**C++11 Limitations:**
- Cannot capture `*this` by value
- No `constexpr` support
- Cannot use `auto` as types for parameters
- Return type deduction limited to simple cases

### C++14: Generalized Lambda Captures & Generic Lambdas

C++14 added two major features:

#### 1. Generalized Lambda Captures (Init Captures)

You can now initialize captured variables with arbitrary expressions:

```cpp
// Move-only types in captures
auto ptr = std::make_unique<int>(42);
auto lambda = [ptr = std::move(ptr)]() {
    return *ptr;
};

// Initialize new variables in capture
auto lambda2 = [value = 5 * 2]() {
    return value;  // value is 10
};

// Complex initializations
std::string str = "Hello";
auto lambda3 = [s = std::move(str)]() {
    return s;  // str is moved into lambda
};

// Multiple initializations
auto lambda4 = [x = 1, y = 2, z = x + y]() {
    return z;  // z is 3
};
```

#### 2. Generic Lambdas (Auto Parameters)

Lambdas can now use `auto` for parameters, making them templates:

```cpp
// Generic lambda - works with any type
auto print = [](auto x) {
    std::cout << x << std::endl;
};

print(42);           // int
print(3.14);         // double
print("Hello");      // const char*

// Multiple auto parameters
auto add = [](auto a, auto b) {
    return a + b;
};

add(1, 2);           // int + int
add(1.5, 2.5);       // double + double
add(std::string("Hello"), std::string(" World"));  // string + string

// Mixing auto and concrete types
auto mixed = [](int x, auto y) {
    return x + y;
};
```

**What the compiler generates:**
```cpp
// Generic lambda
auto lambda = [](auto x) { return x * 2; };

// Becomes approximately:
struct __Lambda {
    template<typename T>
    auto operator()(T x) const {
        return x * 2;
    }
};
```

### C++17: Constexpr Lambdas & *this Capture

#### 1. Constexpr Lambdas

Lambdas are implicitly `constexpr` if they meet the requirements:

```cpp
// Implicitly constexpr
auto squared = [](int x) { return x * x; };
constexpr int result = squared(5);  // Evaluated at compile time

// Explicitly constexpr
constexpr auto cube = [](int x) constexpr { return x * x * x; };
static_assert(cube(3) == 27);

// Using in constexpr contexts
template<int N>
struct Array {
    static constexpr auto size = [](){ return N * 2; }();
};
```

#### 2. Capture *this by Value

Before C++17, when you capture this in a lambda inside a class member function, you only capture the pointer to the object, not the object itself. 

This creates a dangling pointer problem if the object is destroyed before the lambda is executed.

C++17 allows capturing the entire object instead of just the pointer:

```cpp
class Widget {
    int value = 42;
    
public:
    auto getLambda_Cpp11() {
        // Captures 'this' pointer - dangerous if object is destroyed
        return [this]() { return value; };
    }
    
    auto getLambda_Cpp17() {
        // Captures copy of entire object - safe!
        return [*this]() { return value; };
    }
    
    auto getLambda_Mutable() {
        // Captured copy can be modified
        return [*this]() mutable { return ++value; };
    }
};

Widget w;
auto lambda1 = w.getLambda_Cpp11();  // Captures pointer to w
auto lambda2 = w.getLambda_Cpp17();  // Captures copy of w
```

**Why this matters:**
```cpp
auto getLambda() {
    Widget w;
    return [w]() { return w.getValue(); };  // OK: w is copied
    // return [&w]() { return w.getValue(); };  // DANGER: w destroyed!
    // return [this]() { return value; };  // DANGER: this pointer dangling!
    return [*this]() { return value; };  // OK: object copied
}
```

### C++20: Template Lambdas & More

C++20 brought several powerful additions:

#### 1. Template Parameter Syntax for Lambdas

Lambdas can now explicitly specify template parameters:

```cpp
// Explicit template parameters
auto lambda = []<typename T>(T x) {
    std::cout << "Type: " << typeid(T).name() << std::endl;
    return x;
};

// Multiple template parameters
auto pair = []<typename T, typename U>(T first, U second) {
    return std::pair{first, second};
};

// Template parameter with constraints
auto process = []<typename T>(std::vector<T>& vec) {
    // Can use T explicitly in the body
    T sum = T{};
    for (const auto& elem : vec) {
        sum += elem;
    }
    return sum;
};

std::vector<int> nums = {1, 2, 3, 4, 5};
auto result = process(nums);
```

**Why this is useful:**
```cpp
// Before C++20: Can't get the type explicitly
auto oldWay = [](auto vec) {
    // How do we get the element type?
    using T = ???;  // No easy way!
};

// C++20: Direct access to template parameter
auto newWay = []<typename T>(std::vector<T> vec) {
    using ElementType = T;  // Clear and explicit!
    T defaultValue{};
    // ...
};
```

#### 2. Lambdas in Unevaluated Contexts

C++20 allows lambdas in contexts where they're not executed:

```cpp
// Lambda in decltype
auto lambda = [](int x) { return x * 2; };
using ReturnType = decltype(lambda(0));  // ReturnType is int

// Lambda in template parameter
template<auto Lambda>
struct Processor {
    static constexpr auto value = Lambda(10);
};

constexpr auto times2 = [](int x) { return x * 2; };
Processor<times2> p;  // p.value is 20

// Lambda for SFINAE/type traits
template<typename T>
concept Addable = requires(T a, T b) {
    { [](T x, T y) { return x + y; }(a, b) };
};
```

#### 3. Pack Expansion in Lambda Init-Capture

C++20 allows capturing parameter packs:

```cpp
// Variadic template with pack capture
template<typename... Args>
auto captureAll(Args... args) {
    return [...args = std::move(args)] {
        // Each arg is captured individually
        return (args + ...);  // Fold expression
    };
}

auto lambda = captureAll(1, 2, 3, 4);
std::cout << lambda() << std::endl;  // Output: 10

// More complex example
template<typename... Funcs>
auto compose(Funcs... funcs) {
    return [... f = std::move(funcs)](auto x) {
        // Apply all functions in sequence
        return (f(x), ...);  // Fold expression with comma operator
    };
}
```

#### 4. Default Constructible and Assignable Lambdas

C++20 lambdas without captures are default constructible and assignable:

```cpp
// Stateless lambda
auto lambda = [](int x) { return x * 2; };

// Can default construct
decltype(lambda) another;  // OK in C++20!
another = lambda;          // OK in C++20!

// Useful for containers
std::vector<decltype(lambda)> lambdas(10);  // Vector of 10 lambdas

// Compare lambdas
auto l1 = [](int x) { return x; };
auto l2 = l1;
// l1 == l2;  // Still not allowed - use std::function or comparison operators
```

#### 5. Lambdas with Concepts (C++20)

Constrain lambda parameters using concepts:

```cpp
#include <concepts>

// Lambda with concept constraint
auto process = []<std::integral T>(T value) {
    return value * 2;
};

process(5);      // OK: int is integral
// process(5.0);    // Error: double is not integral

// Multiple constraints
auto compare = []<typename T>(T a, T b) 
    requires std::equality_comparable<T> {
    return a == b;
};

// Constraint on return type
auto compute = []<typename T>(T x) -> std::integral auto {
    return static_cast<int>(x * 2);
};
```

### Practical Examples: Evolution in Action

Let's see how the same problem evolves across standards:

#### Problem: Create a customizable filter

**C++11:**
```cpp
// Need to specify types explicitly
auto createFilter(int threshold) {
    return [threshold](int value) {
        return value > threshold;
    };
}

std::vector<int> nums = {1, 5, 10, 15};
auto filter = createFilter(7);
// Can only use with int
```

**C++14:**
```cpp
// Generic lambda - works with any comparable type
auto createFilter(auto threshold) {
    return [threshold](auto value) {
        return value > threshold;
    };
}

std::vector<int> ints = {1, 5, 10, 15};
std::vector<double> doubles = {1.5, 5.5, 10.5};

auto filter = createFilter(7);
// Works with both int and double!
```

**C++17:**
```cpp
class FilterFactory {
    int defaultThreshold = 10;
    
public:
    auto createFilter() {
        // Safe capture of object by value
        return [*this](auto value) {
            return value > defaultThreshold;
        };
    }
};

FilterFactory factory;
auto filter = factory.createFilter();
// filter still works even if factory is destroyed
```

**C++20:**
```cpp
// Full type control with concepts
auto createFilter = []<std::totally_ordered T>(T threshold) {
    return [threshold]<std::totally_ordered U>(U value) 
        requires std::convertible_to<U, T> {
        return static_cast<T>(value) > threshold;
    };
};

auto intFilter = createFilter(10);
// intFilter(15);     // OK
// intFilter("test"); // Compile error: not convertible to int
```

### Complete Feature Comparison Table

| Feature | C++11 | C++14 | C++17 | C++20 |
|---------|-------|-------|-------|-------|
| Basic lambdas | ✅ | ✅ | ✅ | ✅ |
| Capture by value/reference | ✅ | ✅ | ✅ | ✅ |
| Mutable lambdas | ✅ | ✅ | ✅ | ✅ |
| Init captures | ❌ | ✅ | ✅ | ✅ |
| Generic lambdas (auto) | ❌ | ✅ | ✅ | ✅ |
| constexpr lambdas | ❌ | ❌ | ✅ | ✅ |
| Capture *this by value | ❌ | ❌ | ✅ | ✅ |
| Template parameter syntax | ❌ | ❌ | ❌ | ✅ |
| Pack expansion in captures | ❌ | ❌ | ❌ | ✅ |
| Unevaluated contexts | ❌ | ❌ | ❌ | ✅ |
| Default constructible | ❌ | ❌ | ❌ | ✅ |
| Concepts constraints | ❌ | ❌ | ❌ | ✅ |

## Conclusion

The progression from specific algorithms to general ones with predicates represents a fundamental principle in C++ programming: **abstraction without performance loss**.

- **Predicates** allow us to separate the "what to find" from the "how to search"
- **Functors** provide a way to package state with behavior
- **Lambdas** offer modern, concise syntax that the compiler transforms into functors

The evolution of lambdas from C++11 to C++20 shows the language's commitment to:
- **Expressiveness**: More ways to capture and initialize state
- **Safety**: Better lifetime management with `*this` captures
- **Performance**: Compile-time evaluation with `constexpr`
- **Flexibility**: Template parameters and concepts for better type control
