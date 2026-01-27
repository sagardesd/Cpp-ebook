# C++ Interview Questions - Document 4 (Questions 91-120)

## Modern C++ Attributes (Questions 91-93 continued)

### 91. **How does `[[maybe_unused]]` differ from commenting out warnings?**

`[[maybe_unused]]` is a standard, portable attribute that suppresses unused warnings.

**Problem:**
```cpp
void debug_function([[maybe_unused]] int debug_value) {
    #ifdef DEBUG
        std::cout << debug_value;
    #endif
    // Without [[maybe_unused]], warning in release builds
}
```

**Compared to alternatives:**
```cpp
// Old way - cast to void
void func(int value) {
    (void)value;  // Suppress warning
}

// Better - [[maybe_unused]]
void func([[maybe_unused]] int value) {
    // Standard, clear intent
}
```

**Use cases:**
```cpp
// Conditional compilation
void process([[maybe_unused]] const Config& config) {
    #ifdef FEATURE_X
        use(config);
    #endif
}

// Lambda captures
auto lambda = [&data = some_data]() [[maybe_unused]] {
    // data might not be used
};

// Function parameters in derived classes
struct Base {
    virtual void func(int x) = 0;
};

struct Derived : Base {
    void func([[maybe_unused]] int x) override {
        // Might not use x in this implementation
    }
};
```

---

### 92. **Can you combine multiple attributes on the same declaration?**

Yes, multiple attributes can be combined in any order.

```cpp
[[nodiscard]] [[deprecated("use new_func instead")]]
int old_func();

[[nodiscard("check return value")]] [[maybe_unused]]
bool process();

class [[deprecated]] [[nodiscard]] OldClass {
    // Both attributes apply
};
```

**Order doesn't matter:**
```cpp
[[deprecated]] [[nodiscard]] int func1();
[[nodiscard]] [[deprecated]] int func2();
// Both equivalent
```

**Multiple attribute lists:**
```cpp
[[nodiscard]] [[deprecated]]
int func();

// Same as:
[[nodiscard, deprecated]]
int func();
```

**With vendor attributes:**
```cpp
[[gnu::always_inline]] [[nodiscard]]
int fast_func();

[[msvc::noinline]] [[nodiscard]]
int slow_func();
```

---

### 93. **What is `[[assume]]` (C++23) and how does it help optimization?**

`[[assume]]` tells compiler to assume a condition is true without runtime check. Undefined behavior if violated.

```cpp
void process(int* ptr) {
    [[assume(ptr != nullptr)]];
    *ptr = 42;  // Compiler can skip null check
}
```

**Optimization example:**
```cpp
int divide(int a, int b) {
    [[assume(b != 0)]];
    return a / b;  // Compiler can skip division-by-zero check
}
```

**With ranges:**
```cpp
void process(int x) {
    [[assume(x >= 0 && x < 100)]];
    // Compiler can optimize assuming x in [0, 100)
    
    int arr[100];
    arr[x] = 0;  // No bounds check needed
}
```

**Danger - undefined behavior if wrong:**
```cpp
void func(int* ptr) {
    [[assume(ptr != nullptr)]];
    *ptr = 42;
}

func(nullptr);  // ⚠️ Undefined behavior!
```

**Compared to assert:**
```cpp
// assert: runtime check (debug builds)
assert(ptr != nullptr);
*ptr = 42;

// [[assume]]: compiler optimization, no runtime check
[[assume(ptr != nullptr)]];
*ptr = 42;  // UB if ptr is null
```

---

## Lambda Expressions Advanced (Questions 94-100)

### 94. **Why do lambdas capture by value as `const` by default? When do you need `mutable`?**

Lambda's `operator()` is `const` by default, so captured-by-value variables are `const`.

**Problem:**
```cpp
int counter = 0;
auto lambda = [counter]() {
    counter++;  // ❌ Error: counter is const
};
```

**Solution - mutable:**
```cpp
int counter = 0;
auto lambda = [counter]() mutable {
    counter++;  // ✅ OK
    return counter;
};

std::cout << lambda();  // 1
std::cout << lambda();  // 2
std::cout << counter;   // Still 0 (original unchanged)
```

**Why const by default:**
Prevents accidental mutation of copies, making lambdas behave like functions.

**Reference captures can always modify:**
```cpp
int counter = 0;
auto lambda = [&counter]() {
    counter++;  // ✅ OK without mutable
};

lambda();
std::cout << counter;  // 1 (original modified)
```

**Use cases for mutable:**
```cpp
// Stateful lambda
auto accumulator = [sum = 0](int x) mutable {
    sum += x;
    return sum;
};

std::cout << accumulator(5);   // 5
std::cout << accumulator(10);  // 15

// Generator
auto counter = [n = 0]() mutable {
    return n++;
};

std::cout << counter();  // 0
std::cout << counter();  // 1
```

---

### 95. **Explain the difference between capturing `[=]`, `[&]`, `[this]`, and `[*this]` (C++17)**

**`[=]` - Capture all used locals by value:**
```cpp
int x = 10, y = 20;
auto lambda = [=]() {
    return x + y;  // Copies x and y
};
```

**`[&]` - Capture all used locals by reference:**
```cpp
int x = 10, y = 20;
auto lambda = [&]() {
    x++;  // Modifies original x
    return x + y;
};
```

**`[this]` - Capture this pointer:**
```cpp
class Widget {
    int value_ = 42;
public:
    auto getLambda() {
        return [this]() {
            return value_;  // Accesses member via this
        };
    }
};
```

**`[*this]` - Capture copy of entire object (C++17):**
```cpp
class Widget {
    int value_ = 42;
public:
    auto getLambda() {
        return [*this]() {
            return value_;  // Copy of entire Widget
        };
    }
};
```

**Danger of `[this]` in async:**
```cpp
class Widget {
public:
    void async_operation() {
        std::thread([this]() {
            // ⚠️ Dangerous! Widget might be destroyed
            use(member_);
        }).detach();
    }
};

// Better - copy the object
void async_operation() {
    std::thread([*this]() {
        // ✅ Safe - has own copy
        use(member_);
    }).detach();
}
```

**Mixing captures:**
```cpp
int x = 10, y = 20;
auto lambda = [=, &y]() {
    // x by value, y by reference
    y = x + 5;
};
```

---

### 96. **What is a stateless lambda and why is it convertible to function pointer?**

Stateless lambda (empty capture) has no state and can convert to function pointer.

**Stateless lambda:**
```cpp
auto lambda = [](int x) {
    return x * 2;
};

// Convertible to function pointer
int (*func_ptr)(int) = lambda;  // ✅ OK

func_ptr(5);  // 10
```

**With captures - not convertible:**
```cpp
int factor = 2;
auto lambda = [factor](int x) {
    return x * factor;
};

// int (*func_ptr)(int) = lambda;  // ❌ Error: has state
```

**Use with C APIs:**
```cpp
// C library function
extern "C" {
    void register_callback(void (*callback)(int));
}

// Can pass stateless lambda
register_callback([](int value) {
    std::cout << value;
});

// Can't pass lambda with captures
int state = 0;
register_callback([state](int value) {  // ❌ Error
    std::cout << value + state;
});
```

**Workaround for stateful - use context pointer:**
```cpp
extern "C" {
    void register_callback(void (*callback)(int, void*), void* context);
}

int state = 42;
register_callback(
    [](int value, void* ctx) {
        int* state = static_cast<int*>(ctx);
        std::cout << value + *state;
    },
    &state
);
```

---

### 97. **Explain init-capture (generalized lambda capture) and move-only captures**

Init-capture (C++14) allows capturing with initialization, enabling move captures.

**Basic init-capture:**
```cpp
int x = 10;
auto lambda = [y = x + 5]() {
    return y;  // Captured as y = 15
};
```

**Move-only captures:**
```cpp
auto ptr = std::make_unique<int>(42);

auto lambda = [ptr = std::move(ptr)]() {
    return *ptr;  // Moved into lambda
};

// ptr is now nullptr
```

**Real-world examples:**

**Moving large objects:**
```cpp
std::vector<int> large_vec(1000000);

auto lambda = [vec = std::move(large_vec)]() {
    return vec.size();  // Moved, not copied
};
```

**Async operations:**
```cpp
auto data = std::make_unique<Data>(/*...*/);

std::thread([data = std::move(data)]() {
    process(*data);  // Owns the data
}).detach();
```

**Capture thread:**
```cpp
std::thread worker(/*...*/);

auto lambda = [t = std::move(worker)]() mutable {
    t.join();  // Lambda owns the thread
};
```

**Complex initialization:**
```cpp
auto lambda = [
    result = expensive_computation(),
    file = std::ifstream("data.txt"),
    count = 0
]() mutable {
    // Use initialized captures
    count++;
    return result + count;
};
```

**Renaming captures:**
```cpp
int external_name = 42;

auto lambda = [internal = external_name]() {
    return internal * 2;  // Use different name inside
};
```

---

### 98. **How do generic lambdas (C++14) differ from template functions?**

Generic lambdas use `auto` parameters, creating templated `operator()`.

**Generic lambda:**
```cpp
auto lambda = [](auto x, auto y) {
    return x + y;
};

lambda(1, 2);      // int + int
lambda(1.5, 2.5);  // double + double
lambda(std::string("hi"), std::string("there"));  // string + string
```

**Equivalent template function:**
```cpp
template<typename T, typename U>
auto func(T x, U y) {
    return x + y;
}
```

**Can't explicitly specify types (before C++20):**
```cpp
auto lambda = [](auto x) { return x * 2; };

lambda(5);       // OK
lambda<int>(5);  // ❌ Error: can't specify template args
```

**C++20 template lambdas:**
```cpp
auto lambda = []<typename T>(T x) {
    if constexpr (std::is_integral_v<T>) {
        return x * 2;
    } else {
        return x;
    }
};

lambda.operator()<int>(5);  // Can specify type in C++20
```

**Perfect forwarding:**
```cpp
auto forward_lambda = [](auto&&... args) {
    return func(std::forward<decltype(args)>(args)...);
};
```

**With `decltype`:**
```cpp
auto lambda = [](auto x) -> decltype(x * 2) {
    return x * 2;
};
```

---

### 99. **What is the lifetime of lambda captures and what are the dangers?**

**Captured by value - independent lifetime:**
```cpp
auto createLambda() {
    int x = 42;
    return [x]() { return x; };  // ✅ Safe - x is copied
}

auto lambda = createLambda();
lambda();  // OK - has its own copy
```

**Captured by reference - dangling references:**
```cpp
auto createLambda() {
    int x = 42;
    return [&x]() { return x; };  // ❌ Dangerous!
}

auto lambda = createLambda();
lambda();  // ⚠️ Undefined behavior - x destroyed
```

**Capturing `this` - object lifetime:**
```cpp
class Widget {
    int value_ = 42;
public:
    auto getLambda() {
        return [this]() { return value_; };  // ⚠️ Stores this pointer
    }
};

auto lambda = Widget().getLambda();
lambda();  // ⚠️ UB - Widget destroyed
```

**Safe version with `[*this]`:**
```cpp
auto getLambda() {
    return [*this]() { return value_; };  // ✅ Copies entire object
}

auto lambda = Widget().getLambda();
lambda();  // ✅ Safe
```

**Async operations danger:**
```cpp
void process() {
    std::string data = "important";
    
    std::thread([&data]() {
        // ⚠️ data might be destroyed before thread runs
        std::cout << data;
    }).detach();
}  // data destroyed, thread has dangling reference

// Safe version:
void process() {
    std::string data = "important";
    
    std::thread([data]() {
        // ✅ data is copied
        std::cout << data;
    }).detach();
}
```

---

### 100. **Explain immediately-invoked lambda expressions (IIFE) and their use cases**

IIFE: Lambda defined and called immediately.

**Basic syntax:**
```cpp
auto result = []() {
    // Complex initialization
    int temp = compute();
    temp = transform(temp);
    return temp;
}();  // Called immediately
```

**Use case 1 - Complex const initialization:**
```cpp
const int value = []() {
    if (condition) {
        return expensive_computation();
    } else {
        return default_value();
    }
}();

// Instead of:
int value;
if (condition) {
    value = expensive_computation();
} else {
    value = default_value();
}
```

**Use case 2 - Scoped temporaries:**
```cpp
auto result = [&]() {
    std::vector<int> temp = load_data();
    std::sort(temp.begin(), temp.end());
    return process(temp);
}();
// temp doesn't pollute outer scope
```

**Use case 3 - One-time initialization:**
```cpp
static const auto config = []() {
    Config c;
    c.load("settings.ini");
    c.validate();
    return c;
}();
```

**Use case 4 - Complex member initialization:**
```cpp
class Widget {
    const int value_ = []() {
        int x = compute_base();
        x = adjust(x);
        return validate(x);
    }();
};
```

**Use case 5 - Exception-safe cleanup:**
```cpp
void func() {
    Resource* r = acquire();
    
    auto cleanup = [&]() {
        release(r);
    };
    
    try {
        use(r);
    } catch (...) {
        cleanup();
        throw;
    }
    cleanup();
}

// Better with RAII, but IIFE can work too
```

---

## Virtual Functions & OOP Deep Dive (Questions 101-107)

### 101. **Why is it important to make destructors virtual in base classes?**

Without virtual destructor, deleting derived object through base pointer only calls base destructor.

**Problem:**
```cpp
class Base {
public:
    ~Base() { std::cout << "~Base\n"; }
};

class Derived : public Base {
    int* data_;
public:
    Derived() : data_(new int[100]) {}
    ~Derived() {
        std::cout << "~Derived\n";
        delete[] data_;  // Clean up
    }
};

Base* ptr = new Derived();
delete ptr;  // ❌ Only calls ~Base! Memory leak!
// Output: ~Base (Derived destructor never called!)
```

**Solution:**
```cpp
class Base {
public:
    virtual ~Base() { std::cout << "~Base\n"; }
};

class Derived : public Base {
    int* data_;
public:
    Derived() : data_(new int[100]) {}
    ~Derived() override {
        std::cout << "~Derived\n";
        delete[] data_;
    }
};

Base* ptr = new Derived();
delete ptr;  // ✅ Calls both destructors
// Output: ~Derived
//         ~Base
```

**Rule:** If class has ANY virtual function, destructor should be virtual.

**Modern C++:**
```cpp
class Base {
public:
    virtual ~Base() = default;  // Virtual and defaulted
};
```

**When NOT to make virtual:**
```cpp
class NotIntendedAsBase {
    // No virtual functions
    // Not meant for polymorphism
public:
    ~NotIntendedAsBase() {}  // Non-virtual OK
};
```

---

### 102. **What is the performance cost of virtual functions?**

Virtual functions have overhead from indirect calls through vtable.

**Costs:**

**1. Vtable pointer per object:**
```cpp
class NoVirtual {
    int x;  // 4 bytes
};

class WithVirtual {
    int x;  // 4 bytes
    virtual void func() {}  // +8 bytes (vtable pointer)
};

sizeof(NoVirtual);    // 4
sizeof(WithVirtual);  // 16 (4 + 8 for vptr + padding)
```

**2. Indirect call - not inlineable:**
```cpp
// Non-virtual - direct call, can inline
void func(Widget& w) {
    w.compute();  // Direct call, inlined
}

// Virtual - indirect call through vtable
void func(Base& b) {
    b.compute();  // Vtable lookup, can't inline
}
```

**3. Cache misses:**
```cpp
// Virtual function call:
// 1. Load vtable pointer from object
// 2. Load function pointer from vtable (possible cache miss)
// 3. Call function through pointer (branch prediction harder)
```

**Benchmark:**
```cpp
// Non-virtual: ~1ns per call
// Virtual: ~3-5ns per call (3-5x slower)

// 1 billion calls:
// Non-virtual: 1 second
// Virtual: 3-5 seconds
```

**Mitigation - `final`:**
```cpp
class Derived final : public Base {
    void func() override {
        // Compiler knows no further overrides
        // Can devirtualize
    }
};

void process(Derived& d) {
    d.func();  // Can be inlined (Derived is final)
}
```

**When it matters:**
```cpp
// Tight loops - significant impact
for (int i = 0; i < 1000000; ++i) {
    obj.virtual_func();  // 3-5 million ns overhead
}

// I/O bound code - negligible impact
for (auto& obj : objects) {
    obj.virtual_func();
    write_to_disk(obj);  // I/O dominates
}
```

---

### 103. **Explain pure virtual functions and abstract classes. Can abstract classes have constructors?**

Pure virtual function: `virtual void func() = 0;` makes class abstract.

**Abstract class:**
```cpp
class Shape {  // Abstract
public:
    virtual double area() const = 0;  // Pure virtual
    virtual void draw() const = 0;
    virtual ~Shape() = default;
};

// Shape s;  // ❌ Error: can't instantiate abstract class
```

**Abstract classes CAN have constructors:**
```cpp
class Shape {
protected:
    std::string name_;
    
public:
    Shape(std::string name) : name_(std::move(name)) {}
    virtual double area() const = 0;
};

class Circle : public Shape {
    double radius_;
public:
    Circle(double r) : Shape("Circle"), radius_(r) {}
    double area() const override { return 3.14 * radius_ * radius_; }
};

Circle c(5.0);  // Shape constructor called first
```

**Pure virtual with implementation:**
```cpp
class Base {
public:
    virtual void func() = 0;  // Pure virtual
};

// Can provide default implementation
void Base::func() {
    std::cout << "Base implementation\n";
}

class Derived : public Base {
public:
    void func() override {
        Base::func();  // Can call base implementation
        std::cout << "Derived additions\n";
    }
};
```

**Use case - abstract interface with partial implementation:**
```cpp
class Document {
public:
    virtual void save() = 0;  // Must override
    
    virtual void print() {  // Has default
        std::cout << "Printing: " << getTitle() << "\n";
    }
    
protected:
    virtual std::string getTitle() const = 0;
};
```

---

### 104. **What is the difference between `override` and `final` specifiers?**

**`override` - Verifies function overrides base virtual:**
```cpp
class Base {
public:
    virtual void func(int x) {}
};

class Derived : public Base {
public:
    void func(int x) override {}  // ✅ OK - overrides Base::func
    
    void func(double x) override {}  // ❌ Error: no matching virtual function
    
    void other() override {}  // ❌ Error: not in base
};
```

**`final` - Prevents further overriding:**
```cpp
class Base {
public:
    virtual void func() {}
};

class Middle : public Base {
public:
    void func() override final {}  // Can't override further
};

class Derived : public Middle {
public:
    void func() override {}  // ❌ Error: can't override final
};
```

**Class `final` - Prevents derivation:**
```cpp
class Base final {
    // Can't derive from this
};

class Derived : public Base {};  // ❌ Error
```

**Benefits of `override`:**
```cpp
// Without override
class Derived : public Base {
    void func(int x) {}  // Silent bug if signature doesn't match
};

// With override
class Derived : public Base {
    void func(int x) override {}  // Compiler error if wrong
};
```

**Benefits of `final`:**
```cpp
class Optimized : public Base {
    void compute() override final {}
};

void process(Optimized& obj) {
    obj.compute();  // Compiler can devirtualize (knows it's final)
}
```

---

### 105. **Can you override a non-virtual function? What happens?**

Yes syntactically, but it's **hiding**, not overriding. Causes confusion.

**Hiding example:**
```cpp
class Base {
public:
    void func() { std::cout << "Base::func\n"; }
};

class Derived : public Base {
public:
    void func() { std::cout << "Derived::func\n"; }  // Hides Base::func
};

Derived d;
d.func();  // Derived::func

Base* ptr = &d;
ptr->func();  // Base::func (NOT polymorphic!)
```

**Problem:**
```cpp
void process(Base& obj) {
    obj.func();  // Always calls Base::func
}

Derived d;
process(d);  // Prints "Base::func", not "Derived::func"
```

**`override` catches this:**
```cpp
class Derived : public Base {
public:
    void func() override {}  // ❌ Error: Base::func not virtual
};
```

**Correct approach:**
```cpp
class Base {
public:
    virtual void func() { std::cout << "Base::func\n"; }
};

class Derived : public Base {
public:
    void func() override { std::cout << "Derived::func\n"; }
};

Base* ptr = new Derived();
ptr->func();  // Derived::func (polymorphic!)
```

---

### 106. **Explain covariant return types in virtual functions**

Overriding function can return pointer/reference to derived type.

**Example:**
```cpp
class Base {};
class Derived : public Base {};

class Factory {
public:
    virtual Base* create() {
        return new Base();
    }
};

class DerivedFactory : public Factory {
public:
    Derived* create() override {  // Covariant return type
        return new Derived();
    }
};
```

**Why it's useful:**
```cpp
class Animal {
public:
    virtual Animal* clone() const {
        return new Animal(*this);
    }
};

class Dog : public Animal {
public:
    Dog* clone() const override {  // Returns Dog*, not Animal*
        return new Dog(*this);
    }
};

Dog d;
Dog* d2 = d.clone();  // ✅ No cast needed!
```

**Requirements:**
1. Return type must be pointer or reference
2. Derived type must be derived from base return type
3. Both must have same cv-qualifiers

**Doesn't work with value types:**
```cpp
class Base {
public:
    virtual Base create() { return Base(); }
};

class Derived : public Base {
public:
    Derived create() override { return Derived(); }  // ❌ Error
};
```

**Smart pointer example:**
```cpp
class Factory {
public:
    virtual std::unique_ptr<Base> create() {
        return std::make_unique<Base>();
    }
};

class DerivedFactory : public Factory {
public:
    // ❌ Doesn't work with unique_ptr (not raw pointer)
    std::unique_ptr<Derived> create() override;
};
```

---

### 107. **What is the "slicing problem" and how do you prevent it?**

Slicing: Assigning derived object to base variable copies only base portion.

**Problem:**
```cpp
class Base {
public:
    int x = 1;
    virtual void identify() { std::cout << "Base\n"; }
};

class Derived : public Base {
public:
    int y = 2;
    void identify() override { std::cout << "Derived\n"; }
};

Derived d;
Base b = d;  // ⚠️ Slicing! y is lost

b.identify();  // Prints "Base" (not polymorphic)
std::cout << sizeof(b);  // Size of Base, not Derived
```

**Why it happens:**
```cpp
// Copy constructor called
Base b = d;  // Base::Base(const Base& other)
             // Only copies Base portion
```

**Prevention 1 - Use pointers/references:**
```cpp
Derived d;
Base& ref = d;  // ✅ No slicing
ref.identify();  // Prints "Derived" (polymorphic)

Base* ptr = &d;  // ✅ No slicing
ptr->identify();  // Prints "Derived"
```

**Prevention 2 - Make base class abstract:**
```cpp
class Base {
public:
    virtual void identify() = 0;  // Pure virtual
};

// Base b = d;  // ❌ Error: can't instantiate Base
```

**Prevention 3 - Delete copy operations:**
```cpp
class Base {
public:
    Base(const Base&) = delete;
    Base& operator=(const Base&) = delete;
    virtual void identify() { }
};

// Base b = d;  // ❌ Error: copy constructor deleted
```

**Prevention 4 - Use smart pointers:**
```cpp
std::unique_ptr<Base> ptr = std::make_unique<Derived>();
// Can't accidentally slice with smart pointers
```

**Subtle slicing in containers:**
```cpp
std::vector<Base> vec;
Derived d;
vec.push_back(d);  // ⚠️ Slicing!

// Better:
std::vector<std::unique_ptr<Base>> vec;
vec.push_back(std::make_unique<Derived>());  // ✅ No slicing
```

---

## Scoped Enums & Casting (Questions 108-114)

### 108. **What are the advantages of `enum class` over traditional `enum`?**

**Scoped enums (C++11):**

**1. No implicit conversion to int:**
```cpp
enum class Color { Red, Green, Blue };

Color c = Color::Red;
int x = c;  // ❌ Error: no implicit conversion

int y = static_cast<int>(c);  // ✅ Explicit cast required
```

**Traditional enum - implicit conversion:**
```cpp
enum Color { Red, Green, Blue };

Color c = Red;
int x = c;  // ✅ Implicitly converts
```

**2. Scoped names (no pollution):**
```cpp
enum class Color { Red, Green, Blue };
enum class TrafficLight { Red, Yellow, Green };

Color c = Color::Red;  // ✅ OK
TrafficLight t = TrafficLight::Red;  // ✅ OK
```

**Traditional enum - name collision:**
```cpp
enum Color { Red, Green, Blue };
enum TrafficLight { Red, Yellow, Green };  // ❌ Error: Red redefined
```

**3. Can specify underlying type:**
```cpp
enum class Status : uint8_t {
    OK,
    Error,
    Pending
};

sizeof(Status);  // 1 byte
```

**4. Forward-declarable:**
```cpp
// Header
enum class Color : int;  // Forward declaration

// Implementation
enum class Color : int {
    Red, Green, Blue
};
```

**Traditional enum:**
```cpp
enum Color;  // ❌ Error: can't forward declare (C++11 allows with type)
```

**When to use traditional enum:**
```cpp
// Bit flags (want implicit conversion)
enum Flags {
    Read = 1,
    Write = 2,
    Execute = 4
};

int permissions = Read | Write;  // Natural with traditional enum
```

---

### 109. **How do you convert between scoped enum and integer types?**

Explicit cast required in both directions.

**Enum to int:**
```cpp
enum class Color { Red, Green, Blue };

Color c = Color::Red;

int x = static_cast<int>(c);  // ✅ Explicit cast
// int y = c;  // ❌ Error: no implicit conversion
```

**Int to enum:**
```cpp
int x = 1;

Color c = static_cast<Color>(x);  // ✅ Explicit cast
// Color c2 = x;  // ❌ Error: no implicit conversion
```

**Using `std::underlying_type`:**
```cpp
enum class Color : uint8_t { Red, Green, Blue };

using ColorInt = std::underlying_type_t<Color>;
ColorInt x = static_cast<ColorInt>(Color::Red);  // uint8_t
```

**Helper functions:**
```cpp
template<typename Enum>
constexpr auto to_underlying(Enum e) {
    return static_cast<std::underlying_type_t<Enum>>(e);
}

auto x = to_underlying(Color::Red);  // Cleaner
```

**Bitwise operations:**
```cpp
enum class Flags : uint32_t {
    Read = 1,
    Write = 2,
    Execute = 4
};

// Need operator overloads for flags
Flags operator|(Flags a, Flags b) {
    return static_cast<Flags>(
        to_underlying(a) | to_underlying(b)
    );
}

auto permissions = Flags::Read | Flags::Write;
```

---

### 110. **Explain the four types of C++ casts and when to use each**

**`static_cast` - Compile-time conversions:**
```cpp
// Numeric conversions
double d = 3.14;
int i = static_cast<int>(d);  // 3

// Up-cast (derived to base) - safe
Derived* d = new Derived();
Base* b = static_cast<Base*>(d);  // ✅ Safe

// Down-cast (base to derived) - unsafe without check
Base* b = new Base();
Derived* d = static_cast<Derived*>(b);  // ⚠️ Unsafe!

// Enum conversions
enum class Color { Red };
int x = static_cast<int>(Color::Red);
```

**`dynamic_cast` - Runtime polymorphic casting:**
```cpp
Base* b = new Derived();

// Safe downcast with runtime check
Derived* d = dynamic_cast<Derived*>(b);
if (d) {
    // Success
} else {
    // Failed - returns nullptr
}

// With references (throws on failure)
try {
    Base& b_ref = *b;
    Derived& d_ref = dynamic_cast<Derived&>(b_ref);
} catch (std::bad_cast&) {
    // Failed
}
```

**`const_cast` - Add/remove const:**
```cpp
void legacy_func(char* str);  // Non-const API

const char* str = "hello";
legacy_func(const_cast<char*>(str));  // Remove const

// Add const
char* mutable_str = get_string();
const char* immutable = const_cast<const char*>(mutable_str);
```

**`reinterpret_cast` - Reinterpret bit pattern:**
```cpp
// Pointer to int (low-level)
int* p = new int(42);
uintptr_t addr = reinterpret_cast<uintptr_t>(p);

// Type punning (dangerous!)
float f = 3.14f;
uint32_t bits = *reinterpret_cast<uint32_t*>(&f);

// Pointer conversions
void* generic_ptr = malloc(100);
char* char_ptr = reinterpret_cast<char*>(generic_ptr);
```

**When to use:**
```cpp
// static_cast: Most conversions you control
double d = static_cast<double>(i);

// dynamic_cast: Polymorphic type checking
if (auto* derived = dynamic_cast<Derived*>(base)) { }

// const_cast: Interfacing with incorrect const APIs
legacy_api(const_cast<char*>(const_string));

// reinterpret_cast: Low-level operations, rarely needed
auto addr = reinterpret_cast<uintptr_t>(ptr);
```

---

### 111. **What is RTTI and when is `dynamic_cast` safe?**

RTTI (Runtime Type Information): Enables `dynamic_cast` and `typeid`.

**Requires virtual functions:**
```cpp
class Base {  // No virtual functions
    int x;
};

class Derived : public Base { };

Base* b = new Derived();
// dynamic_cast<Derived*>(b);  // ❌ Error: not polymorphic
```

**With virtual functions:**
```cpp
class Base {
public:
    virtual ~Base() {}  // Now polymorphic
};

class Derived : public Base { };

Base* b = new Derived();
Derived* d = dynamic_cast<Derived*>(b);  // ✅ OK
```

**When `dynamic_cast` is safe:**

**1. Polymorphic type (has virtual):**
```cpp
if (auto* derived = dynamic_cast<Derived*>(base)) {
    // Safe - checked at runtime
}
```

**2. Check result:**
```cpp
Derived* d = dynamic_cast<Derived*>(base);
if (d) {
    // Success
} else {
    // Failed - base is not actually Derived
}
```

**3. Catch exception (references):**
```cpp
try {
    Derived& d = dynamic_cast<Derived&>(base_ref);
} catch (std::bad_cast&) {
    // Failed
}
```

**Performance cost:**
```cpp
// dynamic_cast walks vtable hierarchy
// Slower than static_cast
// Use only when needed for type safety
```

**Alternative - virtual functions:**
```cpp
// Instead of:
if (auto* dog = dynamic_cast<Dog*>(animal)) {
    dog->bark();
}

// Better design:
class Animal {
    virtual void makeSound() = 0;
};

animal->makeSound();  // Polymorphism, no cast needed
```

**When to use RTTI:**
```cpp
// Visitor pattern
// Type-specific behavior unavoidable
// Plugin systems
// Debugging/logging
```

---

### 112. **Explain const-correctness and the different types of const member functions**

**Const member function:** Can't modify members.

```cpp
class Widget {
    int value_;
public:
    int getValue() const {  // Const member function
        // value_ = 10;  // ❌ Error: can't modify
        return value_;  // ✅ OK
    }
    
    void setValue(int v) {  // Non-const
        value_ = v;  // ✅ Can modify
    }
};

const Widget w;
w.getValue();   // ✅ OK
// w.setValue(10);  // ❌ Error: can't call non-const on const object
```

**Const return type:**
```cpp
class Widget {
public:
    const int& getValue() const {  // Returns const reference
        return value_;
    }
};

const Widget w;
// w.getValue() = 10;  // ❌ Error: can't modify through const ref
```

**Bitwise const vs logical const:**

**Bitwise const:** All members physically unchanged.
```cpp
class Counter {
    int count_;
public:
    int getCount() const {
        // count_++;  // ❌ Error: modifies member
        return count_;
    }
};
```

**Logical const:** Use `mutable` for implementation details.
```cpp
class Widget {
    mutable int access_count_;  // Mutable
    int value_;
    
public:
    int getValue() const {
        access_count_++;  // ✅ OK - mutable
        return value_;
    }
};
```

**Const overloading:**
```cpp
class Buffer {
    char* data_;
public:
    char& operator[](size_t i) {  // Non-const version
        return data_[i];  // Returns modifiable reference
    }
    
    const char& operator[](size_t i) const {  // Const version
        return data_[i];  // Returns const reference
    }
};

Buffer b;
b[0] = 'x';  // ✅ Calls non-const version

const Buffer cb;
// cb[0] = 'x';  // ❌ Error
char c = cb[0];  // ✅ Calls const version
```

---

### 113. **What is `mutable` keyword and when would you use it?**

`mutable` allows member modification in const member functions.

**Use cases:**

**1. Caching:**
```cpp
class ExpensiveCalculation {
    mutable int cached_result_;
    mutable bool cached_;
    
public:
    int compute() const {
        if (!cached_) {
            cached_result_ = expensive_operation();  // ✅ Mutable
            cached_ = true;
        }
        return cached_result_;
    }
};
```

**2. Mutexes:**
```cpp
class ThreadSafeCounter {
    mutable std::mutex mutex_;
    int count_;
    
public:
    int getCount() const {
        std::lock_guard lock(mutex_);  // ✅ Mutable mutex
        return count_;
    }
};
```

**3. Reference counts:**
```cpp
class SharedData {
    mutable int ref_count_;
    Data data_;
    
public:
    void addRef() const {
        ++ref_count_;  // ✅ Mutable
    }
};
```

**4. Debug/logging:**
```cpp
class Service {
    mutable int call_count_;  // Debug counter
    
public:
    void process() const {
        ++call_count_;  // Track calls without breaking const
        // ... actual work ...
    }
};
```

**When NOT to use:**
```cpp
class Bad {
    mutable int important_state_;  // ❌ Don't abuse!
    
public:
    void modify() const {
        important_state_ = 42;  // ❌ Violates const contract
    }
};
```

---

### 114. **Can you `const_cast` away const and modify the object?**

**Depends on original object:**

**If originally non-const - OK:**
```cpp
int x = 42;
const int& ref = x;
const_cast<int&>(ref) = 100;  // ✅ OK - x was non-const
```

**If originally const - UB:**
```cpp
const int x = 42;
const_cast<int&>(x) = 100;  // ⚠️ Undefined behavior!
```

**Use case - incorrect API:**
```cpp
// Legacy C API with wrong const
void legacy_func(char* str);  // Should be const char*

const std::string s = "hello";
legacy_func(const_cast<char*>(s.c_str()));  // Only if func doesn't modify
```

**Don't use to "fix" const-correctness:**
```cpp
class Bad {
public:
    void modify(const Widget& w) {
        // ❌ Don't do this!
        const_cast<Widget&>(w).change();
    }
};

// Fix the const-correctness properly instead
class Good {
public:
    void modify(Widget& w) {  // ✅ Correct signature
        w.change();
    }
};
```

**Compiler optimization impact:**
```cpp
const int x = 42;
const_cast<int&>(x) = 100;  // UB!
// Compiler may optimize based on x being const
// May not see the change
```

---

## Advanced Edge Cases & Best Practices (Questions 115-120)

### 115. **What is the difference between `nullptr`, `NULL`, and `0` in C++?**

**`nullptr` (C++11):** Type-safe null pointer literal.
```cpp
void func(int x) { std::cout << "int\n"; }
void func(char* p) { std::cout << "pointer\n"; }

func(nullptr);  // ✅ Calls pointer version
func(NULL);     // ⚠️ Might call int version (NULL is typically 0)
func(0);        // Calls int version
```

**`NULL`:** Macro, typically `0` or `((void*)0)`.
```cpp
#define NULL 0  // Or ((void*)0) in C

int* p = NULL;  // Works, but NULL is integer
```

**`0`:** Integer literal.
```cpp
int* p = 0;  // Works due to special rule
```

**Why `nullptr` is better:**
```cpp
template<typename T>
void process(T t) {
    // Can check if t is nullptr
}

process(nullptr);  // T is std::nullptr_t
process(NULL);     // T is int (usually)

// Type safety
auto x = nullptr;  // std::nullptr_t
auto y = NULL;     // int

// Overload resolution
void func(int);
void func(void*);

func(nullptr);  // ✅ Unambiguous - calls pointer version
func(NULL);     // ⚠️ Ambiguous or calls int version
```

---

### 116. **Explain the difference between `delete` and `delete[]`. What happens if you mix them?**

**`delete`:** Deallocates single object, calls one destructor.
**`delete[]`:** Deallocates array, calls destructor for each element.

```cpp
Widget* single = new Widget();
delete single;  // ✅ Correct

Widget* array = new Widget[10];
delete[] array;  // ✅ Correct - calls 10 destructors
```

**Mixing causes UB:**
```cpp
Widget* array = new Widget[10];
delete array;  // ❌ UB! Only one destructor called, heap corruption

Widget* single = new Widget();
delete[] single;  // ❌ UB! Reads array size from wrong location
```

**Modern solution - use RAII:**
```cpp
// Instead of:
Widget* array = new Widget[10];
// ... use ...
delete[] array;

// Use:
std::vector<Widget> vec(10);  // Automatic cleanup

// Or:
std::unique_ptr<Widget[]> array(new Widget[10]);
// Or:
auto array = std::make_unique<Widget[]>(10);
```

---

### 117. **What is the "as-if" rule in C++ optimization?**

Compiler can perform any optimization as long as observable behavior matches.

**Enables optimizations:**
```cpp
int x = 1;
int y = 2;
int z = x + y;
// Compiler can compute z = 3 at compile-time
```

**Constraints:**

**1. Sequence points respected:**
```cpp
int x = 0;
x = x + 1;  // Compiler can't reorder across sequence point
```

**2. Volatile access preserved:**
```cpp
volatile int* mmio = ...;
*mmio = 1;  // Must actually write, can't optimize away
```

**3. I/O side effects maintained:**
```cpp
std::cout << "Hello";  // Must actually output
```

**4. Observable behavior preserved:**
```cpp
int func() {
    int x = expensive();
    if (condition)
        return x;
    return 0;
}
// Compiler can skip expensive() if condition is false
```

**Examples:**
```cpp
// Dead code elimination
int x = 10;
x = 20;  // Compiler can eliminate first assignment
return x;

// Loop unrolling
for (int i = 0; i < 4; ++i)
    arr[i] = 0;
// Becomes:
arr[0] = 0; arr[1] = 0; arr[2] = 0; arr[3] = 0;

// Function inlining
inline int add(int a, int b) { return a + b; }
int x = add(1, 2);
// Becomes:
int x = 3;
```

---

### 118. **Explain name mangling and why `extern "C"` is needed**

Name mangling: Encodes function signature into symbol name for overloading.

**C++ name mangling:**
```cpp
void func(int);
void func(double);
void func(int, int);

// Mangled names (compiler-specific):
// _Z4funci
// _Z4funcd
// _Z4funcii
```

**C - no mangling:**
```c
void func(int);
// Symbol name: func
```

**`extern "C"` - disable mangling:**
```cpp
extern "C" {
    void c_function(int x);  // No mangling
}

// Or:
extern "C" void c_function(int x);
```

**Use cases:**

**1. C library interfaces:**
```cpp
extern "C" {
    #include <stdio.h>  // C library
}
```

**2. DLL exports:**
```cpp
extern "C" {
    __declspec(dllexport) void api_function();
}
```

**3. Plugin systems:**
```cpp
// Plugin interface
extern "C" {
    Plugin* create_plugin();
    void destroy_plugin(Plugin*);
}
```

**Limitations:**
```cpp
// Can't use overloading
extern "C" {
    void func(int);
    // void func(double);  // ❌ Error: overloading not allowed
}

// Can't use templates
extern "C" {
    // template<typename T> void func(T);  // ❌ Error
}
```

---

### 119. **What is aggregate initialization and how does it differ from list initialization?**

**Aggregate:** Struct/class with no user-declared constructors, no private members (C++20 relaxed).

**Aggregate initialization:**
```cpp
struct Point {
    int x, y;
};

Point p{10, 20};  // Aggregate initialization
Point p2 = {10, 20};  // Also aggregate
```

**With inheritance (C++17):**
```cpp
struct Base {
    int x;
};

struct Derived : Base {
    int y;
};

Derived d{{10}, 20};  // Base{10}, y = 20
```

**List initialization (general):**
```cpp
class Widget {
public:
    Widget(std::initializer_list<int> list) {
        // Called with {}
    }
    Widget(int, int) {
        // Also constructible
    }
};

Widget w1{1, 2};  // Calls initializer_list constructor
Widget w2(1, 2);  // Calls (int, int) constructor
```

**Aggregate rules (C++20):**
```cpp
struct Aggregate {
    int x;
private:
    int y;  // ✅ OK in C++20 (was not aggregate in C++17)
public:
    int z;
};

Aggregate a{1, 2, 3};  // C++20: OK
```

**Designated initializers (C++20):**
```cpp
struct Point {
    int x, y, z;
};

Point p{.x = 10, .y = 20, .z = 30};
Point p2{.x = 10, .z = 30};  // y is 0
```

---

### 120. **Explain the "zero-overhead principle" in C++ and give examples where it's violated**

**Principle:** You don't pay for what you don't use; used features can't be more efficient by hand.

**Examples following principle:**

**RAII:**
```cpp
std::unique_ptr<int> ptr(new int(42));
// Zero overhead compared to manual new/delete
```

**Templates:**
```cpp
template<typename T>
T max(T a, T b) { return a > b ? a : b; }
// Inlines to same code as manual comparison
```

**constexpr:**
```cpp
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n-1);
}
// Computed at compile-time, zero runtime cost
```

**Violations:**

**1. RTTI (vtable overhead):**
```cpp
class Base {
    virtual void func() {}  // Adds vtable pointer
};
// Costs 8 bytes per object even if typeid never used
```

**2. Exceptions (code bloat):**
```cpp
void func() {
    // Costs code size even if never throws
    // Unwind tables generated
}
```

**3. `std::function` (type erasure):**
```cpp
std::function<int(int)> f = [](int x) { return x * 2; };
// Heap allocation, virtual dispatch
// More expensive than raw function pointer
```

**4. `std::shared_ptr` (atomic operations):**
```cpp
std::shared_ptr<int> p = std::make_shared<int>(42);
std::shared_ptr<int> p2 = p;  // Atomic increment
// Can't be more efficient by hand (without atomics)
```

**5. Range-based for with copies:**
```cpp
std::vector<HugeObject> vec;
for (auto obj : vec) {  // Copies each object!
    // Should use: for (const auto& obj : vec)
}
```

**When to avoid:**
```cpp
// Embedded systems - disable exceptions/RTTI
// -fno-exceptions -fno-rtti

// Performance critical - avoid shared_ptr
// Use unique_ptr or raw pointers with clear ownership

// Avoid std::function in hot paths
// Use templates or function pointers
```

---

## Conclusion

These 120 questions cover the essential advanced C++ concepts for senior developer interviews:

- **Template Metaprogramming:** SFINAE, CRTP, variadic templates
- **Move Semantics:** Rvalue references, perfect forwarding, RVO
- **Concurrency:** Memory model, atomics, synchronization
- **Modern C++:** C++11/14/17/20 features
- **Smart Pointers & RAII:** Resource management
- **STL Internals:** Container behavior and performance
- **OOP:** Virtual functions, polymorphism, design patterns
- **Edge Cases:** Language quirks and best practices

Remember: Focus on understanding **WHY** features exist and **WHEN** to use them, not just HOW they work!

