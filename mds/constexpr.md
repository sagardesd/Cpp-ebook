# constexpr (>= C++11), consteval (C++20) and constinit (C++20)

## Table of Contents
1. [Introduction](#introduction)
2. [C++11: Introduction of constexpr](#c11-introduction-of-constexpr)
   - [What is constexpr?](#what-is-constexpr)
   - [constexpr vs Template Metaprogramming](#constexpr-vs-template-metaprogramming)
3. [C++11 Limitations: The Single Return Statement Rule](#c11-limitations-the-single-return-statement-rule)
   - [The Problem](#the-problem)
   - [Workarounds in C++11](#workarounds-in-c11)
4. [C++14: Relaxed constexpr](#c14-relaxed-constexpr)
   - [What Changed in C++14?](#what-changed-in-c14)
   - [Multiple Statements Allowed](#multiple-statements-allowed)
   - [Loops in constexpr](#loops-in-constexpr)
   - [Comparison: C++11 vs C++14](#comparison-c11-vs-c14)
5. [C++20: Enhanced Compile-Time Programming](#c20-enhanced-compile-time-programming)
6. [C++20: Introduction of consteval](#c20-introduction-of-consteval)
   - [What is consteval?](#what-is-consteval)
   - [constexpr vs consteval](#constexpr-vs-consteval)
   - [When to Use consteval](#when-to-use-consteval)
7. [C++20: constinit](#c20-constinit)
8. [Practical Examples](#practical-examples)
9. [Benefits of Modern Compile-Time Programming](#benefits-of-modern-compile-time-programming)
10. [Best Practices](#best-practices)

---

## Introduction

Modern C++ has progressively enhanced compile-time programming capabilities. What started with template metaprogramming (TMP) evolved into more readable and powerful features with `constexpr` (C++11), relaxed `constexpr` (C++14), and `consteval` (C++20).

[↑ Back to Table of Contents](#table-of-contents)

---

## C++11: Introduction of constexpr

### What is constexpr?

C++11 introduced the `constexpr` keyword to enable compile-time computation in a more readable way than template metaprogramming.

**Key Features:**
- Functions marked `constexpr` can be evaluated at compile time
- Can also be used at runtime (unlike template metaprogramming)
- More readable than template metaprogramming
- Better error messages

**Syntax:**
```cpp
constexpr return_type function_name(parameters) {
    return expression;
}
```

Let's compare factorial using TMP vs constexpr:

**Template Metaprogramming (Pre-C++11):**
```cpp
#include <iostream>

template <size_t N>
struct Factorial {
    enum { value = N * Factorial<N - 1>::value };
};

template <>
struct Factorial<0> {
    enum { value = 1 };
};

int main() {
    std::cout << Factorial<7>::value << std::endl;  // Only compile-time
    return 0;
}
```

**C++11 constexpr:**
```cpp
#include <iostream>

constexpr int factorial(int n) {
    return (n == 0) ? 1 : n * factorial(n - 1);
}

int main() {
    // Compile-time evaluation
    constexpr int result1 = factorial(7);
    std::cout << result1 << std::endl;
    
    // Can also be used at runtime!
    int n;
    std::cin >> n;
    std::cout << factorial(n) << std::endl;  // Runtime evaluation
    
    return 0;
}
```

**Output:**
```
5040
```

---

### constexpr vs Template Metaprogramming

| Feature | Template Metaprogramming | constexpr |
|---------|-------------------------|-----------|
| **Readability** | Complex, hard to read | Clean, looks like normal code |
| **Flexibility** | Only compile-time | Both compile-time and runtime |
| **Error Messages** | Cryptic and long | Clear and concise |
| **Debugging** | Very difficult | Easier to debug |
| **Syntax** | Requires templates and specialization | Simple function syntax |

[↑ Back to Table of Contents](#table-of-contents)

---

## C++11 Limitations: The Single Return Statement Rule

### The Problem

In C++11, `constexpr` functions were severely limited:

**Restrictions:**
1. Must contain only a **single return statement**
2. No local variables allowed
3. No loops (for, while)
4. No if statements (only ternary operator `?:`)
5. Function body must be a single expression

**Example of the Limitation:**

```cpp
// ❌ This does NOT work in C++11
constexpr int fibonacci(int n) {
    if (n <= 1) return n;           // Error: multiple return statements
    return fibonacci(n-1) + fibonacci(n-2);
}

// ❌ This does NOT work in C++11
constexpr int sum_to_n(int n) {
    int sum = 0;                    // Error: local variable
    for (int i = 1; i <= n; ++i) {  // Error: loop
        sum += i;
    }
    return sum;
}
```

---

### Workarounds in C++11

To work around the single return statement limitation, you had to use recursion and ternary operators:

```cpp
#include <iostream>

// ✅ C++11 compliant - using ternary operator
constexpr int fibonacci(int n) {
    return (n <= 1) ? n : (fibonacci(n-1) + fibonacci(n-2));
}

// ✅ C++11 compliant - using recursion for sum
constexpr int sum_to_n_helper(int n, int sum) {
    return (n == 0) ? sum : sum_to_n_helper(n - 1, sum + n);
}

constexpr int sum_to_n(int n) {
    return sum_to_n_helper(n, 0);
}

int main() {
    constexpr int fib10 = fibonacci(10);
    constexpr int sum = sum_to_n(100);
    
    std::cout << "Fibonacci(10) = " << fib10 << std::endl;
    std::cout << "Sum(1..100) = " << sum << std::endl;
    
    return 0;
}
```

**Output:**
```
Fibonacci(10) = 55
Sum(1..100) = 5050
```

**Problem:** This is awkward and hard to read. Simple iterative algorithms require complex recursive solutions.

[↑ Back to Table of Contents](#table-of-contents)

---

## C++14: Relaxed constexpr

### What Changed in C++14?

C++14 **relaxed** the restrictions on `constexpr` functions, making them much more practical:

**New Capabilities:**
1. ✅ Multiple statements allowed
2. ✅ Local variables allowed
3. ✅ Loops (for, while, do-while)
4. ✅ If-else statements
5. ✅ Multiple return statements
6. ✅ switch statements
7. ✅ Modify local variables

---

### Multiple Statements Allowed

```cpp
#include <iostream>

// ✅ C++14: Multiple statements and local variables
constexpr int sum_to_n(int n) {
    int sum = 0;  // Local variable allowed!
    
    for (int i = 1; i <= n; ++i) {  // Loop allowed!
        sum += i;
    }
    
    return sum;  // Multiple statements allowed!
}

int main() {
    constexpr int result = sum_to_n(100);
    std::cout << "Sum(1..100) = " << result << std::endl;
    return 0;
}
```

**Output:**
```
Sum(1..100) = 5050
```

---

### Loops in constexpr

```cpp
#include <iostream>

// C++14: Factorial with loop instead of recursion
constexpr int factorial(int n) {
    int result = 1;
    for (int i = 2; i <= n; ++i) {
        result *= i;
    }
    return result;
}

// C++14: Fibonacci with loop
constexpr int fibonacci(int n) {
    if (n <= 1) return n;
    
    int prev = 0, curr = 1;
    for (int i = 2; i <= n; ++i) {
        int next = prev + curr;
        prev = curr;
        curr = next;
    }
    return curr;
}

int main() {
    constexpr int fact7 = factorial(7);
    constexpr int fib10 = fibonacci(10);
    
    std::cout << "7! = " << fact7 << std::endl;
    std::cout << "Fibonacci(10) = " << fib10 << std::endl;
    
    return 0;
}
```

**Output:**
```
7! = 5040
Fibonacci(10) = 55
```

---

### Comparison: C++11 vs C++14

**Finding the maximum in an array:**

**C++11 (Complex recursion):**
```cpp
constexpr int max_helper(const int* arr, int size, int current_max, int index) {
    return (index == size) ? current_max :
           max_helper(arr, size, 
                     (arr[index] > current_max ? arr[index] : current_max),
                     index + 1);
}

constexpr int find_max(const int* arr, int size) {
    return max_helper(arr, size, arr[0], 1);
}
```

**C++14 (Simple loop):**
```cpp
constexpr int find_max(const int* arr, int size) {
    int max_val = arr[0];
    for (int i = 1; i < size; ++i) {
        if (arr[i] > max_val) {
            max_val = arr[i];
        }
    }
    return max_val;
}
```

Much cleaner and more readable!

[↑ Back to Table of Contents](#table-of-contents)

---

## C++20: Enhanced Compile-Time Programming

C++20 significantly expanded what can be done at compile time, bringing `constexpr` closer to being as powerful as regular runtime code.

### constexpr Enhancements

**New C++20 Features:**
1. ✅ `constexpr` destructors
2. ✅ `constexpr` dynamic memory allocation (new/delete)
3. ✅ `constexpr` virtual functions
4. ✅ `constexpr` try-catch blocks
5. ✅ `constexpr` standard library containers
6. ✅ `constexpr` algorithms

---

### 1. constexpr Destructors

```cpp
#include <iostream>

struct ConstexprResource {
    constexpr ConstexprResource() {}
    constexpr ~ConstexprResource() {
        // Cleanup operations that must run at compile time
    }
};

constexpr void manage_resource() {
    ConstexprResource r; // Constructor and destructor called at compile time
}

int main() {
    constexpr auto result = manage_resource();
    return 0;
}
```

**Use Case:** Enables user-defined types (UDTs) with specific cleanup requirements to participate in `constexpr` contexts, supporting the creation of other `constexpr` features like containers.

---

### 2. constexpr Dynamic Memory Allocation (new/delete)

```cpp
#include <iostream>

constexpr int sum_array_elements() {
    int* arr = new int[4]{1, 2, 3, 4}; // Allocate at compile time
    int sum = 0;
    for (int i = 0; i < 4; ++i) {
        sum += arr[i];
    }
    delete[] arr; // Deallocate at compile time
    return sum;
}

int main() {
    constexpr int result = sum_array_elements();
    static_assert(result == 10);
    std::cout << "Sum: " << result << std::endl;
    return 0;
}
```

**Output:**
```
Sum: 10
```

**Use Case:** Vital for making standard library containers (`std::vector`, `std::string`) fully `constexpr`, allowing complex data structures to be built and processed entirely at compile time.

---

### 3. constexpr Virtual Functions

```cpp
#include <iostream>

struct Memory {
    constexpr virtual unsigned int capacity() const = 0; 
    constexpr virtual ~Memory() = default; 
};

struct EEPROM_25LC160C : Memory {
    constexpr unsigned int capacity() const override {
        return 2048; // A compile-time constant
    }
};

constexpr unsigned int get_eeprom_capacity() {
    EEPROM_25LC160C chip;
    return chip.capacity(); // Virtual dispatch happens at compile time
}

int main() {
    constexpr unsigned int cap = get_eeprom_capacity();
    static_assert(cap == 2048);
    std::cout << "EEPROM Capacity: " << cap << " bytes" << std::endl;
    return 0;
}
```

**Output:**
```
EEPROM Capacity: 2048 bytes
```

**Use Case:** Enables compile-time polymorphism for scenarios like hardware abstraction layers (HALs) where component properties can be determined during compilation. This was **impossible** before C++20!

---

### 4. constexpr try-catch Blocks

```cpp
#include <iostream>
#include <stdexcept>

constexpr int safe_divide(int a, int b) {
    if (b == 0) {
        throw std::runtime_error("Division by zero!");
    }
    return a / b;
}

constexpr int compute_quotient(int x) {
    try {
        return safe_divide(100, x);
    } catch (const std::runtime_error&) {
        return -1; 
    }
}

int main() {
    constexpr int result1 = compute_quotient(25);
    constexpr int result2 = compute_quotient(0);
    
    static_assert(result1 == 4);
    static_assert(result2 == -1);
    
    std::cout << "100 / 25 = " << result1 << std::endl;
    std::cout << "100 / 0 = " << result2 << " (error handled)" << std::endl;
    
    return 0;
}
```

**Output:**
```
100 / 25 = 4
100 / 0 = -1 (error handled)
```

**Use Case:** Allows library writers to maintain exception safety guarantees while still permitting their code to be used in `constexpr` contexts.

---

### 5. constexpr Standard Library Containers

C++20 allows dynamic containers at compile time:

```cpp
#include <iostream>
#include <vector>
#include <algorithm>

constexpr auto get_sorted_vector_back() {
    std::vector<int> my_vec = {1, 4, 2, 3}; // Works at compile time
    std::sort(my_vec.begin(), my_vec.end()); // Works at compile time
    return my_vec.back(); 
}

constexpr std::vector<int> create_squares(int n) {
    std::vector<int> squares;
    for (int i = 1; i <= n; ++i) {
        squares.push_back(i * i);
    }
    return squares;
}

int main() {
    constexpr int max_val = get_sorted_vector_back();
    static_assert(max_val == 4);
    
    std::cout << "Max value: " << max_val << std::endl;
    
    return 0;
}
```

**Output:**
```
Max value: 4
```

**Use Case:** Enables the preparation of complex, pre-processed data structures entirely at compile time, eliminating runtime initialization overhead.

---

### 6. constexpr Algorithms

```cpp
#include <iostream>
#include <algorithm>
#include <array>

constexpr std::array<int, 4> get_sorted_array() {
    std::array<int, 4> arr = {3, 1, 4, 2};
    std::sort(arr.begin(), arr.end()); // std::sort is constexpr in C++20
    return arr;
}

constexpr int find_max_with_algorithm() {
    std::array<int, 10> arr = {5, 2, 8, 1, 9, 3, 7, 4, 6, 10};
    
    // Use std::max_element at compile time!
    auto max_it = std::max_element(arr.begin(), arr.end());
    return *max_it;
}

int main() {
    constexpr auto sorted_arr = get_sorted_array();
    constexpr int max_val = find_max_with_algorithm();
    
    static_assert(sorted_arr[0] == 1 && sorted_arr[3] == 4);
    static_assert(max_val == 10);
    
    std::cout << "Sorted array: ";
    for (int val : sorted_arr) {
        std::cout << val << " ";
    }
    std::cout << std::endl;
    
    std::cout << "Max value: " << max_val << std::endl;
    
    return 0;
}
```

**Output:**
```
Sorted array: 1 2 3 4
Max value: 10
```

**Use Case:** Permits utility functions that rely on common algorithms (sorting, searching, transforming data) to be evaluated at compile time to produce final, optimized results embedded directly into the executable.

[↑ Back to Table of Contents](#table-of-contents)

---

## C++20: Introduction of consteval

### What is consteval?

C++20 introduced `consteval` for **immediate functions** - functions that **must** be evaluated at compile time.

**Key Difference:**
- `constexpr`: **Can** be evaluated at compile time, but may be evaluated at runtime
- `consteval`: **Must** be evaluated at compile time, never at runtime

**Syntax:**
```cpp
consteval return_type function_name(parameters) {
    // function body
}
```

---

### constexpr vs consteval

```cpp
#include <iostream>

constexpr int square_constexpr(int x) {
    return x * x;
}

consteval int square_consteval(int x) {
    return x * x;
}

int main() {
    // constexpr: Can use at compile time
    constexpr int a = square_constexpr(5);  // ✅ OK: Compile time
    
    // constexpr: Can also use at runtime
    int n = 10;
    int b = square_constexpr(n);  // ✅ OK: Runtime
    
    // consteval: Must use at compile time
    constexpr int c = square_consteval(7);  // ✅ OK: Compile time
    
    // consteval: CANNOT use at runtime
    // int d = square_consteval(n);  // ❌ Error: n is not a constant
    
    std::cout << "a = " << a << std::endl;
    std::cout << "b = " << b << std::endl;
    std::cout << "c = " << c << std::endl;
    
    return 0;
}
```

---

### When to Use consteval

Use `consteval` when:
1. You want to **guarantee** compile-time evaluation
2. You want to prevent accidental runtime usage
3. You're generating compile-time constants
4. You want to catch errors if non-constant arguments are passed

**Example: Compile-Time String Hashing**

```cpp
#include <iostream>
#include <string_view>

// Must be evaluated at compile time
consteval size_t hash_string(std::string_view str) {
    size_t hash = 0;
    for (char c : str) {
        hash = hash * 31 + c;
    }
    return hash;
}

int main() {
    // ✅ Compile time - string literal
    constexpr auto hash1 = hash_string("Hello");
    constexpr auto hash2 = hash_string("World");
    
    std::cout << "Hash of 'Hello': " << hash1 << std::endl;
    std::cout << "Hash of 'World': " << hash2 << std::endl;
    
    // ❌ This would be a compile error:
    // std::string s = "Runtime";
    // auto hash3 = hash_string(s);  // Error: s is not compile-time constant
    
    return 0;
}
```

[↑ Back to Table of Contents](#table-of-contents)

---

## C++20: constinit

C++20 also introduced `constinit` for variables that must be initialized at compile time but can be modified at runtime.

```cpp
#include <iostream>

// Must be initialized at compile time
constinit int global_value = 42;

constexpr int compute_value() {
    return 100;
}

constinit int computed_global = compute_value();

int main() {
    std::cout << "Global value: " << global_value << std::endl;
    
    // Can be modified at runtime (unlike constexpr variables)
    global_value = 100;
    std::cout << "Modified value: " << global_value << std::endl;
    
    return 0;
}
```

**Key Points:**
- `constinit`: Initialization must be at compile time, but value can change at runtime
- `constexpr`: Must be compile-time constant, cannot be modified

[↑ Back to Table of Contents](#table-of-contents)

---

## Practical Examples

### Compile-Time Prime Checker

```cpp
#include <iostream>

consteval bool is_prime(int n) {
    if (n <= 1) return false;
    if (n <= 3) return true;
    if (n % 2 == 0 || n % 3 == 0) return false;
    
    for (int i = 5; i * i <= n; i += 6) {
        if (n % i == 0 || n % (i + 2) == 0) {
            return false;
        }
    }
    return true;
}

int main() {
    constexpr bool result1 = is_prime(17);  // ✅ Compile time
    constexpr bool result2 = is_prime(100); // ✅ Compile time
    
    std::cout << "17 is prime: " << result1 << std::endl;
    std::cout << "100 is prime: " << result2 << std::endl;
    
    return 0;
}
```

---

### Compile-Time String Length

```cpp
#include <iostream>
#include <string_view>

consteval size_t string_length(std::string_view str) {
    return str.length();
}

int main() {
    constexpr auto len = string_length("Hello, World!");
    std::cout << "Length: " << len << std::endl;
    
    return 0;
}
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Benefits of Modern Compile-Time Programming

1. **Performance**: Zero runtime overhead - calculations done during compilation
2. **Type Safety**: Errors caught at compile time
3. **Readability**: Modern syntax is much cleaner than TMP
4. **Flexibility**: `constexpr` works at both compile-time and runtime
5. **Powerful**: C++20 allows almost any code to run at compile time
6. **Guarantees**: `consteval` ensures compile-time evaluation

**Evolution Summary:**

| Feature | C++11 | C++14 | C++20 |
|---------|-------|-------|-------|
| Single return only | ✅ | ❌ | ❌ |
| Multiple statements | ❌ | ✅ | ✅ |
| Loops | ❌ | ✅ | ✅ |
| Virtual functions | ❌ | ❌ | ✅ |
| Dynamic memory | ❌ | ❌ | ✅ |
| STL containers | ❌ | ❌ | ✅ |
| consteval | ❌ | ❌ | ✅ |

[↑ Back to Table of Contents](#table-of-contents)

---

## Best Practices

1. **Use `constexpr` by default** for functions that can be compile-time
2. **Use `consteval`** when you want to guarantee compile-time evaluation
3. **Prefer `constexpr` over TMP** for readability
4. **Use `constinit`** for globals that need compile-time initialization
5. **Test both paths**: If using `constexpr`, test both compile-time and runtime paths
6. **Be aware of compile times**: Complex `constexpr` can increase compilation time
7. **Use `if constexpr`** for compile-time branching (C++17)

[↑ Back to Table of Contents](#table-of-contents)
