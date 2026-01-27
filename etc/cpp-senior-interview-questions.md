# Top 120 Niche C++ Interview Questions for Senior Developers

A carefully curated collection of advanced C++ interview questions focusing on modern C++ features, template metaprogramming, move semantics, concurrency, RAII, smart pointers, STL containers, compile-time programming, lambdas, virtual functions, and lesser-known language intricacies.

---

## Template Metaprogramming & Advanced Templates (Questions 1-12)

### 1. **Explain SFINAE and provide a practical use case**
SFINAE (Substitution Failure Is Not An Error) is a C++ template metaprogramming technique where invalid template substitutions don't cause compilation errors but instead remove that template from overload resolution. Practical use: conditionally enabling functions based on type traits using `std::enable_if`.

### 2. **What are forwarding references (universal references) and how do they differ from rvalue references?**
Forwarding references use `T&&` syntax in a template context and can bind to both lvalues and rvalues through reference collapsing rules. Regular rvalue references (`Type&&`) only bind to rvalues. Forwarding references are essential for perfect forwarding with `std::forward`.

### 3. **Implement a compile-time factorial using template metaprogramming**
```cpp
template<int N>
struct Factorial {
    static constexpr int value = N * Factorial<N-1>::value;
};

template<>
struct Factorial<0> {
    static constexpr int value = 1;
};
```

### 4. **Explain the Curiously Recurring Template Pattern (CRTP). What are its advantages over virtual functions?**
CRTP is a pattern where a class derives from a template base class parameterized by the derived class itself: `class Derived : public Base<Derived>`. Advantages: compile-time polymorphism, no vtable overhead, better performance, and the ability to access derived class members from base.

### 5. **What is template template parameter? Provide an example**
Template template parameters allow templates to accept other templates as parameters:
```cpp
template<template<typename> class Container>
class MyClass {
    Container<int> data;
};

MyClass<std::vector> obj; // std::vector is the template template parameter
```

### 6. **How does `std::enable_if` work internally?**
`std::enable_if` uses SFINAE by defining a `type` member only when its boolean condition is true. If false, substitution fails and the template is removed from consideration. It's commonly used with `typename std::enable_if<condition, type>::type` or as a template parameter.

### 7. **Explain variadic templates and fold expressions (C++17)**
Variadic templates accept variable numbers of template arguments using parameter packs (`typename... Args`). Fold expressions (C++17) simplify recursive variadic template processing: `(args + ...)` folds a parameter pack with the `+` operator.

### 8. **What are the differences between partial and full template specialization?**
Full specialization provides a complete implementation for specific template arguments. Partial specialization provides specialized implementation for a subset of template parameters while keeping others generic. Partial specialization is only allowed for class templates, not function templates.

### 9. **How do you detect if a type has a specific member function at compile-time?**
Use SFINAE with `decltype` and `std::declval`:
```cpp
template<typename T>
class HasFoo {
    template<typename U>
    static auto test(U*) -> decltype(std::declval<U>().foo(), std::true_type{});
    
    template<typename>
    static std::false_type test(...);
    
public:
    static constexpr bool value = decltype(test<T>(nullptr))::value;
};
```

### 10. **Explain dependent names and typename keyword usage**
Dependent names are names that depend on template parameters. The `typename` keyword is required before dependent type names to tell the compiler it's a type, not a value: `typename T::value_type x;`

### 11. **What is Expression Templates and where is it useful?**
Expression Templates delay computation by building expression trees at compile-time instead of computing immediately. Used in linear algebra libraries (like Eigen) to optimize operations like `vector3 = vector1 + vector2 + vector3` into a single loop without temporaries.

### 12. **Implement `is_base_of` type trait using template metaprogramming**
```cpp
template<typename Derived, typename Base>
class IsBaseOf {
    static std::true_type test(Base*);
    static std::false_type test(...);
public:
    static constexpr bool value = 
        std::is_same_v<decltype(test(static_cast<Derived*>(nullptr))), std::true_type>;
};
```

---

## Move Semantics & Perfect Forwarding (Questions 13-28)

### 13. **What is the difference between lvalue, rvalue, prvalue, xvalue, and glvalue?**
Value categories: **lvalue** (has identity, can take address), **rvalue** (temporary/movable), **prvalue** (pure rvalue, no identity), **xvalue** (expiring value, like `std::move(x)`), **glvalue** (generalized lvalue = lvalue + xvalue). C++11 introduced this taxonomy for move semantics.

### 14. **Explain reference collapsing rules in detail**
Rules: `T& &` → `T&`, `T& &&` → `T&`, `T&& &` → `T&`, `T&& &&` → `T&&`. These rules enable perfect forwarding and are fundamental to how forwarding references work.

### 15. **What's the difference between `std::move` and `std::forward`?**
`std::move` unconditionally casts to rvalue reference. `std::forward` conditionally casts based on the template parameter - only casts to rvalue if the original argument was an rvalue. Used for perfect forwarding.

### 16. **Explain the Rule of Zero, Rule of Three, and Rule of Five**
**Rule of Zero**: If you can avoid defining special members, do (use RAII types).
**Rule of Three**: If you define destructor, copy constructor, or copy assignment, define all three.
**Rule of Five**: Add move constructor and move assignment to Rule of Three. If you define any, consider defining all five.

### 17. **When would a move constructor NOT be implicitly generated?**
Move constructor won't be generated if: user declares a copy constructor, copy assignment operator, move assignment operator, or destructor. This follows the "Rule of Five/Zero".

### 18. **What are the performance implications of returning by value with move semantics?**
Modern C++ (post-C++11) returns by value efficiently: RVO/NRVO eliminates copies entirely when possible. When RVO doesn't apply, move constructor is used instead of copy. Only falls back to copy if move is unavailable. Pattern: always return by value for local objects.

### 19. **Explain mandatory copy elision (C++17) and how it differs from NRVO**
C++17 guarantees copy elision for prvalues (temporary objects). No move/copy constructor needed. NRVO (Named RVO) is still optional and requires move/copy constructor to exist. Example: `return Widget{}` is guaranteed elision; `Widget w; return w;` is NRVO (optional).

### 20. **What happens when you call `std::move` on a const object?**
`std::move` on const objects degenerates to a copy because move constructors/assignments take non-const rvalue references. The const qualifier binds to const lvalue references, invoking copy operations instead. This is a common bug.

### 21. **How do you implement move semantics correctly for a class with raw pointers?**
```cpp
class Buffer {
    char* data;
    size_t size;
public:
    // Move constructor
    Buffer(Buffer&& other) noexcept 
        : data(other.data), size(other.size) {
        other.data = nullptr;
        other.size = 0;
    }
    
    // Move assignment
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data;
            data = other.data;
            size = other.size;
            other.data = nullptr;
            other.size = 0;
        }
        return *this;
    }
};
```
Key points: Mark `noexcept`, null out moved-from object, handle self-assignment.

### 22. **Why should move constructors and move assignment operators be `noexcept`?**
`noexcept` is critical for performance in standard containers. `std::vector::push_back` uses move only if move constructor is `noexcept`, otherwise copies for strong exception guarantee. Without `noexcept`, you lose move optimization benefits.

### 23. **Explain perfect forwarding failure cases**
Perfect forwarding fails with: braced initializers (type deduction fails), overloaded function names, bitfields (can't take address), 0 or NULL as nullptr (deduced as int), and declaration-only static const members.

### 24. **What is the "Has-A-Name" rule for rvalues?**
A named rvalue reference is actually an lvalue (it has a name). To forward it as an rvalue, you must explicitly use `std::move` or `std::forward`. This prevents accidental double-moves.

### 25. **What is the purpose of `std::move_if_noexcept`?**
Provides strong exception safety by conditionally moving only if the move constructor is noexcept. If not noexcept, it copies instead. Used in containers like `std::vector` during reallocation to maintain exception safety guarantees.

### 26. **Explain the moved-from state and what guarantees it provides**
After `std::move`, object must be in a valid but unspecified state. Can be destroyed and assigned to safely. No other operations guaranteed. Convention: leave in default-constructed or empty state. Standard library types follow this.

### 27. **How does Return Value Optimization (RVO) interact with move semantics?**
RVO eliminates copy/move entirely by constructing the object directly in the caller's location. Guaranteed copy elision (C++17) occurs for prvalues. NRVO (Named RVO) is optional. Even with move semantics, RVO is more efficient when applicable. Don't use `std::move` on return values—it inhibits RVO.

### 28. **What is the universal reference (forwarding reference) and how does template argument deduction work with it?**
`T&&` in a template context is a forwarding reference (universal reference). Binds to both lvalues and rvalues through deduction: lvalue → deduced as `T&`, rvalue → deduced as `T`. Combined with reference collapsing, enables perfect forwarding. Only works with type deduction, not concrete types.

---

## Memory Model & Atomics (Questions 29-36)

## Memory Model & Atomics (Questions 29-36)

### 29. **Explain the six memory ordering models in C++11**
- `memory_order_relaxed`: No ordering constraints, only atomicity guaranteed
- `memory_order_consume`: Data-dependency ordering (rarely used)
- `memory_order_acquire`: Acquire barrier for loads
- `memory_order_release`: Release barrier for stores
- `memory_order_acq_rel`: Both acquire and release
- `memory_order_seq_cst`: Sequentially consistent (default, strongest)

### 30. **What is the happens-before relationship?**
Happens-before establishes ordering between operations in a multithreaded program. If A happens-before B, then A's effects are visible to B. Created through synchronizes-with relationships (acquire-release pairs) or sequential execution in the same thread.

### 31. **Explain acquire-release semantics**
A store-release on one thread synchronizes-with a load-acquire on another thread if the load reads the stored value. All writes before the release become visible to reads after the acquire on the acquiring thread. Forms the foundation of lock-free programming.

### 32. **What's the difference between `std::atomic<T>::is_lock_free()` and `std::atomic<T>::is_always_lock_free`?**
`is_lock_free()` is a runtime query checking if the specific atomic instance uses lock-free operations. `is_always_lock_free` is a compile-time constant (static constexpr) indicating if that type is always lock-free on the platform.

### 33. **When would you use `memory_order_relaxed`?**
Use for counters where only the final value matters (like statistics/metrics), or when building happens-before relationships through other synchronization mechanisms. Provides atomicity without synchronization overhead.

### 34. **Explain the ABA problem in lock-free programming**
ABA occurs when a value changes from A→B→A. A CAS operation sees the same value but doesn't detect the intermediate change. Solutions: tagged pointers, hazard pointers, or use `std::atomic::compare_exchange_weak` in a loop with proper algorithm design.

### 35. **What are the differences between `compare_exchange_weak` and `compare_exchange_strong`?**
`weak` can spuriously fail even when values match (faster on some architectures). `strong` never spuriously fails. Use `weak` in loops where you'll retry anyway; use `strong` when spurious failure would be expensive.

### 36. **Explain memory fences (`std::atomic_thread_fence`)**
Fences establish synchronization without atomic variables. `atomic_thread_fence(memory_order_acquire)` or `atomic_thread_fence(memory_order_release)` can synchronize non-atomic operations. Stronger than atomic operations with same ordering.

---

## Modern C++ Features (Questions 37-44)

## Modern C++ Features (Questions 37-44)

### 37. **Explain structured bindings (C++17) and their limitations**
Structured bindings decompose objects: `auto [a, b] = pair;`. Works with arrays, tuple-like objects, and types with public data members. Limitations: can't declare types separately, can't skip elements (use `std::ignore` with tie), references must use `auto&`.

### 38. **What is `std::optional` and when should you use it instead of pointers?**
`std::optional<T>` represents a value that may or may not exist. Use instead of pointers for optional values to avoid null pointer issues, express intent clearly, and get value semantics with automatic cleanup. Avoids heap allocation.

### 39. **Explain `if constexpr` (C++17) and how it differs from regular `if`**
`if constexpr` evaluates condition at compile-time and only compiles the true branch. Enables different code paths for different template instantiations. Unlike regular `if`, the false branch isn't compiled at all, preventing compilation errors.

### 40. **What is `std::variant` and how does it compare to unions?**
`std::variant` is a type-safe union that knows which type it currently holds. Unlike unions, it properly constructs/destroys objects, provides type-safe access via `std::visit` or `std::get`, and throws on wrong type access.

### 41. **Explain `std::string_view` and its pitfalls**
`std::string_view` is a non-owning reference to a string. Zero-copy, efficient for read-only operations. Pitfalls: dangling references if the underlying string is destroyed, doesn't null-terminate, can't be safely returned from functions that create temporary strings.

### 42. **What are designated initializers (C++20)?**
Designated initializers allow initializing struct members by name: `Point p{.x = 10, .y = 20};`. Must be in declaration order. Improves readability and makes code less brittle when struct layout changes.

### 43. **Explain concepts (C++20) and their advantages over SFINAE**
Concepts define requirements on template parameters declaratively. More readable than SFINAE, provide better error messages, and make template constraints explicit: `template<std::integral T>`. Can compose with `&&`, `||`.

### 44. **What is `std::span` (C++20) and when should you use it?**
`std::span` is a non-owning view over contiguous memory (arrays, vectors, etc.). Provides bounds-checked access, size information, and iterator support. Use for function parameters instead of pointer+size or vector const references when you need a view into contiguous memory.

---

## Concurrency & Multithreading (Questions 45-50)

## Concurrency & Multithreading (Questions 45-50)

### 45. **Explain the differences between `std::mutex`, `std::recursive_mutex`, and `std::shared_mutex`**
`std::mutex`: Basic mutual exclusion, single lock. `std::recursive_mutex`: Can be locked multiple times by same thread. `std::shared_mutex`: Reader-writer lock; multiple readers OR single writer (C++17).

### 46. **What is `std::condition_variable` and how does it relate to spurious wakeups?**
Condition variables allow threads to wait for notifications. Spurious wakeups occur when `wait()` returns without `notify()`. Always use `wait()` with a predicate in a loop: `cv.wait(lock, []{return condition;})`.

### 47. **Explain deadlock and how to prevent it**
Deadlock occurs when threads wait for each other's resources circularly. Prevention: lock ordering (always acquire locks in same order), `std::lock` for multiple mutexes, timeout-based locking, or use lock-free data structures.

### 48. **What is `std::future` and `std::promise`?**
`std::promise` sets a value that `std::future` retrieves. Enables communication between threads. Promise writes once, future reads once. Use `std::async` for simpler async task execution. `shared_future` allows multiple readers.

### 49. **Explain memory barriers and their role in thread synchronization**
Memory barriers prevent instruction reordering across the barrier. Ensure visibility of writes between threads. Implicit in atomic operations with appropriate memory ordering. Essential for correct lock-free algorithms.

### 50. **What is thread_local storage and when should you use it?**
`thread_local` variables have separate instances per thread. Use for thread-specific data without locks (like random number generators, memory pools). Initialized on first access per thread. Destroyed when thread exits.

---

## RAII & Smart Pointers (Questions 51-58)

## RAII & Smart Pointers (Questions 51-58)

### 51. **Explain RAII and why it's considered a fundamental C++ idiom**
RAII (Resource Acquisition Is Initialization) ties resource lifetime to object lifetime. Resources are acquired in constructor, released in destructor. Guarantees cleanup even with exceptions. Foundation of exception-safe C++. Examples: `std::lock_guard`, `std::unique_ptr`, file handles.

### 52. **When would you use `std::unique_ptr` vs `std::shared_ptr` vs raw pointers?**
`std::unique_ptr`: Exclusive ownership, zero overhead, movable but not copyable. Use for most ownership scenarios.
`std::shared_ptr`: Shared ownership with reference counting. Use when multiple owners need the resource.
Raw pointers: Non-owning observers, legacy APIs, or performance-critical code where ownership is managed elsewhere.

### 53. **What is `std::make_unique` and why is it preferred over `new`?**
`std::make_unique<T>(args...)` creates a `unique_ptr` safely. Advantages: exception safety (avoids leaks in complex expressions), shorter syntax, single allocation. Prevents subtle bugs like `func(std::unique_ptr<T>(new T), might_throw())` where order of evaluation can leak.

### 54. **Explain the control block in `std::shared_ptr` and its performance implications**
Control block stores reference count, weak count, deleter, and allocator. Separate allocation from the managed object (unless using `make_shared`). Performance costs: atomic increments/decrements, extra indirection, cache misses. `make_shared` does single allocation for both.

### 55. **What's the difference between `std::shared_ptr` and `std::weak_ptr`?**
`std::weak_ptr` observes but doesn't own. Doesn't increment reference count. Prevents circular references. Must convert to `shared_ptr` via `lock()` to access object (returns empty `shared_ptr` if object is destroyed). Use for caches, observers, or breaking cycles.

### 56. **How do custom deleters work with smart pointers?**
`std::unique_ptr<T, Deleter>`: Deleter is part of type, zero overhead if stateless (EBO).
`std::shared_ptr<T>`: Deleter stored in control block, type-erased. Can use lambdas, function pointers, or functors. Useful for C APIs (`fclose`, `SDL_FreeSurface`) or custom cleanup.

### 57. **Explain the `std::enable_shared_from_this` pattern**
Allows an object to create `shared_ptr` instances pointing to itself. Inherits from `std::enable_shared_from_this<T>`. Provides `shared_from_this()` method. Critical for callbacks/async operations. Must already be owned by a `shared_ptr` or throws `bad_weak_ptr`.

### 58. **What are the dangers of circular references with `std::shared_ptr` and how do you break them?**
Circular references (A owns B, B owns A) prevent deallocation as reference counts never reach zero. Memory leak. Solution: use `weak_ptr` for one direction. Common pattern: parent uses `shared_ptr` to child, child uses `weak_ptr` to parent.

---

## Obscure Language Features & Edge Cases (Questions 59-66)

## Obscure Language Features & Edge Cases (Questions 59-66)

### 59. **Explain the Most Vexing Parse problem**
`Widget w(Foo());` is parsed as a function declaration, not object construction. C++ prioritizes function declarations when ambiguous. Solutions: use uniform initialization `Widget w{Foo{}}` or add extra parentheses `Widget w((Foo()))`.

### 60. **What is the difference between `struct` and `class` beyond default access?**
Only difference: default member access (`public` vs `private`) and default inheritance access. Otherwise identical. Convention: use `struct` for POD-like types, `class` for objects with methods/invariants.

### 61. **Explain the empty base optimization (EBO)**
Empty base classes take zero size due to EBO. Useful for policy-based design. C++20's `[[no_unique_address]]` extends this to members. Example: stateless allocators in containers.

### 62. **What happens when you throw an exception from a destructor?**
Throwing from a destructor during stack unwinding (when another exception is active) calls `std::terminate()`. Destructors are implicitly `noexcept` since C++11. Always catch and handle exceptions in destructors.

### 63. **Explain name lookup and why `using` is sometimes needed in templates**
Two-phase lookup in templates: non-dependent names at template definition, dependent names at instantiation. Use `this->` or `using Base::member` to find names in dependent base classes (CRTP scenarios).

### 64. **What is the "copy-and-swap" idiom?**
Idiom for implementing exception-safe assignment: create temporary copy, swap with `*this`, let destructor clean up. Provides strong exception guarantee and avoids self-assignment checks. Works well with move semantics.

### 65. **Explain the static initialization order fiasco**
Global/static objects in different translation units have undefined initialization order. Accessing uninitialized globals causes UB. Solutions: Meyers Singleton (function-local static), explicit initialization functions, or avoid cross-TU dependencies.

### 66. **What is Argument-Dependent Lookup (ADL) / Koenig Lookup?**
ADL includes namespaces of argument types when looking up function names. Enables `std::swap(a, b)` without namespace qualification. Can cause surprising behavior when unintended functions are found. Critical for customization points in generic code.

---

## Copy Constructor & Special Members (Questions 67-72)

### 67. **What is the copy-elision guarantee in C++17 and how does it affect copy constructors?**
C++17 guarantees copy elision for prvalues (temporary materialization). Even if copy/move constructors are deleted or private, code like `T obj = T()` compiles. Compiler constructs directly in destination. Named objects still require accessible copy/move constructor for NRVO (even if not called).

### 68. **Explain the difference between shallow copy and deep copy. When does the default copy constructor fail?**
Shallow copy copies pointer values (both point to same memory). Deep copy duplicates pointed-to data. Default copy constructor does shallow copy—fails with raw pointers, file handles, or any resource requiring unique ownership. Results in double-free, dangling pointers.

### 69. **What is the copy-on-write (COW) optimization and why is it problematic in multithreaded code?**
COW delays copying until modification. Multiple objects share data with reference count. Problem: reference count needs synchronization (atomic operations or mutex), defeating the optimization. C++11 banned COW for `std::string` to enable SSO and better thread safety.

### 70. **How does the copy constructor interact with inheritance?**
Derived class copy constructor must explicitly call base class copy constructor, otherwise base default constructor is called. Common bug: forgetting to copy base portion. Use member initializer list: `Derived(const Derived& other) : Base(other), member(other.member) {}`.

### 71. **Explain the copy elision rules for function parameters and return values**
Parameters: Never elided (need copy/move). Return values: NRVO is optional but widely implemented. C++17 prvalue guarantee applies to returns. Don't use `std::move` on return—inhibits RVO. Returning local by value preferred over returning pointer/reference.

### 72. **What happens when copy constructor throws an exception?**
Partially constructed object must be destroyed—destructors called for fully constructed members. Use member initializer list for exception safety. If copy constructor can throw, operations like `vector::push_back` can't provide strong guarantee without `noexcept` move constructor.

---

## STL Containers Deep Dive (Questions 73-80)

### 73. **Explain iterator invalidation rules for `std::vector`, `std::deque`, and `std::list`**
`std::vector`: All iterators/references invalidated on reallocation; insert/erase invalidates at and after position.
`std::deque`: Insert at middle invalidates all; insert at ends invalidates iterators but not references.
`std::list`: Only erased elements invalidated; insert/splice never invalidates.

### 74. **What is Small String Optimization (SSO) and how does it affect `std::string` performance?**
SSO stores short strings (typically ≤15-23 chars) directly in string object instead of heap allocation. Avoids dynamic allocation for small strings. Implementation-defined size. Affects performance: no heap allocation for short strings, but larger sizeof. Move is not free for SSO strings.

### 75. **Explain the difference between `std::map` and `std::unordered_map` in terms of complexity and when to use each**
`std::map`: O(log n) operations, ordered (red-black tree), stable iteration order, better worst-case.
`std::unordered_map`: O(1) average, O(n) worst-case (hash collisions), unordered, better average performance.
Use map for: ordered data, range queries, deterministic performance. Use unordered_map for: faster lookups, don't need ordering.

### 76. **What are the guarantees of `std::vector::push_back` vs `emplace_back`?**
Both provide strong exception guarantee when move constructor is `noexcept`. `push_back` constructs temporary then moves. `emplace_back` constructs in-place with perfect forwarding. `emplace_back` can be slower for types with cheap moves due to forwarding overhead. Use `push_back` for temporaries, `emplace_back` for in-place construction.

### 77. **Explain why `std::vector<bool>` is considered broken**
`std::vector<bool>` is specialized to pack bits (space optimization). Doesn't meet container requirements: `operator[]` returns proxy object, not `bool&`. Can't take address of element. Not thread-safe even for different elements. Use `std::deque<bool>` or `std::vector<char>` instead.

### 78. **What is the difference between `reserve()` and `resize()` for `std::vector`?**
`reserve(n)`: Allocates capacity for n elements, doesn't change size, no construction. Use before push_back to avoid reallocation.
`resize(n)`: Changes size, constructs/destructs elements. New elements default-constructed or value-initialized.
`reserve` for known insertion count; `resize` to actually create elements.

### 79. **How does `std::unordered_map` handle collisions and what is load factor?**
Handles collisions via chaining (linked lists). Load factor = size/bucket_count. Rehashes when load factor exceeds max_load_factor (default 1.0). Rehashing invalidates all iterators. Can reserve buckets with `reserve()`. Custom hash functions critical for performance.

### 80. **Explain the performance characteristics of inserting into middle of different containers**
`std::vector`: O(n) - must shift elements.
`std::deque`: O(n) - better than vector for front/back.
`std::list`: O(1) - once iterator position found.
`std::set/map`: O(log n) - tree rebalancing.
Choose based on insertion pattern: frequent middle insertion → list, mostly end → vector.

---

## Compile-Time Programming (Questions 81-87)

### 81. **What's the difference between `constexpr`, `consteval`, and `constinit` (C++20)?**
`constexpr`: Can run at compile-time OR runtime. Variables are implicitly const.
`consteval`: MUST run at compile-time (immediate functions). Compile error if called with runtime values.
`constinit`: Static/thread_local variables initialized at compile-time but NOT const (mutable). Prevents static initialization order fiasco.

### 82. **Can you have a `constexpr` function that doesn't run at compile-time?**
Yes! `constexpr` means "can be evaluated at compile-time if arguments are constant expressions." If called with runtime values, executes at runtime as normal function. `consteval` forces compile-time evaluation.

### 83. **What are the restrictions on `constexpr` functions in C++11 vs C++14 vs C++20?**
C++11: Single return statement, no loops, no mutation.
C++14: Loops, multiple statements, local variables, mutation allowed.
C++20: `new`/`delete`, virtual functions, `try`/`catch`, `std::vector`, `std::string` allowed.
Modern constexpr is nearly unrestricted.

### 84. **Why would you use `constinit` instead of `constexpr` for a global variable?**
`constinit` ensures compile-time initialization without forcing constness. Use when: need mutable global/static variable, want to avoid static initialization order fiasco, don't want variable usable in constant expressions. Example: configuration values that change at runtime but must initialize safely.

### 85. **Explain `if constexpr` and how it enables compile-time branching in templates**
`if constexpr` evaluates condition at compile-time. Discarded branch not compiled (doesn't need to be valid). Enables different code for different template instantiations. Unlike `std::enable_if`, provides cleaner syntax for conditional compilation within function body.

### 86. **Can a `constexpr` constructor contain `throw` statements?**
C++11: No. C++20: Yes, but if executed at compile-time and throws, compilation fails. If path not taken at compile-time, no error. Enables complex compile-time validation with runtime fallback.

### 87. **What happens when you use `constinit` with a non-static variable?**
Compile error. `constinit` only applies to static storage duration (global, static, thread_local). Local variables can't use `constinit`. Use `constexpr` for local compile-time constants.

---

## Modern C++ Attributes (Questions 88-93)

### 88. **What is `[[no_unique_address]]` (C++20) and when would you use it?**
Allows empty data members to take zero space (extends EBO to members). Useful for: stateless allocators, policy classes, empty base optimization for composition. Enables space-efficient policy-based design without inheritance.

### 89. **Explain `[[nodiscard]]` with a string message (C++20)**
Can provide custom diagnostic: `[[nodiscard("check error code")]]`. Compiler shows message when return value ignored. Improves code documentation and error messages. Use for critical return values where ignoring is likely a bug.

### 90. **What is `[[carries_dependency]]` and when would you use it?**
Related to `memory_order_consume`. Indicates pointer/reference carries dependency for optimization. Rarely used—`memory_order_consume` itself is rarely used due to implementation complexity. Most code uses `memory_order_acquire` instead.

### 91. **How does `[[maybe_unused]]` differ from commenting out warnings?**
Standard attribute, portable across compilers. Suppresses "unused variable" warnings. Use for: conditional compilation variables, function parameters in specific builds, debug-only variables. More explicit than `(void)x` casting.

### 92. **Can you combine multiple attributes on the same declaration?**
Yes: `[[nodiscard]] [[deprecated("use new_func")]] int old_func();`. Order doesn't matter. Standard attributes can be freely combined. Compiler-specific attributes may have restrictions.

### 93. **What is `[[assume]]` (C++23) and how does it help optimization?**
Tells compiler to assume condition is true without runtime check. Undefined behavior if assumption violated. Use for: invariants you've proven, optimization hints. Example: `[[assume(ptr != nullptr)]]` enables optimizations eliminating null checks.

---

## Lambda Expressions Advanced (Questions 94-100)

### 94. **Why do lambdas capture by value as `const` by default? When do you need `mutable`?**
Lambda's `operator()` is `const` by default. Captured-by-value variables can't be modified without `mutable`. Prevents accidental state mutation. Use `mutable` when lambda needs internal state (e.g., counters, accumulators). Reference captures can always modify.

### 95. **Explain the difference between capturing `[=]`, `[&]`, `[this]`, and `[*this]` (C++17)**
`[=]`: Capture all used locals by value (const).
`[&]`: Capture all used locals by reference.
`[this]`: Capture `this` pointer (access members).
`[*this]`: Capture copy of entire object by value (C++17). Use `[*this]` for async operations to avoid dangling references.

### 96. **What is a stateless lambda and why is it convertible to function pointer?**
Stateless lambda (empty capture `[]`) has no state. Convertible to C function pointer: `int(*)(int) = [](int x) { return x*2; }`. Useful for C APIs requiring function pointers. Lambda with captures can't convert—has state.

### 97. **Explain init-capture (generalized lambda capture) and move-only captures**
Init-capture (C++14): `[x = expr]` allows capturing with initialization. Enables move-only captures: `[ptr = std::move(unique_ptr)]`. Can rename: `[count = 0]`. Essential for moving `unique_ptr`, `thread` into lambdas.

### 98. **How do generic lambdas (C++14) differ from template functions?**
Generic lambdas: `[](auto x) {}` creates templated `operator()`. Syntax shorthand for template lambdas. Each call with different type instantiates new specialization. Can't explicitly specify template parameters (until C++20 template lambdas).

### 99. **What is the lifetime of lambda captures and what are the dangers?**
Captured-by-reference: dangling references if referent dies before lambda called. Captured-by-value: copies made when lambda created, independent lifetime. Capturing `this`: dangerous if object destroyed. Async operations: prefer `[*this]` or explicit copies.

### 100. **Explain immediately-invoked lambda expressions (IIFE) and their use cases**
Pattern: `auto result = [&]() { /* complex init */ return value; }();`. Use for: complex const initialization, scope-limited temporaries, conditional initialization without if-else duplication. Enables const variables with complex logic.

---

## Virtual Functions & OOP Deep Dive (Questions 101-107)

### 101. **Why is it important to make destructors virtual in base classes?**
Without virtual destructor, deleting derived object through base pointer calls only base destructor—derived portion not destroyed. Memory leak, resource leak. Rule: If class has any virtual function, destructor should be virtual. Modern: use `= default` virtual destructor.

### 102. **What is the performance cost of virtual functions?**
Indirect call through vtable (not inlineable, pipeline stall). Extra pointer per object (vtable pointer). Cache miss accessing vtable. Typically 1.5-3x slower than direct call. Negligible for I/O bound code, significant for tight loops. Use `final` to enable devirtualization.

### 103. **Explain pure virtual functions and abstract classes. Can abstract classes have constructors?**
Pure virtual: `virtual void f() = 0;`. Class with pure virtual is abstract—can't instantiate. Abstract classes CAN have constructors (called by derived). Can have partial implementation: `virtual void f() = 0; void f() { /* default impl */ }`. Derived can call `Base::f()`.

### 104. **What is the difference between `override` and `final` specifiers?**
`override`: Compiler verifies function actually overrides base virtual. Catches signature mismatches. Explicit intent documentation.
`final`: Prevents further overriding in derived classes (functions) or derivation (classes). Enables devirtualization optimization.
Use `override` always; use `final` for optimization or design constraint.

### 105. **Can you override a non-virtual function? What happens?**
Yes syntactically, but it's hiding, not overriding. Calls resolve based on static type, not dynamic type. Creates confusion—derived object behaves differently depending on pointer type. Almost always a bug. Use `override` to catch this.

### 106. **Explain covariant return types in virtual functions**
Overriding function can return pointer/reference to derived type: Base returns `Base*`, Derived returns `Derived*`. Enables type-safe factory methods. Return type must be covariant (derived from original). Doesn't work with value types, only pointers/references.

### 107. **What is the "slicing problem" and how do you prevent it?**
Assigning derived object to base variable copies only base portion—"slices off" derived part. Polymorphism lost. Prevention: use pointers/references, never pass polymorphic objects by value, make base class abstract or delete copy operations, use smart pointers.

---

## Scoped Enums & Casting (Questions 108-114)

### 108. **What are the advantages of `enum class` over traditional `enum`?**
Scoped enums (C++11): No implicit conversion to int, scoped names (no pollution), can specify underlying type, forward-declarable. Traditional enums: implicit int conversion, unscoped names, type determined by values. Use `enum class` by default.

### 109. **How do you convert between scoped enum and integer types?**
Explicit cast required: `static_cast<int>(MyEnum::Value)` or `static_cast<MyEnum>(42)`. No implicit conversion. Can use `std::underlying_type_t<MyEnum>` to get underlying integer type. Prevents accidental conversions.

### 110. **Explain the four types of C++ casts and when to use each**
`static_cast`: Compile-time checked conversions (int↔float, up/down cast with care). Use for well-understood conversions.
`dynamic_cast`: Runtime polymorphic downcast with nullptr on failure. Requires RTTI, performance cost.
`const_cast`: Add/remove const. Only case: interfacing with incorrect APIs. Usually a code smell.
`reinterpret_cast`: Reinterpret bit pattern. For low-level operations (pointer↔int). Highly dangerous.

### 111. **What is RTTI and when is `dynamic_cast` safe?**
RTTI (Runtime Type Information): enables `dynamic_cast` and `typeid`. Requires virtual functions. `dynamic_cast` safe when: pointer/reference to polymorphic type, checking before use, okay with nullptr. Slow due to vtable walk. Indicates design smell—consider virtual functions instead.

### 112. **Explain const-correctness and the different types of const member functions**
`void f() const`: Can't modify members (except mutable), can call const methods. Enables const object usage.
`const Type&`: Returns const reference, prevents modification.
Bitwise const: All members unchanged.
Logical const: Use `mutable` for caches/implementation details. Good design marks all non-modifying methods const.

### 113. **What is `mutable` keyword and when would you use it?**
Allows member modification in const member function. Use cases: caching (lazy evaluation), mutexes (synchronization), reference counts, implementation details that don't affect logical state. Don't abuse—violates const correctness if overused.

### 114. **Can you `const_cast` away const and modify the object?**
If original object was non-const: okay (unlikely to be useful).
If original object was const: undefined behavior.
Useful only for: interfacing with incorrect APIs that should take const. Almost always indicates API design flaw. Don't cast away const to "fix" const-correctness issues.

---

## Advanced Edge Cases & Best Practices (Questions 115-120)

### 115. **What is the difference between `nullptr`, `NULL`, and `0` in C++?**
`nullptr`: Type-safe null pointer literal (type `nullptr_t`), doesn't convert to int, works correctly with overload resolution.
`NULL`: Macro, typically `0` or `((void*)0)`, can cause overload resolution ambiguity.
`0`: Integer literal, converts to pointer in null context but can match integer overloads.
Always use `nullptr` in modern C++ to avoid ambiguity and express intent clearly.

### 116. **Explain the difference between `delete` and `delete[]`. What happens if you mix them?**
`delete`: Deallocates single object, calls one destructor.
`delete[]`: Deallocates array, calls destructor for each element.
Mixing causes undefined behavior: wrong number of destructors called, heap corruption. Use `std::vector` or `std::unique_ptr<T[]>` to avoid manual array management.

### 117. **What is the "as-if" rule in C++ optimization?**
Compiler can perform any optimization as long as observable behavior matches unoptimized code. Enables: dead code elimination, loop unrolling, inlining, reordering. Constraints: sequence points must be respected, volatile access preserved, I/O side effects maintained. Allows aggressive optimization while maintaining correctness.

### 118. **Explain name mangling and why `extern "C"` is needed**
Name mangling encodes function signature into symbol name for overloading, namespaces, templates. Each compiler uses different scheme. `extern "C"` disables mangling for C compatibility—function has C linkage, no overloading. Necessary for: C library interfaces, DLL exports, plugin systems. Can't use with overloaded functions or templates.

### 119. **What is aggregate initialization and how does it differ from list initialization?**
Aggregate initialization (C++20 enhanced): Initializes public data members in order without constructors: `Point p{1, 2};`. Works with structs/arrays without user-declared constructors. List initialization (`{}`): Works with constructors via `std::initializer_list` or aggregate initialization. Aggregate rules: no private members, no base classes (C++17), no virtual functions.

### 120. **Explain the "zero-overhead principle" in C++ and give examples where it's violated**
Zero-overhead: You don't pay for features you don't use; used features can't be implemented more efficiently by hand. Examples following principle: RAII, templates, constexpr. Violations: RTTI (vtable overhead even if unused), exceptions (code size bloat even in no-throw paths), `std::function` (type erasure overhead), `std::shared_ptr` (atomic operations). Choose features based on actual needs.

---

## Additional Tips for Senior C++ Interviews

### Key Areas to Master:
1. **Modern C++ Standards**: Be fluent in C++11/14/17/20 features
2. **Template Metaprogramming**: SFINAE, concepts, type traits
3. **Concurrency**: Memory model, atomics, synchronization primitives
4. **Performance**: Understanding compiler optimizations, cache effects, profiling
5. **Design Patterns**: CRTP, PIMPL, Policy-based design, Type erasure
6. **STL Internals**: How containers/algorithms work under the hood

### Interview Strategy:
- **Explain trade-offs**: Always discuss pros/cons of different approaches
- **Show code**: Write actual code, not pseudocode
- **Mention edge cases**: Demonstrate deep understanding
- **Discuss real-world usage**: Connect concepts to practical problems
- **Know when NOT to use advanced features**: Show judgment

### Resources for Deeper Study:
- "Effective Modern C++" by Scott Meyers
- "C++ Concurrency in Action" by Anthony Williams
- "C++ Templates: The Complete Guide" by Vandevoorde & Josuttis
- CppCon talks (YouTube)
- ISO C++ Standard drafts

---

**Remember**: These 120 questions test deep C++ knowledge. Focus on understanding WHY features exist and WHEN to use them, not just HOW they work. Senior developers should demonstrate wisdom in design choices, not just technical prowess.
