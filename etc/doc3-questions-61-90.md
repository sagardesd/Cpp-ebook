# C++ Interview Questions - Document 3 (Questions 61-90)

## Obscure Language Features & Edge Cases (Questions 61-66 continued)

### 61. **Explain the empty base optimization (EBO)**

Empty base classes take zero size due to EBO.

**Without EBO:**
```cpp
struct Empty {};
struct Derived {
    Empty e;
    int x;
};

sizeof(Derived);  // 8 bytes (1 byte for Empty + padding + 4 for int)
```

**With EBO:**
```cpp
struct Empty {};
struct Derived : Empty {
    int x;
};

sizeof(Derived);  // 4 bytes! Empty base takes no space
```

**Why it matters - policy-based design:**
```cpp
template<typename Allocator>
class Vector : private Allocator {  // EBO applied
    int* data_;
    size_t size_;
public:
    void allocate(size_t n) {
        data_ = Allocator::allocate(n);  // Use allocator
    }
};

// If Allocator is empty, no size overhead!
```

**C++20 `[[no_unique_address]]` extends to members:**
```cpp
struct Empty {};

struct S {
    [[no_unique_address]] Empty e;
    int x;
};

sizeof(S);  // 4 bytes! Member optimized too
```

**Example:**
```cpp
template<typename T, typename Allocator = std::allocator<T>>
class MyVector : private Allocator {
    T* data_;
    size_t size_;
    size_t capacity_;
    
public:
    void push_back(const T& value) {
        if (size_ == capacity_) {
            // Use inherited Allocator
            T* new_data = this->allocate(capacity_ * 2);
            // ...
        }
    }
};

// std::allocator is typically empty - EBO saves space
```

---

### 62. **What happens when you throw an exception from a destructor?**

Throwing from destructor during stack unwinding calls `std::terminate()`.

**Problem:**
```cpp
class BadWidget {
public:
    ~BadWidget() {
        throw std::runtime_error("Oops");  // ❌ Dangerous!
    }
};

void func() {
    BadWidget w;
    throw std::exception();  // Stack unwinding starts
}  // ~BadWidget called, throws again → std::terminate()!
```

**Why it's dangerous:**
```cpp
// If exception already active and destructor throws:
try {
    BadWidget w;
    throw std::runtime_error("First exception");
    // Stack unwinding → ~BadWidget throws
    // Two active exceptions → std::terminate()
} catch (...) {
    // Never reached
}
```

**Correct approach - catch in destructor:**
```cpp
class GoodWidget {
public:
    ~GoodWidget() noexcept {  // Implicitly noexcept since C++11
        try {
            cleanup();  // Might throw
        } catch (...) {
            // Log error, don't propagate
            std::cerr << "Cleanup failed\n";
        }
    }
};
```

**C++11 implicit noexcept:**
```cpp
// Destructors are implicitly noexcept
~Widget();  // Same as ~Widget() noexcept

// Explicit throwing (not recommended)
~Widget() noexcept(false);  // Can throw
```

---

### 63. **Explain name lookup and why `using` is sometimes needed in templates**

Two-phase lookup in templates: non-dependent names at definition time, dependent names at instantiation time.

**Problem:**
```cpp
template<typename T>
class Derived : public Base<T> {
public:
    void func() {
        value = 42;  // ❌ Error: 'value' not found
                     // Even though Base<T>::value exists!
    }
};
```

**Why:** `value` is dependent name (depends on `T`). Compiler doesn't look in Base until instantiation.

**Solution 1 - `this->`:**
```cpp
template<typename T>
class Derived : public Base<T> {
public:
    void func() {
        this->value = 42;  // ✅ OK
    }
};
```

**Solution 2 - `using` declaration:**
```cpp
template<typename T>
class Derived : public Base<T> {
    using Base<T>::value;  // Make name visible
public:
    void func() {
        value = 42;  // ✅ OK
    }
};
```

**Solution 3 - Fully qualify:**
```cpp
template<typename T>
class Derived : public Base<T> {
public:
    void func() {
        Base<T>::value = 42;  // ✅ OK
    }
};
```

**CRTP example:**
```cpp
template<typename Derived>
class Base {
public:
    void interface() {
        static_cast<Derived*>(this)->implementation();
    }
};

template<typename T>
class MyClass : public Base<MyClass<T>> {
    using Base<MyClass<T>>::interface;  // Make accessible
public:
    void implementation() { }
};
```

---

### 64. **What is the "copy-and-swap" idiom?**

Idiom for exception-safe assignment using swap.

**Implementation:**
```cpp
class Widget {
    int* data_;
    size_t size_;
    
public:
    Widget(const Widget& other) 
        : data_(new int[other.size_]), size_(other.size_) {
        std::copy(other.data_, other.data_ + size_, data_);
    }
    
    Widget& operator=(Widget other) {  // Pass by value!
        swap(other);  // Swap with copy
        return *this;
    }  // other destroyed, cleaning up old data
    
    void swap(Widget& other) noexcept {
        std::swap(data_, other.data_);
        std::swap(size_, other.size_);
    }
    
    ~Widget() {
        delete[] data_;
    }
};
```

**How it works:**
```cpp
Widget w1(100);
Widget w2(50);

w1 = w2;
// 1. Copy constructor creates temporary from w2
// 2. Swap w1's data with temporary
// 3. Temporary destroyed (cleaning up w1's old data)
```

**Benefits:**
1. **Strong exception guarantee:** If copy fails, original unchanged
2. **No self-assignment check needed:** Works automatically
3. **Code reuse:** Uses copy constructor

**With move semantics:**
```cpp
class Widget {
public:
    Widget& operator=(Widget other) {  // Universal assignment
        swap(other);
        return *this;
    }
};

Widget w1;
w1 = w2;           // Copy assignment (w2 copied)
w1 = std::move(w2); // Move assignment (w2 moved)
```

---

### 65. **Explain the static initialization order fiasco**

Global/static objects in different translation units have undefined initialization order.

**Problem:**
```cpp
// File1.cpp
Logger& getLogger() {
    static Logger logger;  // When is this initialized?
    return logger;
}

// File2.cpp
extern Logger& getLogger();
Config config(getLogger());  // ⚠️ getLogger() might not be initialized!
```

**Static initialization fiasco:**
```cpp
// a.cpp
extern int y;
int x = y + 1;  // Uses y

// b.cpp
extern int x;
int y = x + 1;  // Uses x

// Order undefined! One will use uninitialized value
```

**Solution 1 - Function-local static (Meyers Singleton):**
```cpp
Logger& getLogger() {
    static Logger logger;  // Initialized on first call
    return logger;
}

// Safe to use from any global constructor
Config config(getLogger());  // OK - logger initialized if needed
```

**Solution 2 - Explicit initialization:**
```cpp
// init.cpp
void initialize() {
    getLogger();  // Force initialization
    getConfig();
}

int main() {
    initialize();  // Call before anything else
    // ...
}
```

**Solution 3 - Avoid cross-TU dependencies:**
```cpp
// Keep globals in same translation unit
// Or use headers with inline definitions
```

**Why Meyers Singleton works:**
```cpp
Logger& getLogger() {
    static Logger logger;
    return logger;
}
// C++ guarantees static locals initialized on first call
// Thread-safe since C++11
```

---

### 66. **What is Argument-Dependent Lookup (ADL) / Koenig Lookup?**

ADL includes namespaces of argument types when looking up function names.

**Basic example:**
```cpp
namespace MyNamespace {
    class Widget {};
    
    void process(Widget w) {
        std::cout << "Processing widget\n";
    }
}

MyNamespace::Widget w;
process(w);  // ✅ Finds MyNamespace::process via ADL!
             // No need for MyNamespace::process(w)
```

**How it works:**
```cpp
namespace N {
    class T {};
    void func(T) {}
}

N::T obj;
func(obj);  // ADL looks in namespace N
```

**Standard library example:**
```cpp
std::vector<int> vec;
std::cout << vec;  // ❌ No operator<< in global namespace

// But this works:
namespace std {
    // operator<< found via ADL
    // because vec is in std namespace
}
```

**Swap example:**
```cpp
namespace MyLib {
    class Widget {
        // ...
    };
    
    void swap(Widget& a, Widget& b) {
        // Optimized swap
    }
}

MyLib::Widget w1, w2;
std::swap(w1, w2);  // Might not call MyLib::swap

// Better:
using std::swap;
swap(w1, w2);  // ADL finds MyLib::swap
```

**When ADL applies:**
```cpp
func(arg);  // ADL looks in arg's namespace
obj.method();  // No ADL (qualified lookup)
::func(arg);  // No ADL (global namespace specified)
```

**Potential issues:**
```cpp
namespace N {
    class T {};
}

void func(int);  // Global function

namespace N {
    void func(double);  // Overload in N
}

N::T obj;
func(obj);  // ❌ Error: N::func(double) found via ADL
            // But doesn't match T argument
```

---

## Copy Constructor & Special Members (Questions 67-72)

### 67. **What is the copy-elision guarantee in C++17 and how does it affect copy constructors?**

C++17 guarantees copy elision for prvalues (temporary materialization).

**Guaranteed elision:**
```cpp
struct NonMovable {
    NonMovable() = default;
    NonMovable(const NonMovable&) = delete;
    NonMovable(NonMovable&&) = delete;
};

NonMovable create() {
    return NonMovable();  // ✅ OK in C++17 - guaranteed elision
}

auto obj = create();  // ✅ No copy or move needed
```

**Before C++17:**
```cpp
// Required accessible copy/move constructor even if not called
struct S {
    S(const S&) = delete;
};

S create() {
    return S();  // ❌ Error in C++14 (even with RVO)
}
```

**Named objects - still need copy/move:**
```cpp
Widget create() {
    Widget w;
    return w;  // NRVO - optional, needs copy/move constructor
}

// Copy constructor must exist:
struct NonCopyable {
    NonCopyable(const NonCopyable&) = delete;
};

NonCopyable create() {
    NonCopyable nc;
    return nc;  // ❌ Error - even though might not be called
}
```

**Passing to function:**
```cpp
void func(Widget w);

func(Widget());  // C++17: Guaranteed elision
func(create());  // C++17: Guaranteed elision if create returns prvalue

Widget w;
func(w);  // Still requires copy/move
```

---

### 68. **Explain the difference between shallow copy and deep copy. When does the default copy constructor fail?**

**Shallow copy:** Copies pointer values (both point to same memory).
**Deep copy:** Duplicates pointed-to data.

**Default copy constructor (shallow):**
```cpp
class BadArray {
    int* data_;
    size_t size_;
public:
    BadArray(size_t size) : data_(new int[size]), size_(size) {}
    ~BadArray() { delete[] data_; }
    
    // Implicit copy constructor does shallow copy:
    // BadArray(const BadArray& other) 
    //     : data_(other.data_), size_(other.size_) {}
};

BadArray arr1(10);
BadArray arr2 = arr1;  // Shallow copy - both point to same data!
// ~arr2 deletes data
// ~arr1 tries to delete again → double delete!
```

**Deep copy solution:**
```cpp
class GoodArray {
    int* data_;
    size_t size_;
public:
    GoodArray(size_t size) : data_(new int[size]), size_(size) {}
    
    ~GoodArray() { delete[] data_; }
    
    // Deep copy
    GoodArray(const GoodArray& other) 
        : data_(new int[other.size_]), size_(other.size_) {
        std::memcpy(data_, other.data_, size_ * sizeof(int));
    }
    
    GoodArray& operator=(const GoodArray& other) {
        if (this != &other) {
            delete[] data_;
            size_ = other.size_;
            data_ = new int[size_];
            std::memcpy(data_, other.data_, size_ * sizeof(int));
        }
        return *this;
    }
};
```

**When default fails:**
1. **Raw pointers to owned data**
2. **File handles:** `FILE*`, socket descriptors
3. **Unique resources:** Mutexes, database connections
4. **Reference-counted resources:** Need manual count management

**Better approach - RAII:**
```cpp
class ModernArray {
    std::vector<int> data_;  // Handles deep copy automatically
public:
    ModernArray(size_t size) : data_(size) {}
    // Compiler-generated copy is correct!
};
```

---

### 69. **What is the copy-on-write (COW) optimization and why is it problematic in multithreaded code?**

COW delays copying until modification. Multiple objects share data with reference count.

**COW implementation:**
```cpp
class COWString {
    struct Data {
        char* str;
        std::atomic<int> refcount;
    };
    Data* data_;
    
public:
    COWString(const COWString& other) {
        data_ = other.data_;
        ++data_->refcount;  // Atomic increment
    }
    
    void modify(size_t pos, char c) {
        if (data_->refcount > 1) {
            // Copy on write
            Data* new_data = new Data{strdup(data_->str), 1};
            --data_->refcount;
            data_ = new_data;
        }
        data_->str[pos] = c;
    }
};
```

**Problems in multithreaded code:**

1. **Atomic operations expensive:**
```cpp
// Every copy needs atomic increment
std::string s1 = "hello";
std::string s2 = s1;  // Atomic increment
std::string s3 = s1;  // Atomic increment
// Even though strings never modified!
```

2. **False sharing:**
```cpp
// Thread 1
std::string s1 = shared_string;  // Touches refcount

// Thread 2
std::string s2 = shared_string;  // Touches same cache line
// Cache ping-pong between cores
```

3. **Overhead defeats purpose:**
```cpp
// COW intended to avoid copies
// But atomic operations + cache coherence
// often slower than just copying!
```

**Why C++11 banned COW for std::string:**

C++11 move semantics made COW unnecessary:
```cpp
std::string s1 = "hello";
std::string s2 = std::move(s1);  // Cheap move, no COW needed
```

**SSO (Small String Optimization) replaced COW:**
```cpp
// Modern std::string stores short strings inline
std::string s = "short";  // No heap allocation!
```

---

### 70. **How does the copy constructor interact with inheritance?**

Derived class copy constructor must explicitly call base copy constructor.

**Problem - forgot base copy:**
```cpp
class Base {
    int x_;
public:
    Base(int x) : x_(x) {}
    Base(const Base& other) : x_(other.x_) {}
};

class Derived : public Base {
    int y_;
public:
    Derived(int x, int y) : Base(x), y_(y) {}
    
    // ❌ Wrong - doesn't copy base portion!
    Derived(const Derived& other) : y_(other.y_) {}
    // Base default constructor called, x_ not copied
};

Derived d1(10, 20);
Derived d2 = d1;  // d2.x_ is uninitialized or default!
```

**Correct - explicitly copy base:**
```cpp
class Derived : public Base {
    int y_;
public:
    Derived(const Derived& other) 
        : Base(other),  // ✅ Call base copy constructor
          y_(other.y_) {}
};
```

**With multiple inheritance:**
```cpp
class Derived : public Base1, public Base2 {
public:
    Derived(const Derived& other)
        : Base1(other),  // Copy Base1
          Base2(other),  // Copy Base2
          member_(other.member_) {}
};
```

**Virtual inheritance:**
```cpp
class Derived : public virtual Base {
public:
    Derived(const Derived& other)
        : Base(other),  // Must explicitly call
          member_(other.member_) {}
};
```

---

### 71. **Explain the copy elision rules for function parameters and return values**

**Parameters - never elided:**
```cpp
void func(Widget w);  // Always requires copy or move

Widget source;
func(source);  // Copy or move, never elided
```

**Return values - NRVO (optional):**
```cpp
Widget create() {
    Widget w;
    // ...
    return w;  // NRVO may or may not happen
}
// Copy/move constructor must exist
```

**C++17 prvalue guarantee:**
```cpp
Widget create() {
    return Widget();  // Guaranteed elision
}

auto w = create();  // No copy or move
```

**Don't use std::move on return:**
```cpp
Widget wrong() {
    Widget w;
    return std::move(w);  // ❌ Prevents NRVO!
}

Widget correct() {
    Widget w;
    return w;  // ✅ Enables NRVO
}
```

**Multiple return paths prevent NRVO:**
```cpp
Widget conditional(bool flag) {
    Widget w1, w2;
    return flag ? w1 : w2;  // Can't do NRVO - different objects
                            // Will use move
}
```

**RVO priority:**
```cpp
Widget create() {
    return Widget();  // 1. RVO (best)
}

Widget create() {
    Widget w;
    return w;  // 1. Try NRVO  2. Try move  3. Copy
}
```

---

### 72. **What happens when copy constructor throws an exception?**

Partially constructed object must be destroyed. Destructors called for fully constructed members.

**Problem:**
```cpp
class Widget {
    Resource* r1_;
    Resource* r2_;
public:
    Widget(const Widget& other) {
        r1_ = new Resource(*other.r1_);  // Might throw
        r2_ = new Resource(*other.r2_);  // If this throws, r1_ leaks!
    }
};
```

**Solution - use member initializer list:**
```cpp
class Widget {
    std::unique_ptr<Resource> r1_;
    std::unique_ptr<Resource> r2_;
public:
    Widget(const Widget& other)
        : r1_(std::make_unique<Resource>(*other.r1_)),
          r2_(std::make_unique<Resource>(*other.r2_))
    {
        // If r2_ construction throws,
        // r1_ automatically destroyed
    }
};
```

**Impact on containers:**
```cpp
std::vector<Widget> vec;

Widget w;
vec.push_back(w);  // Copy constructor called

// If copy constructor can throw and no noexcept move:
// vector must use copy for strong exception guarantee
```

**Exception safety levels:**
```cpp
class Widget {
public:
    // Basic guarantee - might leak but no corruption
    Widget(const Widget& other);
    
    // Strong guarantee - all or nothing
    Widget(const Widget& other) {
        // Use copy-and-swap or RAII
    }
    
    // No-throw guarantee
    Widget(const Widget& other) noexcept;
};
```

---

## STL Containers Deep Dive (Questions 73-80)

### 73. **Explain iterator invalidation rules for `std::vector`, `std::deque`, and `std::list`**

**`std::vector`:**
```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};
auto it = vec.begin();

vec.push_back(6);  // May reallocate
// ⚠️ it now invalid if reallocation happened

vec.insert(vec.begin() + 2, 10);
// ⚠️ All iterators from insert point onward invalid

vec.erase(vec.begin() + 2);
// ⚠️ Iterators at and after erase point invalid
```

**Rules:**
- Any operation causing reallocation → ALL iterators invalid
- Insert/erase → Iterators at/after position invalid

**`std::deque`:**
```cpp
std::deque<int> deq = {1, 2, 3, 4, 5};
auto it = deq.begin();

deq.push_back(6);   // ✅ it still valid
deq.push_front(0);  // ✅ it still valid (but position changed)

deq.insert(deq.begin() + 2, 10);
// ⚠️ ALL iterators invalid (middle insertion)
```

**Rules:**
- Insert at ends → Iterators valid, references valid
- Insert in middle → ALL iterators invalid, references valid
- Erase at ends → Only at erased element
- Erase in middle → ALL iterators invalid

**`std::list`:**
```cpp
std::list<int> lst = {1, 2, 3, 4, 5};
auto it = lst.begin();
++it;  // Points to 2

lst.insert(lst.begin(), 0);  // ✅ it still valid
lst.erase(lst.begin());      // ✅ it still valid

auto it2 = std::next(it);
lst.erase(it2);  // Only it2 invalid, it still valid
```

**Rules:**
- Only erased elements' iterators invalid
- All other iterators remain valid

**Summary:**
```cpp
// vector: Insert/erase → invalidates at/after position
// deque: Insert/erase middle → invalidates all iterators
// list: Only erased elements invalid
```

---

### 74. **What is Small String Optimization (SSO) and how does it affect `std::string` performance?**

SSO stores short strings directly in string object, avoiding heap allocation.

**Implementation concept:**
```cpp
class string {
    union {
        char* heap_ptr;
        char buffer[16];  // Typical size: 15 chars + null
    };
    size_t size_;
    size_t capacity_;
    
public:
    string(const char* str) {
        if (strlen(str) < 16) {
            strcpy(buffer, str);  // Use buffer (SSO)
        } else {
            heap_ptr = new char[strlen(str) + 1];  // Heap
            strcpy(heap_ptr, str);
        }
    }
};
```

**Performance impact:**

**Short strings - no allocation:**
```cpp
std::string s1 = "hello";        // No heap allocation!
std::string s2 = "short";        // No heap allocation!
std::string s3 = s1;             // Copies buffer, still no heap
```

**Long strings - heap allocation:**
```cpp
std::string s = "This is a much longer string that exceeds SSO size";
// Heap allocation required
```

**Move is not always free:**
```cpp
std::string short_str = "hi";
std::string s2 = std::move(short_str);  // Still copies buffer!
                                         // Can't steal pointer (it's not a pointer)

std::string long_str = "very long string exceeding SSO buffer size";
std::string s3 = std::move(long_str);   // Cheap - steals pointer
```

**Typical SSO sizes:**
```cpp
// GCC libstdc++: 15 bytes
// MSVC: 15 bytes  
// Clang libc++: 22 bytes

sizeof(std::string);  // Usually 24 or 32 bytes (holds buffer)
```

**Implications:**
```cpp
void func(std::string s);  // Pass by value

func("hello");       // Fast - SSO, no heap
func(very_long_str); // Heap allocation for copy

// Prefer:
void func(std::string_view s);  // No copy at all
```

---

### 75. **Explain the difference between `std::map` and `std::unordered_map` in terms of complexity and when to use each**

**`std::map`:** Ordered, tree-based (red-black tree).
```cpp
std::map<int, std::string> map;
map[3] = "three";
map[1] = "one";
map[2] = "two";

for (auto& [k, v] : map) {
    std::cout << k;  // Prints: 1, 2, 3 (sorted)
}

// Complexity:
// Insert: O(log n)
// Find: O(log n)
// Delete: O(log n)
// Iteration: O(n) in sorted order
```

**`std::unordered_map`:** Unordered, hash table.
```cpp
std::unordered_map<int, std::string> map;
map[3] = "three";
map[1] = "one";
map[2] = "two";

for (auto& [k, v] : map) {
    std::cout << k;  // Prints: unspecified order
}

// Complexity:
// Insert: O(1) average, O(n) worst
// Find: O(1) average, O(n) worst
// Delete: O(1) average, O(n) worst
// Iteration: O(n) unordered
```

**When to use `std::map`:**
```cpp
// 1. Need sorted order
for (auto& [k, v] : map) {
    // Process in sorted order
}

// 2. Range queries
auto it1 = map.lower_bound(10);
auto it2 = map.upper_bound(20);
// All elements between 10 and 20

// 3. Predictable performance (no hash collisions)
// Always O(log n), never O(n)

// 4. No hash function available
struct ComplexKey {
    // Has operator< but no hash
};
std::map<ComplexKey, Value> map;
```

**When to use `std::unordered_map`:**
```cpp
// 1. Faster lookups (average case)
if (map.find(key) != map.end()) {  // O(1) average
}

// 2. Don't need sorted order
// Just need fast key->value lookup

// 3. Large datasets
// O(1) vs O(log n) becomes significant

// 4. Simple keys (int, string)
std::unordered_map<int, Data> cache;  // Fast cache
```

**Memory usage:**
```cpp
// map: ~32 bytes per entry (tree nodes + pointers)
// unordered_map: ~16 bytes per entry + bucket overhead
```

---

### 76. **What are the guarantees of `std::vector::push_back` vs `emplace_back`?**

Both provide strong exception guarantee when move constructor is `noexcept`.

**`push_back` - constructs then moves:**
```cpp
std::vector<Widget> vec;
Widget w(args);
vec.push_back(w);      // 1. Copy w  2. Move into vector
vec.push_back(Widget(args));  // 1. Construct temp  2. Move into vector
```

**`emplace_back` - constructs in-place:**
```cpp
std::vector<Widget> vec;
vec.emplace_back(args);  // Construct directly in vector
```

**Performance:**
```cpp
// push_back with temporary
vec.push_back(Widget(1, 2, 3));
// 1. Construct Widget(1,2,3)
// 2. Move into vector
// 3. Destroy temporary

// emplace_back
vec.emplace_back(1, 2, 3);
// 1. Construct directly in vector
// More efficient!
```

**When emplace_back is actually slower:**
```cpp
Widget w(args);
vec.push_back(w);      // One copy
vec.emplace_back(w);   // Still one copy! No advantage

// emplace_back only wins with construction arguments
vec.emplace_back(1, 2, 3);  // Better
```

**Surprising behavior:**
```cpp
std::vector<std::string> vec;
vec.emplace_back(10, 'a');  // Constructs string("aaaaaaaaaa")
vec.push_back(10, 'a');     // ❌ Error - no implicit conversion
```

**Exception safety:**
```cpp
// If Widget constructor throws:
vec.emplace_back(args);  // Vector unchanged (strong guarantee)
                         // If noexcept move available
```

**Recommendation:**
```cpp
// Use push_back for existing objects
Widget w;
vec.push_back(std::move(w));

// Use emplace_back for construction
vec.emplace_back(arg1, arg2);

// Don't overthink it - performance difference usually negligible
```

---

### 77. **Explain why `std::vector<bool>` is considered broken**

`std::vector<bool>` is specialized to pack bits, violating container requirements.

**Problems:**

**1. `operator[]` returns proxy, not `bool&`:**
```cpp
std::vector<bool> vec = {true, false, true};

bool& ref = vec[0];  // ❌ Error! Returns proxy object

auto& ref = vec[0];  // Actually a proxy, not reference
ref = false;         // Modifies through proxy
```

**2. Can't take address of element:**
```cpp
std::vector<bool> vec = {true, false};

bool* ptr = &vec[0];  // ❌ Error! Can't take address of bitfield
```

**3. Not thread-safe for different elements:**
```cpp
std::vector<bool> vec(100);

// Thread 1
vec[0] = true;

// Thread 2  
vec[1] = false;  // ⚠️ Data race! Same byte
```

**4. Breaks generic code:**
```cpp
template<typename T>
void process(std::vector<T>& vec) {
    T& ref = vec[0];  // ❌ Fails for T=bool
}
```

**Implementation (conceptual):**
```cpp
template<>
class vector<bool> {
    struct reference {
        unsigned char* byte;
        unsigned char mask;
        
        operator bool() const { return *byte & mask; }
        reference& operator=(bool b) {
            if (b) *byte |= mask;
            else *byte &= ~mask;
            return *this;
        }
    };
    
public:
    reference operator[](size_t pos);
};
```

**Alternatives:**
```cpp
// 1. std::deque<bool>
std::deque<bool> d = {true, false};
bool& ref = d[0];  // ✅ Real reference

// 2. std::vector<char>
std::vector<char> vec = {1, 0, 1};
char& ref = vec[0];  // ✅ Works

// 3. std::bitset (if size known)
std::bitset<100> bits;
bits[0] = true;

// 4. boost::dynamic_bitset (if size dynamic)
```

---

### 78. **What is the difference between `reserve()` and `resize()` for `std::vector`?**

**`reserve(n)`:** Allocates capacity, doesn't change size.
```cpp
std::vector<int> vec;
std::cout << vec.size();      // 0
std::cout << vec.capacity();  // 0

vec.reserve(100);
std::cout << vec.size();      // Still 0!
std::cout << vec.capacity();  // 100

vec.push_back(1);  // No reallocation
```

**`resize(n)`:** Changes size, constructs/destructs elements.
```cpp
std::vector<int> vec;
vec.resize(100);
std::cout << vec.size();      // 100
std::cout << vec[0];          // 0 (default-constructed)

vec.resize(50);  // Destroys last 50 elements
std::cout << vec.size();      // 50
```

**Use cases:**

**reserve - avoid reallocations:**
```cpp
std::vector<int> vec;
vec.reserve(1000);  // Pre-allocate

for (int i = 0; i < 1000; ++i) {
    vec.push_back(i);  // No reallocations!
}
```

**resize - create elements:**
```cpp
std::vector<int> vec(100);  // 100 default-constructed elements
vec[50] = 42;  // Can access immediately
```

**With value:**
```cpp
vec.resize(100, 42);  // All elements initialized to 42
```

**Performance:**
```cpp
// Bad - multiple reallocations
std::vector<int> vec;
for (int i = 0; i < 10000; ++i) {
    vec.push_back(i);  // Reallocates ~14 times
}

// Good - one allocation
std::vector<int> vec;
vec.reserve(10000);
for (int i = 0; i < 10000; ++i) {
    vec.push_back(i);  // No reallocations
}
```

---

### 79. **How does `std::unordered_map` handle collisions and what is load factor?**

**Collision handling - separate chaining:**
```cpp
// Simplified unordered_map
template<typename Key, typename Value>
class unordered_map {
    struct Node {
        Key key;
        Value value;
        Node* next;  // Linked list for collisions
    };
    
    std::vector<Node*> buckets_;
    size_t size_;
    
    size_t hash(const Key& key) const {
        return std::hash<Key>{}(key) % buckets_.size();
    }
    
    void insert(const Key& key, const Value& value) {
        size_t bucket = hash(key);
        // Add to linked list at buckets_[bucket]
    }
};
```

**Load factor:**
```cpp
float load_factor = size() / bucket_count();

std::unordered_map<int, int> map;
map.insert({1, 1});
map.insert({2, 2});

std::cout << map.load_factor();        // e.g., 0.2
std::cout << map.max_load_factor();    // 1.0 (default)
```

**Rehashing:**
```cpp
std::unordered_map<int, int> map;

for (int i = 0; i < 100; ++i) {
    map[i] = i;
    if (map.load_factor() > map.max_load_factor()) {
        // Automatic rehash triggered!
        // Doubles bucket count, rehashes all elements
    }
}
```

**Manual control:**
```cpp
std::unordered_map<int, int> map;

// Pre-allocate buckets
map.reserve(1000);  // Avoids rehashing up to 1000 elements

// Change max load factor
map.max_load_factor(0.75);  // Rehash more aggressively

// Force rehash
map.rehash(2000);  // Set bucket count
```

**Performance impact:**
```cpp
// With good hash function:
// Average O(1): Most lookups hit right bucket
// Worst O(n): All elements hash to same bucket

// High load factor:
// More collisions → longer chains → slower lookups

// Low load factor:
// Fewer collisions but more memory usage
```

---

### 80. **Explain the performance characteristics of inserting into middle of different containers**

**`std::vector`:** O(n) - must shift elements.
```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};
vec.insert(vec.begin() + 2, 99);  // Shifts 3,4,5 right
// Time: O(n) where n = elements after insert point
```

**`std::deque`:** O(n) - better than vector for front/back.
```cpp
std::deque<int> deq = {1, 2, 3, 4, 5};
deq.insert(deq.begin() + 2, 99);  // Shifts smaller half
// Time: O(min(pos, size-pos))
```

**`std::list`:** O(1) - once iterator found.
```cpp
std::list<int> lst = {1, 2, 3, 4, 5};
auto it = std::next(lst.begin(), 2);
lst.insert(it, 99);  // Just relink pointers
// Time: O(1) for insert, O(n) to find position
```

**`std::set/map`:** O(log n) - tree rebalancing.
```cpp
std::set<int> s = {1, 2, 3, 4, 5};
s.insert(99);  // Tree rebalance
// Time: O(log n)
```

**Benchmark example:**
```cpp
// Inserting 10,000 elements in middle:

std::vector:  ~50ms   (50 million shifts)
std::deque:   ~30ms   (optimized shifts)
std::list:    ~100ms  (pointer chasing, cache misses)
std::set:     ~5ms    (logarithmic, cache-friendly)

// Inserting at end:
std::vector:  ~0.1ms  (amortized O(1))
std::list:    ~10ms   (allocation overhead)
```

**Choosing container:**
```cpp
// Frequent middle insertion:
// - Few elements: std::vector (cache-friendly)
// - Many elements with known insert point: std::list
// - Need sorting: std::set

// Mostly end insertion:
// - std::vector (best choice)

// Front and back insertion:
// - std::deque
```

---

## Compile-Time Programming (Questions 81-87)

### 81. **What's the difference between `constexpr`, `consteval`, and `constinit` (C++20)?**

**`constexpr`:** Can run at compile-time OR runtime.
```cpp
constexpr int square(int x) {
    return x * x;
}

constexpr int a = square(5);  // Compile-time: 25
int b = 5;
int c = square(b);  // Runtime: calls function
```

**`consteval`:** MUST run at compile-time (immediate functions).
```cpp
consteval int square(int x) {
    return x * x;
}

constexpr int a = square(5);  // ✅ Compile-time
int b = 5;
int c = square(b);  // ❌ Error: b not constant
```

**`constinit`:** Variable initialized at compile-time but NOT const.
```cpp
constinit int global = 42;  // Initialized at compile-time
// global = 50;  // ✅ Can be modified at runtime!

constexpr int another = 42;  // Also compile-time
// another = 50;  // ❌ Error: constexpr is const
```

**Comparison:**
```cpp
// constexpr: compile-time if possible, const
constexpr int x = 10;

// consteval: always compile-time
consteval int get_ten() { return 10; }

// constinit: compile-time init, mutable
constinit int y = 10;
y = 20;  // OK
```

---

### 82. **Can you have a `constexpr` function that doesn't run at compile-time?**

Yes! `constexpr` means CAN be evaluated at compile-time if arguments are constant.

```cpp
constexpr int add(int a, int b) {
    return a + b;
}

// Compile-time
constexpr int x = add(10, 20);  // Computed at compile-time

// Runtime
int a = 10, b = 20;
int y = add(a, b);  // Computed at runtime

// Verify:
static_assert(x == 30);  // OK - x is compile-time constant
// static_assert(y == 30);  // ❌ Error - y not compile-time
```

**When constexpr runs at compile-time:**
1. Used in constant expression context
2. All arguments are constant expressions
3. Function body satisfies constexpr requirements

**When constexpr runs at runtime:**
1. Arguments not constant expressions
2. Result not used in constant context

**Example:**
```cpp
constexpr int factorial(int n) {
    return (n <= 1) ? 1 : n * factorial(n - 1);
}

// Compile-time
constexpr int a = factorial(5);  // Computed at compile-time
int arr[factorial(4)];  // Array size must be compile-time

// Runtime
int n;
std::cin >> n;
int b = factorial(n);  // Computed at runtime
```

---

### 83. **What are the restrictions on `constexpr` functions in C++11 vs C++14 vs C++20?**

**C++11:** Very restrictive.
```cpp
// OK
constexpr int square(int x) {
    return x * x;  // Single return statement only
}

// ❌ Not allowed:
// - Loops
// - Multiple statements
// - Local variables
// - Mutation
```

**C++14:** Much more permissive.
```cpp
// Now OK:
constexpr int factorial(int n) {
    int result = 1;  // ✅ Local variables
    for (int i = 2; i <= n; ++i) {  // ✅ Loops
        result *= i;  // ✅ Mutation
    }
    return result;  // ✅ Multiple statements
}
```

**C++20:** Almost unrestricted.
```cpp
// Now OK:
constexpr int parse(const char* str) {
    std::vector<int> vec;  // ✅ std::vector
    std::string s = str;   // ✅ std::string
    
    try {  // ✅ try/catch
        throw 1;
    } catch (...) {
    }
    
    return vec.size();
}

// ✅ Virtual functions
// ✅ new/delete (must clean up)
// ✅ union access
```

---

### 84. **Why would you use `constinit` instead of `constexpr` for a global variable?**

`constinit` ensures compile-time initialization without forcing constness.

**Problem with constexpr:**
```cpp
constexpr int max_connections = 100;  // const - can't modify
// max_connections = 200;  // ❌ Error
```

**Solution with constinit:**
```cpp
constinit int max_connections = 100;  // Not const - can modify
max_connections = 200;  // ✅ OK at runtime

// But prevents static initialization order fiasco:
constinit int x = compute_at_compile_time();  // ✅ Safe
```

**Use cases:**

**1. Mutable configuration:**
```cpp
constinit int thread_pool_size = 4;  // Compile-time init

void configure(int size) {
    thread_pool_size = size;  // ✅ Can change at runtime
}
```

**2. Avoiding initialization order issues:**
```cpp
// a.cpp
constinit Logger* logger = init_logger();  // ✅ Compile-time

// b.cpp
extern Logger* logger;
Config config(logger);  // Safe - logger already initialized
```

**3. Thread-local storage:**
```cpp
thread_local constinit int thread_id = get_thread_id();
// Compile-time init per thread
```

---

### 85. **Explain `if constexpr` and how it enables compile-time branching in templates**

`if constexpr` discards non-taken branch at compile-time.

**Example:**
```cpp
template<typename T>
auto process(T value) {
    if constexpr (std::is_integral_v<T>) {
        return value * 2;  // Only compiled for integral
    } else if constexpr (std::is_floating_point_v<T>) {
        return value * 1.5;  // Only compiled for floating point
    } else {
        return value;  // Only compiled for others
    }
}

process(10);    // Returns 20 (int * 2)
process(10.0);  // Returns 15.0 (double * 1.5)
process("hi");  // Returns "hi"
```

**Enables different return types:**
```cpp
template<typename T>
auto get_value(T container) {
    if constexpr (requires { container.size(); }) {
        return container.size();  // Returns size_t
    } else {
        return 0;  // Returns int
    }
}
```

**Recursion termination:**
```cpp
template<typename T, typename... Args>
void print(T first, Args... args) {
    std::cout << first << ' ';
    
    if constexpr (sizeof...(args) > 0) {
        print(args...);  // Only compiled if more args
    }
}

print(1, 2, 3, "hello");  // Works!
```

---

### 86. **Can a `constexpr` constructor contain `throw` statements?**

**C++11:** No.
**C++20:** Yes, but throwing at compile-time is compilation error.

```cpp
struct Widget {
    constexpr Widget(int x) {
        if (x < 0)
            throw std::invalid_argument("negative");  // C++20: OK
    }
};

// Compile-time - must not throw
constexpr Widget w1(10);  // ✅ OK

// Compile-time - throws → compilation error
// constexpr Widget w2(-1);  // ❌ Compilation error

// Runtime - can throw
Widget w3(-1);  // ✅ Throws at runtime
```

---

### 87. **What happens when you use `constinit` with a non-static variable?**

Compile error - `constinit` only for static storage duration.

```cpp
void func() {
    constinit int x = 10;  // ❌ Error: local variable
}

constinit int global = 10;  // ✅ OK: global
static constinit int s = 10;  // ✅ OK: static
thread_local constinit int t = 10;  // ✅ OK: thread_local

class Widget {
    constinit int member = 10;  // ❌ Error: non-static member
    static constinit int s_member = 10;  // ✅ OK: static member
};
```

---

## Modern C++ Attributes (Questions 88-90)

### 88. **What is `[[no_unique_address]]` (C++20) and when would you use it?**

Allows empty members to take zero space (extends EBO to members).

**Problem:**
```cpp
struct Empty {};

struct S {
    Empty e;
    int x;
};

sizeof(S);  // 8 bytes (4 for int + padding)
```

**Solution:**
```cpp
struct S {
    [[no_unique_address]] Empty e;
    int x;
};

sizeof(S);  // 4 bytes! Empty e takes no space
```

**Use case - policy-based design:**
```cpp
template<typename Allocator>
struct Vector {
    [[no_unique_address]] Allocator alloc;
    int* data;
    size_t size;
};

// If Allocator is empty, no size overhead!
```

---

### 89. **Explain `[[nodiscard]]` with a string message (C++20)**

Provides custom diagnostic when return value ignored.

```cpp
[[nodiscard("Always check for errors")]]
bool save_file(const std::string& filename);

save_file("data.txt");  // ⚠️ Warning: Always check for errors
```

---

### 90. **What is `[[carries_dependency]]` and when would you use it?**

Related to `memory_order_consume`. Rarely used in practice.

```cpp
[[carries_dependency]]
int* get_ptr(std::atomic<int*>& ptr) {
    return ptr.load(std::memory_order_consume);
}
```

Most code uses `memory_order_acquire` instead.

---

