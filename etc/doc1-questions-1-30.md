# Top 120 C++ Interview Questions with Detailed Answers

---

## Template Metaprogramming & Advanced Templates (Questions 1-12)

### 1. **Explain SFINAE and provide a practical use case**

SFINAE (Substitution Failure Is Not An Error) is a template metaprogramming technique where invalid template substitutions remove that template from overload resolution instead of causing compilation errors.

When the compiler substitutes template parameters, if the substitution creates invalid code, that template is silently discarded. This enables conditional function overloading based on type properties.

**Example:**
```cpp
#include <type_traits>
#include <iostream>

// Enable only for integral types
template<typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
divide(T a, T b) {
    std::cout << "Integer division\n";
    return a / b;
}

// Enable only for floating-point types
template<typename T>
typename std::enable_if<std::is_floating_point<T>::value, T>::type
divide(T a, T b) {
    std::cout << "Float division\n";
    return a / b;
}

int main() {
    divide(10, 3);      // Calls integer version
    divide(10.0, 3.0);  // Calls float version
}
```

**Detecting member functions:**
```cpp
template<typename T>
class has_serialize {
    template<typename U>
    static auto test(U*) -> decltype(std::declval<U>().serialize(), std::true_type{});
    
    template<typename>
    static std::false_type test(...);
    
public:
    static constexpr bool value = decltype(test<T>(nullptr))::value;
};
```

---

### 2. **What are forwarding references (universal references) and how do they differ from rvalue references?**

Forwarding references use `T&&` in a template context where `T` is deduced, and can bind to both lvalues and rvalues. Regular rvalue references (`Type&&`) only bind to rvalues.

**Key difference:**
```cpp
void func(Widget&& w);           // Rvalue reference - only rvalues

template<typename T>
void func(T&& param);            // Forwarding reference - lvalues and rvalues

auto&& var = getValue();         // Forwarding reference
```

**How it works - Reference collapsing:**
```cpp
T& &   → T&
T& &&  → T&
T&& &  → T&
T&& && → T&&
```

**Example:**
```cpp
template<typename T>
void wrapper(T&& param) {
    // Forward preserving value category
    func(std::forward<T>(param));
}

int x = 42;
wrapper(x);           // T deduced as int&,  param is int&
wrapper(10);          // T deduced as int,   param is int&&
wrapper(std::move(x)); // T deduced as int,   param is int&&
```

Forwarding references enable perfect forwarding - preserving whether an argument was an lvalue or rvalue.

---

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

// Usage
constexpr int result = Factorial<5>::value;  // 120, computed at compile-time
```

**How it works:**
- Recursive template instantiation: `Factorial<5>` → `5 * Factorial<4>` → `5 * 4 * Factorial<3>` ...
- Base case: Template specialization for `N=0` returns 1
- All computation happens at compile-time, result is a constant

**Modern alternative (C++11+):**
```cpp
constexpr int factorial(int n) {
    return (n == 0) ? 1 : n * factorial(n - 1);
}

constexpr int result = factorial(5);  // Same compile-time evaluation, cleaner syntax
```

---

### 4. **Explain the Curiously Recurring Template Pattern (CRTP). What are its advantages over virtual functions?**

CRTP is when a class derives from a template base class parameterized by itself: `class Derived : public Base<Derived>`. This enables static (compile-time) polymorphism.

**Structure:**
```cpp
template<typename Derived>
class Base {
public:
    void interface() {
        static_cast<Derived*>(this)->implementation();
    }
};

class Derived : public Base<Derived> {
public:
    void implementation() {
        std::cout << "Derived implementation\n";
    }
};
```

**Example:**
```cpp
template<typename Derived>
class Shape {
public:
    double area() const {
        return static_cast<const Derived*>(this)->area_impl();
    }
};

class Circle : public Shape<Circle> {
    double radius_;
public:
    Circle(double r) : radius_(r) {}
    double area_impl() const { return 3.14159 * radius_ * radius_; }
};

class Rectangle : public Shape<Rectangle> {
    double w_, h_;
public:
    Rectangle(double w, double h) : w_(w), h_(h) {}
    double area_impl() const { return w_ * h_; }
};
```

**Advantages over virtual functions:**
1. **No runtime overhead** - Direct function calls, can be inlined
2. **No vtable pointer** - Smaller object size
3. **Return derived types** - Not limited to base class pointers
4. **Compile-time type safety** - Errors caught at compile-time
5. **Better performance** - No virtual dispatch, better optimization

**Trade-off:** Can't have heterogeneous collections or runtime polymorphism.

---

### 5. **What is template template parameter? Provide an example**

Template template parameters allow templates to accept other templates as parameters, not just types.

**Syntax:**
```cpp
template<template<typename> class Container>
class Stack {
    Container<int> data;
};

Stack<std::vector> s1;  // Container is std::vector
Stack<std::list> s2;    // Container is std::list
```

**Complete example:**
```cpp
template<typename T, template<typename, typename> class Container>
class Storage {
private:
    Container<T, std::allocator<T>> data_;
public:
    void add(const T& item) { data_.push_back(item); }
    size_t size() const { return data_.size(); }
};

// Usage
Storage<int, std::vector> vectorStorage;
Storage<int, std::deque> dequeStorage;
Storage<int, std::list> listStorage;
```

Note: STL containers have 2 template parameters (type and allocator), so the template template parameter must match: `template<typename, typename> class Container`.

**C++17 simplified syntax:**
```cpp
template<template<typename...> class Container>
class MyClass {
    Container<int> data;  // Works with any number of template parameters
};
```

---

### 6. **How does `std::enable_if` work internally?**

`std::enable_if` uses SFINAE by defining a `type` member only when its boolean condition is true. If false, there's no `type` member, causing substitution failure.

**Implementation:**
```cpp
template<bool B, typename T = void>
struct enable_if {};

template<typename T>
struct enable_if<true, T> {
    using type = T;
};
```

**Usage patterns:**

**1. Return type:**
```cpp
template<typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type
process(T value) {
    return value * 2;
}
// If T is not integral, enable_if<false, T> has no ::type, causing SFINAE
```

**2. Template parameter:**
```cpp
template<typename T, typename std::enable_if<std::is_integral<T>::value, int>::type = 0>
void process(T value) {
    // Only enabled for integral types
}
```

**3. Function parameter:**
```cpp
template<typename T>
void process(T value, typename std::enable_if<std::is_integral<T>::value>::type* = nullptr) {
    // Only enabled for integral types
}
```

**Modern C++14 shorthand:**
```cpp
template<typename T>
std::enable_if_t<std::is_integral_v<T>, T>
process(T value) {
    return value * 2;
}
```

---

### 7. **Explain variadic templates and fold expressions (C++17)**

Variadic templates accept variable numbers of template arguments using parameter packs (`typename... Args`).

**Basic syntax:**
```cpp
template<typename... Args>
void print(Args... args) {
    ((std::cout << args << " "), ...);  // Fold expression
}

print(1, 2.5, "hello", 'x');  // Prints: 1 2.5 hello x
```

**Recursive expansion (pre-C++17):**
```cpp
// Base case
void print() {}

// Recursive case
template<typename T, typename... Args>
void print(T first, Args... rest) {
    std::cout << first << " ";
    print(rest...);
}
```

**Fold expressions (C++17):**

Simplify parameter pack expansion:

```cpp
// Unary left fold: (... op pack)
template<typename... Args>
auto sum(Args... args) {
    return (... + args);  // ((arg1 + arg2) + arg3) + ...
}

// Unary right fold: (pack op ...)
template<typename... Args>
auto sum(Args... args) {
    return (args + ...);  // arg1 + (arg2 + (arg3 + ...))
}

// Binary left fold: (init op ... op pack)
template<typename... Args>
auto sum(Args... args) {
    return (0 + ... + args);  // (((0 + arg1) + arg2) + arg3) + ...
}
```

**Examples:**
```cpp
// All
template<typename... Args>
bool all(Args... args) {
    return (... && args);
}

// Any
template<typename... Args>
bool any(Args... args) {
    return (... || args);
}

// Print with fold
template<typename... Args>
void print(Args... args) {
    ((std::cout << args << '\n'), ...);
}
```

**Perfect forwarding with variadic templates:**
```cpp
template<typename T, typename... Args>
std::unique_ptr<T> make_unique(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}
```

---

### 8. **What are the differences between partial and full template specialization?**

**Full specialization:** Provides complete implementation for specific template arguments. All template parameters are specified.

```cpp
template<typename T>
class Container {
    T data;
public:
    void process() { std::cout << "Generic\n"; }
};

// Full specialization for int
template<>
class Container<int> {
    int data;
public:
    void process() { std::cout << "Specialized for int\n"; }
};
```

**Partial specialization:** Specializes for a subset of template parameters while keeping others generic. Only allowed for class templates.

```cpp
// Primary template
template<typename T, typename U>
class Pair {
public:
    void identify() { std::cout << "Generic pair\n"; }
};

// Partial specialization - both types same
template<typename T>
class Pair<T, T> {
public:
    void identify() { std::cout << "Same type pair\n"; }
};

// Partial specialization - U is pointer
template<typename T, typename U>
class Pair<T, U*> {
public:
    void identify() { std::cout << "Second is pointer\n"; }
};

Pair<int, double> p1;  // Uses generic
Pair<int, int> p2;     // Uses same type specialization
Pair<int, double*> p3; // Uses pointer specialization
```

**Key differences:**
- Full specialization: All parameters concrete (`Container<int>`)
- Partial specialization: Some parameters still generic (`Pair<T, T>`)
- Function templates: Can only be fully specialized, not partially
- Class templates: Can be both fully and partially specialized

---

### 9. **How do you detect if a type has a specific member function at compile-time?**

Use SFINAE with `decltype` and `std::declval`:

```cpp
#include <type_traits>

template<typename T>
class has_foo {
private:
    // This overload selected if T::foo() exists
    template<typename U>
    static auto test(U*) -> decltype(std::declval<U>().foo(), std::true_type{});
    
    // Fallback overload
    template<typename>
    static std::false_type test(...);
    
public:
    static constexpr bool value = decltype(test<T>(nullptr))::value;
};

// Usage
class WithFoo {
public:
    void foo() {}
};

class WithoutFoo {};

static_assert(has_foo<WithFoo>::value);      // true
static_assert(!has_foo<WithoutFoo>::value);  // true
```

**How it works:**
1. Try to call `std::declval<U>().foo()` in `decltype`
2. If `foo()` exists, this succeeds and returns `std::true_type`
3. If `foo()` doesn't exist, SFINAE removes this overload
4. Fallback `test(...)` is selected, returning `std::false_type`
5. `decltype` extracts the return type

**C++17 void_t pattern:**
```cpp
template<typename, typename = void>
struct has_foo : std::false_type {};

template<typename T>
struct has_foo<T, std::void_t<decltype(std::declval<T>().foo())>> : std::true_type {};
```

**C++20 with concepts:**
```cpp
template<typename T>
concept has_foo = requires(T t) {
    t.foo();
};
```

---

### 10. **Explain dependent names and typename keyword usage**

Dependent names are names that depend on template parameters. The compiler can't resolve them until template instantiation.

**Problem:**
```cpp
template<typename T>
void func() {
    T::value_type x;  // Error! Is value_type a type or a static member?
}
```

The compiler doesn't know if `T::value_type` is:
- A type (like `int`)
- A static member variable

**Solution - typename keyword:**
```cpp
template<typename T>
void func() {
    typename T::value_type x;  // Tell compiler it's a type
}
```

**When to use typename:**
```cpp
template<typename T>
class Container {
    // Need typename for dependent type names
    typename T::value_type getValue();
    
    // Need typename for nested types
    typename std::vector<T>::iterator it;
    
    // Need typename in template parameter lists
    template<typename U = typename T::nested_type>
    void process();
};
```

**Don't need typename:**
```cpp
template<typename T>
class Container {
    T value;                    // T is not dependent qualified name
    std::vector<T> vec;         // std::vector<T> is not dependent
    T::static_member = 5;       // Using as value, not type
};
```

**template keyword for dependent template names:**
```cpp
template<typename T>
void func() {
    T obj;
    obj.template method<int>();  // Need 'template' keyword
    
    typename T::template nested<int>::type x;  // Both typename and template
}
```

---

### 11. **What is Expression Templates and where is it useful?**

Expression Templates build compile-time expression trees to delay computation and enable optimization. Instead of computing immediately, operations create template objects representing the computation.

**Problem - Temporary objects:**
```cpp
Vector a, b, c, d;
Vector result = a + b + c + d;  // Creates 3 temporary vectors!
// Expands to: temp1 = a + b; temp2 = temp1 + c; temp3 = temp2 + d;
```

**Solution - Expression Templates:**
```cpp
template<typename E>
class VecExpression {
public:
    double operator[](size_t i) const {
        return static_cast<const E&>(*this)[i];
    }
    size_t size() const {
        return static_cast<const E&>(*this).size();
    }
};

class Vector : public VecExpression<Vector> {
    std::vector<double> data_;
public:
    double operator[](size_t i) const { return data_[i]; }
    size_t size() const { return data_.size(); }
    
    template<typename E>
    Vector(const VecExpression<E>& expr) {
        data_.resize(expr.size());
        for (size_t i = 0; i < size(); ++i)
            data_[i] = expr[i];  // Evaluated here, single loop!
    }
};

template<typename E1, typename E2>
class VecSum : public VecExpression<VecSum<E1, E2>> {
    const E1& lhs_;
    const E2& rhs_;
public:
    VecSum(const E1& lhs, const E2& rhs) : lhs_(lhs), rhs_(rhs) {}
    
    double operator[](size_t i) const {
        return lhs_[i] + rhs_[i];
    }
    size_t size() const { return lhs_.size(); }
};

template<typename E1, typename E2>
VecSum<E1, E2> operator+(const VecExpression<E1>& lhs, const VecExpression<E2>& rhs) {
    return VecSum<E1, E2>(static_cast<const E1&>(lhs), static_cast<const E2&>(rhs));
}

// Usage
Vector a, b, c, d;
Vector result = a + b + c + d;  // No temporaries! Single loop!
// Creates: VecSum<VecSum<VecSum<Vector, Vector>, Vector>, Vector>
```

**Benefits:**
- Eliminates temporary objects
- Fuses loops into single pass
- Enables compiler optimizations (vectorization, inlining)
- Used in: Eigen, Blitz++, Armadillo libraries

---

### 12. **Implement `is_base_of` type trait using template metaprogramming**

```cpp
template<typename Derived, typename Base>
class IsBaseOf {
private:
    // Converts Derived* to Base* if Derived inherits from Base
    static std::true_type test(Base*);
    
    // Fallback for non-related types
    static std::false_type test(...);
    
public:
    static constexpr bool value = 
        std::is_same<
            decltype(test(static_cast<Derived*>(nullptr))),
            std::true_type
        >::value;
};

// Usage
class Base {};
class Derived : public Base {};
class Unrelated {};

static_assert(IsBaseOf<Derived, Base>::value);      // true
static_assert(!IsBaseOf<Base, Derived>::value);     // false
static_assert(!IsBaseOf<Unrelated, Base>::value);   // false
```

**How it works:**
1. Try to convert `Derived*` to `Base*`
2. If Derived inherits from Base, conversion succeeds → `test(Base*)` called → returns `std::true_type`
3. Otherwise, conversion fails → `test(...)` called → returns `std::false_type`
4. Check return type with `std::is_same`

**More complete implementation:**
```cpp
template<typename Derived, typename Base>
struct IsBaseOf {
    static constexpr bool value = 
        std::is_class<Derived>::value &&
        std::is_class<Base>::value &&
        decltype(test(static_cast<Derived*>(nullptr)))::value;
        
private:
    template<typename T>
    static auto test(T*) -> typename std::is_convertible<T*, Base*>::type;
    
    template<typename>
    static std::false_type test(...);
};
```

This handles edge cases like non-class types and uses `std::is_convertible` for the actual check.

---

## Move Semantics & Perfect Forwarding (Questions 13-28)

### 13. **What is the difference between lvalue, rvalue, prvalue, xvalue, and glvalue?**

C++11 introduced a taxonomy of value categories:

**Primary categories:**
- **lvalue** (locator value): Has identity, you can take its address
- **rvalue**: Temporary or movable

**C++11 refines rvalues into:**
- **prvalue** (pure rvalue): Temporary with no identity (literals, `42`, `foo()`)
- **xvalue** (expiring value): Has identity but can be moved from (`std::move(x)`)

**Compound categories:**
- **glvalue** (generalized lvalue): lvalue or xvalue (has identity)

```cpp
int x = 10;              // x is lvalue
int& lref = x;           // lvalue reference binds to lvalue

int&& rref = 42;         // rvalue reference binds to prvalue
int&& rref2 = std::move(x); // binds to xvalue

// Value categories:
x                        // lvalue
42                       // prvalue
std::move(x)             // xvalue
foo()                    // prvalue (if returns by value)
x + y                    // prvalue
std::string("hello")     // prvalue
++x                      // lvalue (returns reference)
x++                      // prvalue (returns copy)
*ptr                     // lvalue
&x                       // prvalue
```

**Why it matters:**
- Overload resolution depends on value category
- Move semantics triggered by rvalues
- Understanding prevents bugs like `std::move` on const

---

### 14. **Explain reference collapsing rules in detail**

When references-to-references appear during template instantiation, they collapse according to these rules:

```cpp
T& &   → T&
T& &&  → T&
T&& &  → T&
T&& && → T&&
```

**Rule summary:** Rvalue reference only if both are rvalue references.

**Example:**
```cpp
template<typename T>
void func(T&& param);

int x = 42;

func(x);    // T deduced as int&
            // param type: int& && → int& (by collapsing)

func(10);   // T deduced as int
            // param type: int&&
```

**Why reference collapsing exists:**

It makes perfect forwarding possible:

```cpp
template<typename T>
void wrapper(T&& arg) {
    // If arg is lvalue: T is int&, forward<int&>(arg) returns int&
    // If arg is rvalue: T is int, forward<int>(arg) returns int&&
    func(std::forward<T>(arg));
}
```

**std::forward implementation:**
```cpp
template<typename T>
T&& forward(std::remove_reference_t<T>& param) {
    return static_cast<T&&>(param);
}

// If T is int&:  returns int& && → int&
// If T is int:   returns int&&
```

---

### 15. **What's the difference between `std::move` and `std::forward`?**

**std::move:** Unconditionally casts to rvalue reference.
**std::forward:** Conditionally casts based on template parameter.

```cpp
// std::move implementation
template<typename T>
std::remove_reference_t<T>&& move(T&& param) {
    return static_cast<std::remove_reference_t<T>&&>(param);
}

// std::forward implementation
template<typename T>
T&& forward(std::remove_reference_t<T>& param) {
    return static_cast<T&&>(param);
}
```

**When to use:**

```cpp
// std::move - when you KNOW you want to move
Widget w;
process(std::move(w));  // w is now in moved-from state

// std::forward - in templates to preserve value category
template<typename T>
void wrapper(T&& arg) {
    func(std::forward<T>(arg));  // Forwards as lvalue or rvalue
}

int x = 10;
wrapper(x);           // Forwards as lvalue
wrapper(10);          // Forwards as rvalue
```

**Example showing difference:**
```cpp
void process(int& x) { std::cout << "lvalue\n"; }
void process(int&& x) { std::cout << "rvalue\n"; }

template<typename T>
void test_forward(T&& x) {
    process(std::forward<T>(x));  // Preserves value category
}

template<typename T>
void test_move(T&& x) {
    process(std::move(x));  // Always rvalue
}

int a = 10;
test_forward(a);    // Prints: lvalue
test_forward(10);   // Prints: rvalue

test_move(a);       // Prints: rvalue
test_move(10);      // Prints: rvalue
```

---

### 16. **Explain the Rule of Zero, Rule of Three, and Rule of Five**

**Rule of Zero:** If you can avoid defining special member functions, do. Let compiler generate them. Use RAII types for resource management.

```cpp
class Good {
    std::string name_;
    std::vector<int> data_;
    std::unique_ptr<Resource> resource_;
    // Compiler generates: destructor, copy/move constructors, copy/move assignment
};
```

**Rule of Three:** If you define ANY of these, define ALL three:
- Destructor
- Copy constructor
- Copy assignment operator

```cpp
class Rule3 {
    int* data_;
public:
    ~Rule3() { delete data_; }
    
    Rule3(const Rule3& other) : data_(new int(*other.data_)) {}
    
    Rule3& operator=(const Rule3& other) {
        if (this != &other) {
            delete data_;
            data_ = new int(*other.data_);
        }
        return *this;
    }
};
```

**Rule of Five:** If you define ANY of the five, consider defining ALL five:
- Destructor
- Copy constructor
- Copy assignment operator
- Move constructor
- Move assignment operator

```cpp
class Rule5 {
    int* data_;
public:
    ~Rule5() { delete data_; }
    
    Rule5(const Rule5& other) : data_(new int(*other.data_)) {}
    
    Rule5& operator=(const Rule5& other) {
        if (this != &other) {
            delete data_;
            data_ = new int(*other.data_);
        }
        return *this;
    }
    
    Rule5(Rule5&& other) noexcept : data_(other.data_) {
        other.data_ = nullptr;
    }
    
    Rule5& operator=(Rule5&& other) noexcept {
        if (this != &other) {
            delete data_;
            data_ = other.data_;
            other.data_ = nullptr;
        }
        return *this;
    }
};
```

**Which to follow:**
- **Rule of Zero**: Always prefer this
- **Rule of Three**: Legacy code or C++98
- **Rule of Five**: When managing resources manually

---

### 17. **When would a move constructor NOT be implicitly generated?**

Move constructor won't be implicitly generated if you declare:
1. Copy constructor
2. Copy assignment operator
3. Move assignment operator
4. Destructor

```cpp
class NoMove1 {
    ~NoMove1() {}  // Destructor declared → no move constructor
};

class NoMove2 {
    NoMove2(const NoMove2&) = default;  // Copy constructor → no move
};

class NoMove3 {
    NoMove3& operator=(const NoMove3&) = default;  // Copy assignment → no move
};

class HasMove {
    // No special members declared → compiler generates move constructor
};
```

**To explicitly get move with destructor:**
```cpp
class ExplicitMove {
public:
    ExplicitMove() = default;
    ~ExplicitMove() { /* cleanup */ }
    
    ExplicitMove(const ExplicitMove&) = default;
    ExplicitMove& operator=(const ExplicitMove&) = default;
    ExplicitMove(ExplicitMove&&) = default;
    ExplicitMove& operator=(ExplicitMove&&) = default;
};
```

---

### 18. **What are the performance implications of returning by value with move semantics?**

Modern C++ makes returning by value efficient through:
1. **RVO/NRVO**: Copy/move elision (no constructor called at all)
2. **Move semantics**: If RVO doesn't apply, move instead of copy
3. **Only fallback to copy** if move unavailable

```cpp
std::vector<int> createVector() {
    std::vector<int> result;
    result.push_back(1);
    return result;  // RVO or move, never copy
}

// Caller
auto vec = createVector();  // No copy made!
```

**Performance comparison:**
```cpp
// C++98 - forced copy
std::vector<int> getData() {
    std::vector<int> v(1000000);
    return v;  // Expensive copy!
}

// C++11 - RVO or move
std::vector<int> getData() {
    std::vector<int> v(1000000);
    return v;  // RVO (zero cost) or move (cheap)
}
```

**Don't use std::move on return:**
```cpp
std::vector<int> wrong() {
    std::vector<int> v;
    return std::move(v);  // ❌ Inhibits RVO!
}

std::vector<int> correct() {
    std::vector<int> v;
    return v;  // ✅ Enables RVO, falls back to move
}
```

---

### 19. **Explain mandatory copy elision (C++17) and how it differs from NRVO**

**Copy elision:** Compiler optimizes away copy/move constructors by constructing object directly in final location.

**C++17 mandatory copy elision (prvalue):**
Guaranteed for temporary objects (prvalues). Copy/move constructor doesn't even need to exist.

```cpp
struct NonMovable {
    NonMovable() = default;
    NonMovable(const NonMovable&) = delete;
    NonMovable(NonMovable&&) = delete;
};

NonMovable create() {
    return NonMovable();  // OK in C++17 - mandatory elision
}

auto obj = create();  // OK - no copy or move needed
```

**NRVO (Named Return Value Optimization):**
Optional optimization for named objects. Copy/move constructor must exist even if not called.

```cpp
std::string createString() {
    std::string str("hello");
    return str;  // NRVO may or may not happen
}
// Move constructor must exist, but likely not called
```

**Key differences:**
- **Guaranteed vs Optional**: C++17 elision is mandatory, NRVO is optional
- **Named vs Unnamed**: Mandatory for prvalues, NRVO for named objects
- **Constructor requirement**: Mandatory elision doesn't need constructor, NRVO does

```cpp
Widget create() {
    return Widget();  // C++17: Guaranteed elision
}

Widget create() {
    Widget w;
    return w;  // NRVO: Optional (but usually applied)
}

Widget create() {
    Widget w;
    return std::move(w);  // ❌ Prevents NRVO, forces move
}
```

---

### 20. **What happens when you call `std::move` on a const object?**

`std::move` on const object casts to `const T&&`, which binds to const lvalue reference, invoking copy constructor instead of move constructor.

```cpp
class Widget {
public:
    Widget() = default;
    Widget(const Widget&) { std::cout << "Copy\n"; }
    Widget(Widget&&) { std::cout << "Move\n"; }
};

const Widget w1;
Widget w2 = std::move(w1);  // Prints: "Copy" (not "Move"!)
```

**Why:**
Move constructors take non-const rvalue reference: `Widget(Widget&& other)`
Const rvalue reference `const Widget&&` can't bind to non-const reference.
Falls back to copy constructor: `Widget(const Widget& other)`

**This is a common bug:**
```cpp
const std::string str = "hello";
std::string str2 = std::move(str);  // Copies! const prevents move

// Correct - don't make movable objects const
std::string str = "hello";
std::string str2 = std::move(str);  // Actually moves
```

---

### 21. **How do you implement move semantics correctly for a class with raw pointers?**

```cpp
class Buffer {
    char* data_;
    size_t size_;
    
public:
    // Constructor
    Buffer(size_t size) : data_(new char[size]), size_(size) {}
    
    // Destructor
    ~Buffer() { delete[] data_; }
    
    // Copy constructor
    Buffer(const Buffer& other) : data_(new char[other.size_]), size_(other.size_) {
        std::memcpy(data_, other.data_, size_);
    }
    
    // Copy assignment
    Buffer& operator=(const Buffer& other) {
        if (this != &other) {
            delete[] data_;
            size_ = other.size_;
            data_ = new char[size_];
            std::memcpy(data_, other.data_, size_);
        }
        return *this;
    }
    
    // Move constructor
    Buffer(Buffer&& other) noexcept 
        : data_(other.data_), size_(other.size_) {
        other.data_ = nullptr;
        other.size_ = 0;
    }
    
    // Move assignment
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            data_ = other.data_;
            size_ = other.size_;
            other.data_ = nullptr;
            other.size_ = 0;
        }
        return *this;
    }
};
```

**Key points:**
1. **Mark noexcept** - Critical for std::vector and exception safety
2. **Null out source** - Prevents double-delete
3. **Handle self-assignment** - Check `this != &other`
4. **Leave moved-from valid** - Can be safely destroyed

---

### 22. **Why should move constructors and move assignment operators be `noexcept`?**

`noexcept` is critical because STL containers use move only if it's `noexcept`, otherwise they copy for strong exception guarantee.

```cpp
class Widget {
public:
    Widget(Widget&&) {  // Not noexcept
        // vector will COPY during reallocation
    }
};

class Better {
public:
    Widget(Widget&&) noexcept {  // noexcept
        // vector will MOVE during reallocation (fast!)
    }
};
```

**std::vector reallocation logic:**
```cpp
if (std::is_nothrow_move_constructible<T>::value) {
    move_elements();  // Fast
} else {
    copy_elements();  // Slow but safe
}
```

Without `noexcept`, vector must copy to maintain strong exception guarantee.

---

### 23. **Explain perfect forwarding failure cases**

Perfect forwarding fails in these cases:

**1. Braced initializers:**
```cpp
template<typename T>
void forward_func(T&& arg);

real_func({1, 2, 3});      // OK
forward_func({1, 2, 3});   // ❌ Error: can't deduce type
```

**2. Overloaded function names:**
```cpp
void process(int);
void process(double);

forward_func(process);  // ❌ Error: which overload?
forward_func<void(*)(int)>(process);  // ✅ Fix
```

**3. Bitfields:**
```cpp
struct Bits { unsigned int a : 1; };
Bits b;
forward_func(b.a);  // ❌ Can't take address of bitfield
```

**4. 0 or NULL as nullptr:**
```cpp
forward_func(nullptr);  // ✅ OK
forward_func(NULL);     // ❌ NULL is int, not pointer
```

---

### 24. **What is the "Has-A-Name" rule for rvalues?**

A named rvalue reference is an lvalue. Must explicitly use `std::move` or `std::forward` to treat it as rvalue.

```cpp
void process(int&& x) {
    // x has a name, so it's an lvalue!
    func(x);             // Calls lvalue overload
    func(std::move(x));  // Calls rvalue overload
}

void func(int&) { std::cout << "lvalue\n"; }
void func(int&&) { std::cout << "rvalue\n"; }

process(42);
// Prints: lvalue
//         rvalue
```

**Prevents double-move:**
```cpp
std::string process(std::string&& str) {
    modify(str);    // Uses lvalue - won't move
    return str;     // Moved on return
}
```

---

### 25. **What is the purpose of `std::move_if_noexcept`?**

Moves only if move constructor is `noexcept`, otherwise copies for exception safety.

```cpp
class Widget {
public:
    Widget(Widget&&) noexcept { std::cout << "Move\n"; }
    Widget(const Widget&) { std::cout << "Copy\n"; }
};

Widget w1;
Widget w2 = std::move_if_noexcept(w1);  // Prints: Move

class Risky {
public:
    Risky(Risky&&) { std::cout << "Move\n"; }  // Can throw
    Risky(const Risky&) { std::cout << "Copy\n"; }
};

Risky r1;
Risky r2 = std::move_if_noexcept(r1);  // Prints: Copy (safer!)
```

Used in `std::vector` reallocation to maintain strong exception guarantee.

---

### 26. **Explain the moved-from state and what guarantees it provides**

After `std::move`, object must be in **valid but unspecified state**:
- Can be destroyed safely
- Can be assigned to
- No other operations guaranteed

```cpp
std::string str = "hello";
std::string str2 = std::move(str);

// Guaranteed operations:
str.~string();        // ✅ Can destroy
str = "new value";    // ✅ Can assign

// Unspecified operations:
std::cout << str;     // ⚠️ Unspecified content
str.size();           // ⚠️ Unspecified value
```

**Convention - leave in default-constructed or empty state:**
```cpp
class Resource {
    int* data_;
public:
    Resource(Resource&& other) noexcept : data_(other.data_) {
        other.data_ = nullptr;  // Empty state
    }
};
```

---

### 27. **How does Return Value Optimization (RVO) interact with move semantics?**

RVO eliminates copy/move entirely by constructing directly in caller's location.

**Priority: RVO > Move > Copy**

```cpp
Widget create() {
    Widget w;
    return w;  // 1. Try RVO  2. Try move  3. Copy
}
```

**Guaranteed copy elision (C++17) for prvalues:**
```cpp
Widget create() {
    return Widget();  // Guaranteed RVO
}

Widget w = create();  // Constructed directly in w
```

**Don't use `std::move` on return - inhibits RVO:**
```cpp
Widget wrong() {
    Widget w;
    return std::move(w);  // ❌ Prevents RVO!
}

Widget correct() {
    Widget w;
    return w;  // ✅ Enables RVO or move
}
```

---

### 28. **What is the universal reference and how does template argument deduction work with it?**

Universal/forwarding reference is `T&&` where T is deduced. Binds to both lvalues and rvalues.

**Deduction rules:**
```cpp
template<typename T>
void func(T&& param);

int x = 42;

func(x);            // T = int&,  param = int&
func(10);           // T = int,   param = int&&
func(std::move(x)); // T = int,   param = int&&
```

**How it works with reference collapsing:**
```cpp
int x = 42;
func(x);

// T deduced as int&
// Substitution: void func(int& && param)
// Collapsing: int& && → int&
// Final: void func(int& param)
```

**Perfect forwarding:**
```cpp
template<typename T>
void wrapper(T&& arg) {
    func(std::forward<T>(arg));  // Preserves value category
}

int x = 10;
wrapper(x);    // Forwards as lvalue
wrapper(10);   // Forwards as rvalue
```

---

## Memory Model & Atomics (Questions 29-36)

### 29. **Explain the six memory ordering models in C++11**

**`memory_order_relaxed`:**
Only atomicity guaranteed, no ordering

```cpp
counter.fetch_add(1, std::memory_order_relaxed);
```

**`memory_order_consume`:**
Data-dependency ordering (rarely used)

**`memory_order_acquire`:**
Load operation - prevents reordering after

```cpp
if (ready.load(std::memory_order_acquire)) {
    // All writes before paired release are visible
}
```

**`memory_order_release`:**
Store operation - prevents reordering before

```cpp
data = 42;
ready.store(true, std::memory_order_release);
```

**`memory_order_acq_rel`:**
Both acquire and release for read-modify-write

```cpp
counter.fetch_add(1, std::memory_order_acq_rel);
```

**`memory_order_seq_cst`:**
Sequentially consistent (default, strongest, slowest)

```cpp
x.store(1);  // Default is seq_cst
```

**Example with acquire-release:**
```cpp
std::atomic<bool> ready{false};
int data;

// Thread 1
data = 42;
ready.store(true, std::memory_order_release);

// Thread 2
if (ready.load(std::memory_order_acquire)) {
    assert(data == 42);  // Guaranteed
}
```

---

### 30. **What is the happens-before relationship?**

Happens-before establishes ordering between operations. If A happens-before B, A's effects are visible to B.

**Sources:**

**1. Sequenced-before (same thread):**
```cpp
int x = 0;
x = 1;       // Sequenced after x = 0
int y = x;   // Sees x == 1
```

**2. Synchronizes-with (across threads):**
```cpp
std::atomic<bool> flag{false};
int data;

// Thread 1
data = 42;
flag.store(true, std::memory_order_release);  // A

// Thread 2
while (!flag.load(std::memory_order_acquire));  // B
assert(data == 42);  // OK: A synchronizes-with B
```

**3. Transitivity:**
If A happens-before B and B happens-before C, then A happens-before C.

**Chain example:**
```cpp
std::atomic<int> a{0}, b{0};
int x = 0, y = 0;

// Thread 1
x = 1;
a.store(1, std::memory_order_release);

// Thread 2
while (a.load(std::memory_order_acquire) == 0);
y = 1;
b.store(1, std::memory_order_release);

// Thread 3
while (b.load(std::memory_order_acquire) == 0);
assert(x == 1);  // OK: transitivity
```

---

