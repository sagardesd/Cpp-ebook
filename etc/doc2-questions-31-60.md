# C++ Interview Questions - Document 2 (Questions 31-60)

## Memory Model & Atomics (Questions 31-36 continued)

### 31. **Explain acquire-release semantics**

Acquire-release is a synchronization mechanism where release store synchronizes-with acquire load.

**Release (store):** All writes before release become visible to acquirer
**Acquire (load):** All writes before paired release are visible after acquire

```cpp
std::atomic<bool> ready{false};
std::vector<int> data;

// Thread 1 (Producer)
data.push_back(1);
data.push_back(2);
data.push_back(3);
ready.store(true, std::memory_order_release);  // Release barrier

// Thread 2 (Consumer)
while (!ready.load(std::memory_order_acquire));  // Acquire barrier
// All data writes are visible here
for (int x : data) {
    std::cout << x;  // Safe - sees all 3 elements
}
```

**Visualization:**
```
Thread 1:                Thread 2:
  write data
  write more data
  ↓ (release)
  store flag -------→ load flag (acquire)
                            ↓
                        read data (sees all writes)
```

**Multiple readers:**
```cpp
std::atomic<Data*> ptr{nullptr};

// Writer
Data* d = new Data{...};
ptr.store(d, std::memory_order_release);

// Reader 1
if (auto p = ptr.load(std::memory_order_acquire)) {
    use(p);  // Safe
}

// Reader 2  
if (auto p = ptr.load(std::memory_order_acquire)) {
    use(p);  // Also safe
}
```

---

### 32. **What's the difference between `is_lock_free()` and `is_always_lock_free`?**

**`is_lock_free()` - runtime query:**
```cpp
std::atomic<int> ai;
if (ai.is_lock_free()) {
    // This instance uses lock-free operations
}
```

**`is_always_lock_free` - compile-time constant:**
```cpp
if constexpr (std::atomic<int>::is_always_lock_free) {
    // Type is ALWAYS lock-free on this platform
}
```

**Example showing difference:**
```cpp
struct alignas(64) LargeStruct {
    int data[16];
};

std::atomic<LargeStruct> als;

// May vary at runtime based on alignment
als.is_lock_free();  // Depends on actual alignment

// Compile-time constant - false (too large)
std::atomic<LargeStruct>::is_always_lock_free;  // false
```

**Use in templates:**
```cpp
template<typename T>
class LockFreeQueue {
    static_assert(std::atomic<T*>::is_always_lock_free,
                  "Pointers must be lock-free");
    std::atomic<T*> head_;
};
```

---

### 33. **When would you use `memory_order_relaxed`?**

Use `memory_order_relaxed` when only atomicity matters, not ordering.

**Typical use - counters:**
```cpp
class StatsCollector {
    std::atomic<uint64_t> request_count_{0};
    std::atomic<uint64_t> error_count_{0};
    
public:
    void onRequest() {
        request_count_.fetch_add(1, std::memory_order_relaxed);
    }
    
    void onError() {
        error_count_.fetch_add(1, std::memory_order_relaxed);
    }
    
    uint64_t getRequests() const {
        return request_count_.load(std::memory_order_relaxed);
    }
};
```

**Why relaxed works:** Don't care about order of increments, only that each is atomic.

**Won't work for synchronization:**
```cpp
std::atomic<bool> ready{false};
int data;

// Thread 1
data = 42;
ready.store(true, std::memory_order_relaxed);  // ❌ No synchronization!

// Thread 2
if (ready.load(std::memory_order_relaxed)) {
    std::cout << data;  // ⚠️ May see garbage!
}
```

**Correct with other synchronization:**
```cpp
std::atomic<bool> shutdown{false};
std::mutex mtx;

// Thread 1
{
    std::lock_guard lock(mtx);
    // work...
}
shutdown.store(true, std::memory_order_relaxed);  // OK - mutex synchronizes

// Thread 2
while (!shutdown.load(std::memory_order_relaxed)) {
    std::lock_guard lock(mtx);
    // work...
}
```

---

### 34. **Explain the ABA problem in lock-free programming**

ABA problem: Value changes A→B→A, CAS doesn't detect the intermediate change.

**Problem scenario:**
```cpp
template<typename T>
class LockFreeStack {
    struct Node { T data; Node* next; };
    std::atomic<Node*> head_;
    
public:
    void push(T value) {
        Node* new_node = new Node{value, head_.load()};
        while (!head_.compare_exchange_weak(new_node->next, new_node));
    }
    
    bool pop(T& result) {
        Node* old_head = head_.load();
        while (old_head && 
               !head_.compare_exchange_weak(old_head, old_head->next));
        
        if (old_head) {
            result = old_head->data;
            delete old_head;  // ⚠️ ABA problem!
            return true;
        }
        return false;
    }
};
```

**ABA scenario:**
```
Initial: head → A → B → C

Thread 1: pop() reads head (A), preempted

Thread 2: pop() removes A
Thread 2: pop() removes B  
Thread 2: push() reuses A's memory address
          head → A' → C

Thread 1: resumes, CAS succeeds (head still equals A!)
Thread 1: sets head to B (which was deleted!)
```

**Solution 1 - Tagged pointers:**
```cpp
struct TaggedPointer {
    Node* ptr;
    uint64_t tag;
};

std::atomic<TaggedPointer> head_;

void pop() {
    TaggedPointer old_head = head_.load();
    TaggedPointer new_head;
    do {
        if (!old_head.ptr) return;
        new_head.ptr = old_head.ptr->next;
        new_head.tag = old_head.tag + 1;  // Increment tag
    } while (!head_.compare_exchange_weak(old_head, new_head));
}
```

**Solution 2 - Hazard pointers:**
```cpp
thread_local std::atomic<Node*> hazard_ptr;

Node* protect(std::atomic<Node*>& ptr) {
    Node* p;
    do {
        p = ptr.load();
        hazard_ptr.store(p);
    } while (p != ptr.load());
    return p;
}
```

---

### 35. **What are the differences between `compare_exchange_weak` and `compare_exchange_strong`?**

**`compare_exchange_weak`:**
- May spuriously fail even when values match
- Faster on some architectures (ARM, PowerPC)
- Use in loops

```cpp
int expected = 5;
while (!atomic.compare_exchange_weak(expected, 10)) {
    // Handles spurious failures
}
```

**`compare_exchange_strong`:**
- Never spuriously fails
- Only fails on actual mismatch
- Use for single attempts

```cpp
int expected = 5;
if (!atomic.compare_exchange_strong(expected, 10)) {
    // Definitely a real mismatch
    handle_failure(expected);
}
```

**Performance comparison:**
```cpp
// On x86: No difference (both use CMPXCHG)
// On ARM: weak is faster (avoids LL/SC retry loop)

// Weak in loop (better on ARM)
while (!atomic.compare_exchange_weak(old, new)) {
    // Retry
}

// Strong in loop (same on x86, slower on ARM)
while (!atomic.compare_exchange_strong(old, new)) {
    // Retry
}
```

**When to use each:**
```cpp
// Use weak in loops
int old_value = counter.load();
while (!counter.compare_exchange_weak(old_value, old_value + 1));

// Use strong for one-time state transitions
State expected = State::IDLE;
if (state.compare_exchange_strong(expected, State::RUNNING)) {
    start_processing();
} else {
    log_error("Already running");
}
```

---

### 36. **Explain memory fences (`std::atomic_thread_fence`)**

Memory fences establish synchronization without atomic variables.

**Acquire fence:**
```cpp
std::atomic_thread_fence(std::memory_order_acquire);
// All loads/stores after fence can't move before it
```

**Release fence:**
```cpp
std::atomic_thread_fence(std::memory_order_release);
// All loads/stores before fence can't move after it
```

**Example with fences:**
```cpp
int data;
std::atomic<bool> ready{false};

// Thread 1
data = 42;
std::atomic_thread_fence(std::memory_order_release);
ready.store(true, std::memory_order_relaxed);  // Can use relaxed!

// Thread 2
while (!ready.load(std::memory_order_relaxed));
std::atomic_thread_fence(std::memory_order_acquire);
assert(data == 42);  // Guaranteed
```

**When fences are useful - multiple variables:**
```cpp
int x, y, z;
std::atomic<bool> ready{false};

// Thread 1
x = 1;
y = 2;
z = 3;
std::atomic_thread_fence(std::memory_order_release);
ready.store(true, std::memory_order_relaxed);

// Thread 2
while (!ready.load(std::memory_order_relaxed));
std::atomic_thread_fence(std::memory_order_acquire);
// All x, y, z visible
```

**Avoiding expensive atomics:**
```cpp
// Without fence - each needs atomic
atomic_x.store(1, std::memory_order_release);
atomic_y.store(2, std::memory_order_release);
atomic_z.store(3, std::memory_order_release);

// With fence - only flag needs atomic
x = 1;
y = 2;
z = 3;
std::atomic_thread_fence(std::memory_order_release);
flag.store(true, std::memory_order_relaxed);
```

---

## Modern C++ Features (Questions 37-44)

### 37. **Explain structured bindings (C++17) and their limitations**

Structured bindings decompose objects into components with auto type deduction.

**Basic usage:**
```cpp
std::pair<int, std::string> getPair() {
    return {42, "hello"};
}

auto [num, str] = getPair();
// num = 42, str = "hello"
```

**With arrays:**
```cpp
int arr[] = {1, 2, 3};
auto [a, b, c] = arr;
```

**With structs:**
```cpp
struct Point { int x, y; };
Point p{10, 20};
auto [x, y] = p;
```

**With maps:**
```cpp
std::map<std::string, int> map{{"a", 1}, {"b", 2}};

for (const auto& [key, value] : map) {
    std::cout << key << ": " << value << "\n";
}
```

**References:**
```cpp
Point p{10, 20};

auto& [x, y] = p;       // References
x = 30;                 // Modifies p.x

const auto& [cx, cy] = p;  // Const references
```

**Limitations:**

1. **Can't skip elements:**
```cpp
auto [x, _] = getPair();  // ❌ Error
auto [x, ignored] = getPair();  // ✅ Workaround
```

2. **Can't specify types:**
```cpp
auto [int x, std::string s] = getPair();  // ❌ Error
```

3. **Must match element count:**
```cpp
auto [a, b, c] = getPair();  // ❌ Error: only 2 elements
```

---

### 38. **What is `std::optional` and when should you use it instead of pointers?**

`std::optional<T>` represents a value that may or may not exist.

**Basic usage:**
```cpp
std::optional<int> divide(int a, int b) {
    if (b == 0)
        return std::nullopt;
    return a / b;
}

auto result = divide(10, 2);
if (result) {
    std::cout << *result;  // 5
}
```

**Accessing values:**
```cpp
std::optional<int> opt = 42;

// Check and dereference
if (opt) {
    std::cout << *opt;
}

// value_or with default
std::cout << opt.value_or(0);  // 42

// value() throws on empty
try {
    std::cout << opt.value();
} catch (std::bad_optional_access&) {
}

// has_value()
if (opt.has_value()) {
    std::cout << opt.value();
}
```

**Comparison with pointers:**

**Pointer (old way):**
```cpp
int* findValue(const std::vector<int>& vec, int target) {
    for (auto& val : vec) {
        if (val == target)
            return &val;  // Dangerous!
    }
    return nullptr;
}
```

**Optional (better):**
```cpp
std::optional<int> findValue(const std::vector<int>& vec, int target) {
    for (auto val : vec) {
        if (val == target)
            return val;
    }
    return std::nullopt;
}
```

**Benefits:**
1. No heap allocation
2. Value semantics
3. Type safety
4. Explicit optionality

**Use cases:**
```cpp
// Configuration
std::optional<std::string> getConfigValue(const std::string& key);

// Parsing
std::optional<int> parseInt(const std::string& str);

// Optional parameter
void process(int req, std::optional<int> opt = std::nullopt);
```

---

### 39. **Explain `if constexpr` (C++17) and how it differs from regular `if`**

`if constexpr` evaluates at compile-time and only compiles the taken branch.

**Regular if (runtime):**
```cpp
template<typename T>
void process(T value) {
    if (std::is_integral_v<T>) {
        return value * 2;  // ❌ Error in void function
    } else {
        std::cout << value;  // ❌ May not have operator<<
    }
}
```

**if constexpr (compile-time):**
```cpp
template<typename T>
auto process(T value) {
    if constexpr (std::is_integral_v<T>) {
        return value * 2;  // Only compiled for integral
    } else {
        return value;      // Only compiled for non-integral
    }
}

process(10);    // Returns 20
process(3.14);  // Returns 3.14
```

**Discarded branch not compiled:**
```cpp
template<typename T>
void print(T value) {
    if constexpr (std::is_pointer_v<T>) {
        std::cout << *value;  // Only if pointer
    } else {
        std::cout << value;   // Only if not pointer
    }
}

print(42);   // ✅ pointer branch not compiled
int* p = nullptr;
print(p);    // ✅ non-pointer branch not compiled
```

**Multiple conditions:**
```cpp
template<typename T>
void process(T value) {
    if constexpr (std::is_integral_v<T>) {
        std::cout << "Integer: " << value;
    } else if constexpr (std::is_floating_point_v<T>) {
        std::cout << "Float: " << value;
    } else if constexpr (std::is_pointer_v<T>) {
        std::cout << "Pointer: " << *value;
    } else {
        std::cout << "Other";
    }
}
```

---

### 40. **What is `std::variant` and how does it compare to unions?**

`std::variant` is a type-safe union that knows which type it holds.

**Basic usage:**
```cpp
std::variant<int, double, std::string> var;

var = 42;       // Holds int
var = 3.14;     // Holds double
var = "hello";  // Holds std::string
```

**Accessing values:**

**1. std::get (throws):**
```cpp
std::variant<int, std::string> var = 42;

int i = std::get<int>(var);  // OK
// int i = std::get<0>(var);  // By index

try {
    auto s = std::get<std::string>(var);  // Throws
} catch (std::bad_variant_access&) {
}
```

**2. std::get_if (returns nullptr):**
```cpp
if (auto* p = std::get_if<int>(&var)) {
    std::cout << *p;
}
```

**3. std::visit:**
```cpp
std::variant<int, double, std::string> var = 42;

std::visit([](auto&& arg) {
    using T = std::decay_t<decltype(arg)>;
    if constexpr (std::is_same_v<T, int>) {
        std::cout << "int: " << arg;
    } else if constexpr (std::is_same_v<T, double>) {
        std::cout << "double: " << arg;
    } else {
        std::cout << "string: " << arg;
    }
}, var);
```

**Comparison with unions:**

**Union (not type-safe):**
```cpp
union Data {
    int i;
    double d;
};

Data data;
data.i = 42;
std::cout << data.d;  // ⚠️ UB! Reading wrong member
```

**Variant (type-safe):**
```cpp
std::variant<int, double> var = 42;
std::cout << std::get<double>(var);  // ✅ Throws exception
```

**Benefits:**
1. Type safety
2. Works with complex types (std::string, std::vector)
3. Proper construction/destruction
4. Exception on wrong access

---

### 41. **Explain `std::string_view` and its pitfalls**

`std::string_view` is a non-owning reference to a string.

**Basic usage:**
```cpp
void print(std::string_view sv) {
    std::cout << sv;  // No copy!
}

std::string str = "hello";
print(str);          // OK
print("world");      // OK
print(str.substr(0, 3));  // OK
```

**Benefits:**
```cpp
// Before - copies string
void process(const std::string& str);

// After - no copy, accepts any string-like type
void process(std::string_view sv);

process("literal");           // No temp string created
process(std::string("temp")); // No copy
process(str.substr(0, 5));    // No copy
```

**Pitfalls:**

**1. Dangling references:**
```cpp
std::string_view getView() {
    std::string temp = "hello";
    return temp;  // ❌ Dangling! temp destroyed
}

auto sv = getView();  // ⚠️ sv points to deleted memory
```

**2. Temporary lifetime:**
```cpp
std::string_view sv = std::string("temp");  // ❌ Temporary destroyed
std::cout << sv;  // ⚠️ Undefined behavior
```

**3. Not null-terminated:**
```cpp
std::string str = "hello world";
std::string_view sv = str.substr(0, 5);
const char* cstr = sv.data();  // ⚠️ Not null-terminated!
printf("%s", cstr);  // May print: "hello world"
```

**Safe usage:**
```cpp
// ✅ OK - string outlives view
std::string str = "hello";
std::string_view sv = str;
std::cout << sv;

// ✅ OK - literal has static lifetime
std::string_view sv = "hello";

// ✅ OK - parameter doesn't escape
void process(std::string_view sv) {
    std::cout << sv;  // Used immediately
}
```

---

### 42. **What are designated initializers (C++20)?**

Designated initializers allow initializing struct members by name.

**Basic usage:**
```cpp
struct Point {
    int x;
    int y;
    int z;
};

Point p{.x = 10, .y = 20, .z = 30};
```

**Benefits:**
```cpp
// Before - positional
Point p1{10, 20, 30};  // Which is x, y, z?

// After - explicit
Point p2{.x = 10, .y = 20, .z = 30};  // Clear!
```

**Can skip members (default-initialized):**
```cpp
Point p{.x = 10, .z = 30};  // y is 0
```

**Must be in order:**
```cpp
Point p{.z = 30, .x = 10};  // ❌ Error: wrong order
```

**Nested structs:**
```cpp
struct Rectangle {
    Point topLeft;
    Point bottomRight;
};

Rectangle r{
    .topLeft = {.x = 0, .y = 0},
    .bottomRight = {.x = 100, .y = 100}
};
```

**With inheritance (base first):**
```cpp
struct ColoredPoint : Point {
    int color;
};

ColoredPoint cp{
    .x = 10,
    .y = 20,
    .z = 30,
    .color = 0xFF0000
};
```

---

### 43. **Explain concepts (C++20) and their advantages over SFINAE**

Concepts define requirements on template parameters declaratively.

**Basic syntax:**
```cpp
template<typename T>
concept Integral = std::is_integral_v<T>;

template<Integral T>
T add(T a, T b) {
    return a + b;
}

add(1, 2);      // ✅ OK
add(1.5, 2.5);  // ❌ Error: doesn't satisfy Integral
```

**Compared to SFINAE:**

**Before (SFINAE):**
```cpp
template<typename T>
std::enable_if_t<std::is_integral_v<T>, T>
add(T a, T b) {
    return a + b;
}
```

**After (Concepts):**
```cpp
template<std::integral T>
T add(T a, T b) {
    return a + b;
}
```

**Custom concepts:**
```cpp
template<typename T>
concept Addable = requires(T a, T b) {
    { a + b } -> std::convertible_to<T>;
};

template<Addable T>
T add(T a, T b) {
    return a + b;
}
```

**Compound concepts:**
```cpp
template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

template<Numeric T>
T multiply(T a, T b) {
    return a * b;
}
```

**Requires clause:**
```cpp
template<typename T>
requires std::integral<T> && (sizeof(T) >= 4)
T process(T value) {
    return value * 2;
}
```

**Better error messages:**
```cpp
// SFINAE error: pages of template errors

// Concepts error:
// error: 'double' does not satisfy 'Integral'
```

---

### 44. **What is `std::span` (C++20) and when should you use it?**

`std::span` is a non-owning view over contiguous memory.

**Basic usage:**
```cpp
void process(std::span<int> data) {
    for (int& x : data) {
        x *= 2;
    }
}

std::vector<int> vec = {1, 2, 3};
int arr[] = {4, 5, 6};

process(vec);  // Works with vector
process(arr);  // Works with array
process(std::span(arr, 2));  // First 2 elements
```

**Replaces pointer + size:**

**Before:**
```cpp
void process(int* data, size_t size) {
    for (size_t i = 0; i < size; ++i) {
        data[i] *= 2;
    }
}

int arr[] = {1, 2, 3};
process(arr, 3);  // Easy to pass wrong size!
```

**After:**
```cpp
void process(std::span<int> data) {
    for (int& x : data) {
        x *= 2;
    }
}

int arr[] = {1, 2, 3};
process(arr);  // Size is automatic
```

**Fixed vs dynamic size:**
```cpp
void process_fixed(std::span<int, 3> data) {
    // Size known at compile-time
}

void process_dynamic(std::span<int> data) {
    // Size known at runtime
}

int arr[3] = {1, 2, 3};
process_fixed(arr);   // ✅ Size matches
process_dynamic(arr); // ✅ Also works
```

**Subspans:**
```cpp
std::vector<int> vec = {1, 2, 3, 4, 5};
std::span<int> sp(vec);

auto first3 = sp.first(3);     // {1, 2, 3}
auto last2 = sp.last(2);       // {4, 5}
auto middle = sp.subspan(1, 3); // {2, 3, 4}
```

**Const span:**
```cpp
void read_only(std::span<const int> data) {
    for (int x : data) {
        std::cout << x;
    }
}
```

---

## Concurrency & Multithreading (Questions 45-50)

### 45. **Explain the differences between `std::mutex`, `std::recursive_mutex`, and `std::shared_mutex`**

**`std::mutex`:**
Basic mutual exclusion, single lock per thread.

```cpp
std::mutex mtx;

void critical_section() {
    std::lock_guard<std::mutex> lock(mtx);
    // Only one thread here at a time
}
```

**`std::recursive_mutex`:**
Can be locked multiple times by same thread.

```cpp
std::recursive_mutex rmtx;

void func1() {
    std::lock_guard lock(rmtx);
    func2();  // OK - same thread can lock again
}

void func2() {
    std::lock_guard lock(rmtx);  // Won't deadlock
    // ...
}
```

**`std::shared_mutex` (C++17):**
Reader-writer lock - multiple readers OR single writer.

```cpp
std::shared_mutex smtx;
std::map<int, std::string> cache;

// Multiple readers
std::string read(int key) {
    std::shared_lock lock(smtx);  // Shared access
    return cache[key];
}

// Single writer
void write(int key, std::string value) {
    std::unique_lock lock(smtx);  // Exclusive access
    cache[key] = value;
}
```

**Comparison:**
```cpp
// mutex: Thread 1 locks → Thread 2 blocks
// recursive_mutex: Thread 1 locks twice → OK
// shared_mutex: Thread 1 read, Thread 2 read → Both OK
//               Thread 1 write, Thread 2 read → Thread 2 blocks
```

---

### 46. **What is `std::condition_variable` and how does it relate to spurious wakeups?**

`std::condition_variable` allows threads to wait for notifications.

**Basic usage:**
```cpp
std::mutex mtx;
std::condition_variable cv;
bool ready = false;

// Waiting thread
void wait_for_signal() {
    std::unique_lock lock(mtx);
    cv.wait(lock, []{ return ready; });  // Wait with predicate
    // Proceeds when ready == true
}

// Notifying thread
void send_signal() {
    {
        std::lock_guard lock(mtx);
        ready = true;
    }
    cv.notify_one();
}
```

**Spurious wakeups:**
`wait()` can return without `notify()` being called.

**Wrong - without predicate:**
```cpp
std::unique_lock lock(mtx);
cv.wait(lock);  // ⚠️ Might wake up spuriously!
if (ready) {    // Might miss this check
    process();
}
```

**Correct - with predicate:**
```cpp
std::unique_lock lock(mtx);
cv.wait(lock, []{ return ready; });  // Loops internally
process();  // Guaranteed ready == true
```

**Implementation (what predicate does):**
```cpp
// cv.wait(lock, predicate) is equivalent to:
while (!predicate()) {
    cv.wait(lock);
}
```

**Producer-consumer example:**
```cpp
std::queue<int> queue;
std::mutex mtx;
std::condition_variable cv;

void producer() {
    for (int i = 0; i < 10; ++i) {
        {
            std::lock_guard lock(mtx);
            queue.push(i);
        }
        cv.notify_one();
    }
}

void consumer() {
    while (true) {
        std::unique_lock lock(mtx);
        cv.wait(lock, []{ return !queue.empty(); });
        int value = queue.front();
        queue.pop();
        lock.unlock();
        process(value);
    }
}
```

---

### 47. **Explain deadlock and how to prevent it**

Deadlock occurs when threads wait for each other's resources circularly.

**Classic deadlock example:**
```cpp
std::mutex mtx1, mtx2;

// Thread 1
void func1() {
    std::lock_guard lock1(mtx1);
    // ... Thread 2 gets mtx2
    std::lock_guard lock2(mtx2);  // Waits for mtx2
}

// Thread 2
void func2() {
    std::lock_guard lock2(mtx2);
    // ... Thread 1 gets mtx1
    std::lock_guard lock1(mtx1);  // Waits for mtx1
}
// Deadlock!
```

**Prevention strategies:**

**1. Lock ordering:**
```cpp
// Always lock in same order
void func1() {
    std::lock_guard lock1(mtx1);  // Lock 1 first
    std::lock_guard lock2(mtx2);  // Then lock 2
}

void func2() {
    std::lock_guard lock1(mtx1);  // Same order
    std::lock_guard lock2(mtx2);
}
```

**2. `std::lock` for multiple mutexes:**
```cpp
void func() {
    std::scoped_lock lock(mtx1, mtx2);  // Locks both atomically
    // No deadlock possible
}

// Or manually:
void func() {
    std::unique_lock lock1(mtx1, std::defer_lock);
    std::unique_lock lock2(mtx2, std::defer_lock);
    std::lock(lock1, lock2);  // Deadlock-free
}
```

**3. Timeout-based locking:**
```cpp
void func() {
    std::unique_lock lock1(mtx1);
    if (auto lock2 = std::unique_lock(mtx2, std::try_to_lock)) {
        // Both locked
    } else {
        // Couldn't get mtx2, release mtx1 and retry
    }
}
```

**4. Lock-free data structures:**
```cpp
std::atomic<int> counter{0};
// No locks, no deadlock
```

---

### 48. **What is `std::future` and `std::promise`?**

`std::promise` sets a value, `std::future` retrieves it. Enables thread communication.

**Basic usage:**
```cpp
std::promise<int> promise;
std::future<int> future = promise.get_future();

// Thread 1 - compute and set value
std::thread t([&promise]() {
    int result = expensive_computation();
    promise.set_value(result);
});

// Thread 2 - wait for and get value
int value = future.get();  // Blocks until ready
std::cout << value;

t.join();
```

**With `std::async`:**
```cpp
// Simpler alternative
std::future<int> future = std::async(std::launch::async, 
    expensive_computation);

int value = future.get();  // Wait for result
```

**Exception handling:**
```cpp
std::promise<int> promise;
std::future<int> future = promise.get_future();

std::thread t([&promise]() {
    try {
        int result = might_throw();
        promise.set_value(result);
    } catch (...) {
        promise.set_exception(std::current_exception());
    }
});

try {
    int value = future.get();
} catch (const std::exception& e) {
    std::cerr << "Caught: " << e.what();
}

t.join();
```

**Multiple readers with `std::shared_future`:**
```cpp
std::promise<int> promise;
std::shared_future<int> future = promise.get_future().share();

// Multiple threads can call get()
std::thread t1([future]() {
    int val = future.get();  // OK
});

std::thread t2([future]() {
    int val = future.get();  // Also OK
});
```

**Checking status:**
```cpp
std::future<int> future = std::async(computation);

if (future.wait_for(std::chrono::seconds(1)) == 
    std::future_status::ready) {
    int value = future.get();
} else {
    std::cout << "Still computing...\n";
}
```

---

### 49. **Explain memory barriers and their role in thread synchronization**

Memory barriers prevent instruction reordering and ensure visibility between threads.

**Problem without barriers:**
```cpp
// Thread 1
data = 42;
ready = true;

// Thread 2
if (ready) {
    std::cout << data;  // Might see old value!
}
// CPU/compiler can reorder operations
```

**With memory barriers (acquire-release):**
```cpp
std::atomic<bool> ready{false};
int data;

// Thread 1
data = 42;
ready.store(true, std::memory_order_release);  // Release barrier

// Thread 2
if (ready.load(std::memory_order_acquire)) {  // Acquire barrier
    std::cout << data;  // Guaranteed to see 42
}
```

**Types of barriers:**

**1. Acquire barrier:** Prevents reordering of loads after the barrier
```cpp
// All loads after can't move before
std::atomic_thread_fence(std::memory_order_acquire);
```

**2. Release barrier:** Prevents reordering of stores before the barrier
```cpp
// All stores before can't move after
std::atomic_thread_fence(std::memory_order_release);
```

**3. Full barrier (seq_cst):**
```cpp
// Strongest - both directions
std::atomic_thread_fence(std::memory_order_seq_cst);
```

**Example - protecting multiple variables:**
```cpp
int x, y, z;
std::atomic<bool> ready{false};

// Writer
x = 1;
y = 2;
z = 3;
std::atomic_thread_fence(std::memory_order_release);
ready.store(true, std::memory_order_relaxed);

// Reader
while (!ready.load(std::memory_order_relaxed));
std::atomic_thread_fence(std::memory_order_acquire);
// All writes visible here
assert(x == 1 && y == 2 && z == 3);
```

---

### 50. **What is thread_local storage and when should you use it?**

`thread_local` variables have separate instances per thread.

**Basic usage:**
```cpp
thread_local int counter = 0;

void increment() {
    ++counter;  // Each thread has its own counter
}

std::thread t1([]() {
    increment();
    increment();
    std::cout << counter;  // Prints: 2
});

std::thread t2([]() {
    increment();
    std::cout << counter;  // Prints: 1
});
```

**Use cases:**

**1. Thread-specific random generators:**
```cpp
thread_local std::mt19937 rng(std::random_device{}());

int random_number() {
    std::uniform_int_distribution<int> dist(1, 100);
    return dist(rng);  // Thread-safe, no locking
}
```

**2. Thread-specific caching:**
```cpp
thread_local std::unordered_map<Key, Value> cache;

Value get_or_compute(const Key& key) {
    if (auto it = cache.find(key); it != cache.end()) {
        return it->second;
    }
    Value val = expensive_computation(key);
    cache[key] = val;
    return val;
}
```

**3. Thread ID tracking:**
```cpp
thread_local int thread_id = []() {
    static std::atomic<int> next_id{0};
    return next_id++;
}();

void log(const std::string& msg) {
    std::cout << "[Thread " << thread_id << "] " << msg << "\n";
}
```

**Lifetime:**
```cpp
thread_local std::string name;

void worker() {
    name = "Worker";
    // name exists for lifetime of thread
}  // name destroyed when thread exits

std::thread t(worker);
t.join();  // name already destroyed
```

**With classes:**
```cpp
class ThreadData {
public:
    ThreadData() {
        std::cout << "Constructed in thread " 
                  << std::this_thread::get_id() << "\n";
    }
    ~ThreadData() {
        std::cout << "Destroyed in thread " 
                  << std::this_thread::get_id() << "\n";
    }
};

thread_local ThreadData data;  // Constructed on first use per thread
```

---

## RAII & Smart Pointers (Questions 51-58)

### 51. **Explain RAII and why it's considered a fundamental C++ idiom**

RAII (Resource Acquisition Is Initialization) ties resource lifetime to object lifetime.

**Principle:**
- Acquire resources in constructor
- Release resources in destructor
- Automatic cleanup via destructors

**Problem without RAII:**
```cpp
void process() {
    FILE* f = fopen("file.txt", "r");
    if (error_condition) {
        return;  // ❌ Leak! Forgot to close
    }
    // ... work ...
    fclose(f);
}
```

**Solution with RAII:**
```cpp
class FileHandle {
    FILE* file_;
public:
    FileHandle(const char* name, const char* mode) 
        : file_(fopen(name, mode)) {
        if (!file_) throw std::runtime_error("Can't open file");
    }
    
    ~FileHandle() {
        if (file_) fclose(file_);  // Always called
    }
    
    FILE* get() { return file_; }
};

void process() {
    FileHandle f("file.txt", "r");
    if (error_condition) {
        return;  // ✅ File automatically closed
    }
    // ... work with f.get() ...
}  // File closed here too
```

**Standard RAII examples:**

**1. Locks:**
```cpp
std::mutex mtx;

void critical_section() {
    std::lock_guard lock(mtx);  // Locks in constructor
    // ... critical section ...
}  // Unlocks in destructor
```

**2. Memory:**
```cpp
void process() {
    std::unique_ptr<int> ptr(new int(42));
    // ... use ptr ...
}  // Memory automatically freed
```

**3. Containers:**
```cpp
void process() {
    std::vector<int> vec;
    vec.push_back(1);
    // ... use vec ...
}  // Memory automatically cleaned up
```

**Exception safety:**
```cpp
void process() {
    std::lock_guard lock1(mtx1);
    std::unique_ptr<Data> data(new Data);
    std::lock_guard lock2(mtx2);
    
    throw std::exception();  // All resources cleaned up!
}  // lock2 unlocked, data deleted, lock1 unlocked
```

---

### 52. **When would you use `std::unique_ptr` vs `std::shared_ptr` vs raw pointers?**

**`std::unique_ptr`:** Exclusive ownership, zero overhead.
```cpp
std::unique_ptr<Widget> ptr = std::make_unique<Widget>();
// Movable but not copyable
auto ptr2 = std::move(ptr);  // Transfer ownership
```

**Use when:**
- Single owner
- Ownership transfer needed
- Zero overhead required

**`std::shared_ptr`:** Shared ownership with reference counting.
```cpp
std::shared_ptr<Widget> ptr1 = std::make_shared<Widget>();
std::shared_ptr<Widget> ptr2 = ptr1;  // Both own it
// Deleted when last shared_ptr destroyed
```

**Use when:**
- Multiple owners
- Unclear ownership lifetimes
- Caching, callbacks

**Raw pointers:** Non-owning observers.
```cpp
void process(Widget* widget) {
    // Doesn't own widget, just uses it
    widget->doSomething();
}

std::unique_ptr<Widget> owner = std::make_unique<Widget>();
process(owner.get());  // Pass non-owning pointer
```

**Use when:**
- Observing, not owning
- Function parameters (doesn't take ownership)
- Performance-critical code with managed ownership elsewhere

**Comparison:**
```cpp
// unique_ptr - exclusive ownership
std::unique_ptr<File> file = openFile("data.txt");
// Only one owner, moves ownership

// shared_ptr - shared ownership
std::shared_ptr<Cache> cache = getCache();
// Multiple owners, reference counted

// raw pointer - non-owning
void process(Data* data) {
    // Just observing, someone else owns it
}
```

---

### 53. **What is `std::make_unique` and why is it preferred over `new`?**

`std::make_unique<T>(args...)` creates a `unique_ptr` safely.

**Basic usage:**
```cpp
// With make_unique (C++14)
auto ptr = std::make_unique<Widget>(arg1, arg2);

// Equivalent to:
std::unique_ptr<Widget> ptr(new Widget(arg1, arg2));
```

**Advantages:**

**1. Exception safety:**
```cpp
// Unsafe
func(std::unique_ptr<Widget>(new Widget()), might_throw());
// Order of evaluation undefined!
// If might_throw() evaluated first and throws, new Widget leaks

// Safe
func(std::make_unique<Widget>(), might_throw());
// make_unique is atomic operation
```

**2. Shorter syntax:**
```cpp
// Verbose
std::unique_ptr<LongTypeName> ptr(new LongTypeName(args));

// Concise
auto ptr = std::make_unique<LongTypeName>(args);
```

**3. No type repetition:**
```cpp
// DRY violation
std::unique_ptr<Widget> ptr(new Widget());

// Better
auto ptr = std::make_unique<Widget>();
```

**Arrays:**
```cpp
auto arr = std::make_unique<int[]>(10);  // Array of 10 ints
arr[0] = 42;
```

**Custom deleters (need manual construction):**
```cpp
// make_unique doesn't support custom deleters
auto ptr = std::unique_ptr<FILE, decltype(&fclose)>(
    fopen("file.txt", "r"), 
    &fclose
);
```

---

### 54. **Explain the control block in `std::shared_ptr` and its performance implications**

`std::shared_ptr` uses a control block containing:
- Reference count
- Weak count  
- Deleter
- Allocator

**Memory layout:**
```cpp
std::shared_ptr<Widget> ptr = std::make_shared<Widget>();

// Memory layout with make_shared:
// [Control Block | Widget Object]
// Single allocation

std::shared_ptr<Widget> ptr2(new Widget());

// Memory layout without make_shared:
// [Control Block] -> [Widget Object]
// Two allocations
```

**Performance implications:**

**1. Atomic operations:**
```cpp
std::shared_ptr<int> ptr1 = std::make_shared<int>(42);
std::shared_ptr<int> ptr2 = ptr1;  // Atomic increment
// Each copy/move involves atomic operations
```

**2. Extra indirection:**
```cpp
ptr->method();
// 1. Load control block pointer
// 2. Follow to object
// 3. Call method
// vs unique_ptr: direct pointer to object
```

**3. make_shared optimization:**
```cpp
// One allocation
auto ptr1 = std::make_shared<Widget>();

// Two allocations
std::shared_ptr<Widget> ptr2(new Widget());
```

**Memory overhead:**
```cpp
sizeof(std::shared_ptr<int>)  // 16 bytes (2 pointers)
sizeof(std::unique_ptr<int>)  // 8 bytes (1 pointer)

// Control block: ~24 bytes
// - Reference count: 8 bytes
// - Weak count: 8 bytes
// - Pointers for deleter/allocator: 8 bytes
```

**Why make_shared is faster:**
```cpp
// make_shared: 1 allocation
auto ptr = std::make_shared<Widget>();

// new then shared_ptr: 2 allocations
std::shared_ptr<Widget> ptr(new Widget());
```

---

### 55. **What's the difference between `std::shared_ptr` and `std::weak_ptr`?**

**`std::shared_ptr`:** Owns the object, increments reference count.
**`std::weak_ptr`:** Observes the object, doesn't increment reference count.

**Basic usage:**
```cpp
std::shared_ptr<int> shared = std::make_shared<int>(42);
std::weak_ptr<int> weak = shared;  // Doesn't increase ref count

std::cout << shared.use_count();  // 1 (not 2!)

// Access through weak_ptr
if (auto locked = weak.lock()) {  // Returns shared_ptr
    std::cout << *locked;  // 42
} else {
    std::cout << "Object deleted";
}
```

**Preventing circular references:**
```cpp
// Circular reference problem
class Node {
    std::shared_ptr<Node> parent;  // ❌ Circular reference
    std::shared_ptr<Node> child;
};

std::shared_ptr<Node> p = std::make_shared<Node>();
std::shared_ptr<Node> c = std::make_shared<Node>();
p->child = c;
c->parent = p;  // Circular! Never deleted

// Solution with weak_ptr
class Node {
    std::weak_ptr<Node> parent;  // ✅ Breaks cycle
    std::shared_ptr<Node> child;
};
```

**Caching example:**
```cpp
class Cache {
    std::map<Key, std::weak_ptr<Value>> cache_;
    
public:
    std::shared_ptr<Value> get(const Key& key) {
        auto it = cache_.find(key);
        if (it != cache_.end()) {
            if (auto value = it->second.lock()) {
                return value;  // Cache hit
            }
            cache_.erase(it);  // Expired, remove
        }
        
        // Cache miss, create new
        auto value = std::make_shared<Value>(load(key));
        cache_[key] = value;
        return value;
    }
};
```

**Observer pattern:**
```cpp
class Subject {
    std::vector<std::weak_ptr<Observer>> observers_;
    
public:
    void notify() {
        // Remove expired observers while notifying
        observers_.erase(
            std::remove_if(observers_.begin(), observers_.end(),
                [](auto& weak) { 
                    if (auto obs = weak.lock()) {
                        obs->update();
                        return false;
                    }
                    return true;  // Expired, remove
                }),
            observers_.end()
        );
    }
};
```

---

### 56. **How do custom deleters work with smart pointers?**

**`std::unique_ptr<T, Deleter>`:** Deleter is part of type.
```cpp
// Custom deleter for FILE*
auto deleter = [](FILE* f) { 
    if (f) fclose(f); 
};

std::unique_ptr<FILE, decltype(deleter)> file(
    fopen("file.txt", "r"),
    deleter
);
// File automatically closed
```

**Zero overhead with stateless deleters:**
```cpp
struct FileDeleter {
    void operator()(FILE* f) const {
        if (f) fclose(f);
    }
};

// Empty base optimization - no size overhead
std::unique_ptr<FILE, FileDeleter> file(fopen("file.txt", "r"));
```

**`std::shared_ptr<T>`:** Deleter is type-erased (stored in control block).
```cpp
// Lambda deleter
std::shared_ptr<FILE> file(
    fopen("file.txt", "r"),
    [](FILE* f) { if (f) fclose(f); }
);

// Different deleters, same type!
std::shared_ptr<FILE> file2(
    fopen("file2.txt", "r"),
    &fclose
);

// Both are std::shared_ptr<FILE>
```

**SDL example:**
```cpp
std::shared_ptr<SDL_Window> window(
    SDL_CreateWindow("Title", 0, 0, 800, 600, 0),
    SDL_DestroyWindow
);

std::shared_ptr<SDL_Renderer> renderer(
    SDL_CreateRenderer(window.get(), -1, 0),
    SDL_DestroyRenderer
);
```

**Custom resource example:**
```cpp
class DatabaseConnection {
public:
    static void close(DatabaseConnection* conn) {
        if (conn) {
            conn->disconnect();
            delete conn;
        }
    }
};

std::unique_ptr<DatabaseConnection, decltype(&DatabaseConnection::close)> 
    conn(new DatabaseConnection(), &DatabaseConnection::close);
```

---

### 57. **Explain the `std::enable_shared_from_this` pattern**

Allows object to create `shared_ptr` to itself.

**Problem:**
```cpp
class Widget {
public:
    std::shared_ptr<Widget> getShared() {
        return std::shared_ptr<Widget>(this);  // ❌ Wrong!
        // Creates new control block, double delete!
    }
};
```

**Solution:**
```cpp
class Widget : public std::enable_shared_from_this<Widget> {
public:
    std::shared_ptr<Widget> getShared() {
        return shared_from_this();  // ✅ Correct
    }
};

std::shared_ptr<Widget> w = std::make_shared<Widget>();
std::shared_ptr<Widget> w2 = w->getShared();  // Same control block
```

**Async callback example:**
```cpp
class AsyncTask : public std::enable_shared_from_this<AsyncTask> {
public:
    void start() {
        std::thread([self = shared_from_this()]() {
            self->doWork();  // Keeps object alive during async work
        }).detach();
    }
    
private:
    void doWork() {
        // Work here...
    }
};

auto task = std::make_shared<AsyncTask>();
task->start();  // Task stays alive even if 'task' goes out of scope
```

**Requirements:**
```cpp
auto w = std::make_shared<Widget>();
w->getShared();  // ✅ OK

Widget w2;
// w2.getShared();  // ❌ Throws std::bad_weak_ptr
// Must be owned by shared_ptr first!
```

---

### 58. **What are the dangers of circular references with `std::shared_ptr` and how do you break them?**

Circular references prevent deallocation when reference counts never reach zero.

**Problem:**
```cpp
class Node {
public:
    std::shared_ptr<Node> next;
    std::shared_ptr<Node> prev;
};

auto n1 = std::make_shared<Node>();
auto n2 = std::make_shared<Node>();

n1->next = n2;  // n2 ref count = 2
n2->prev = n1;  // n1 ref count = 2

// n1 and n2 go out of scope
// But ref counts never reach 0!
// Memory leak!
```

**Solution - use `weak_ptr`:**
```cpp
class Node {
public:
    std::shared_ptr<Node> next;
    std::weak_ptr<Node> prev;  // ✅ Breaks cycle
};

auto n1 = std::make_shared<Node>();
auto n2 = std::make_shared<Node>();

n1->next = n2;  // n2 ref count = 2
n2->prev = n1;  // n1 ref count = 1 (weak doesn't count)

// n1 and n2 go out of scope
// n1 deleted (ref count -> 0)
// n2 deleted (ref count -> 0)
```

**Parent-child pattern:**
```cpp
class Node {
    std::shared_ptr<Node> child_;  // Parent owns child
    std::weak_ptr<Node> parent_;   // Child observes parent
    
public:
    void setChild(std::shared_ptr<Node> child) {
        child_ = child;
        child->parent_ = shared_from_this();
    }
    
    std::shared_ptr<Node> getParent() {
        return parent_.lock();  // May return null if parent deleted
    }
};
```

**Detecting cycles:**
```cpp
// Check reference count
auto ptr = std::make_shared<Node>();
std::cout << ptr.use_count();  // Should be 1

auto ptr2 = ptr;
std::cout << ptr.use_count();  // 2

// If count stays high after going out of scope, possible cycle
```

---

## Obscure Language Features (Questions 59-60)

### 59. **Explain the Most Vexing Parse problem**

Most Vexing Parse: C++ prioritizes function declarations when ambiguous.

**Problem:**
```cpp
class Widget {
public:
    Widget() {}
};

Widget w(Widget());  // ❌ Not object construction!
                     // Declares function w that returns Widget
                     // Takes function pointer returning Widget
```

**More examples:**
```cpp
std::mutex m();  // ❌ Function declaration, not object!

int x(int(y));   // ❌ Declares function x taking int and returning int

TimeKeeper time_keeper(Timer());  // ❌ Function declaration
```

**Solutions:**

**1. Uniform initialization (C++11):**
```cpp
Widget w{Widget{}};  // ✅ Object construction
std::mutex m{};      // ✅ Object construction
```

**2. Extra parentheses:**
```cpp
Widget w((Widget()));  // ✅ Object construction
```

**3. Copy initialization:**
```cpp
Widget w = Widget();  // ✅ Object construction
```

**Why it happens:**
C++ grammar prefers function declarations to resolve ambiguity.

```cpp
// Ambiguous: object or function?
Type name(Type2());

// Interpreted as function:
Type name(Type2 (*)());  // Function pointer parameter
```

---

### 60. **What is the difference between `struct` and `class` beyond default access?**

Only difference: default member and inheritance access.

**Default member access:**
```cpp
struct S {
    int x;  // public by default
};

class C {
    int x;  // private by default
};
```

**Default inheritance:**
```cpp
struct Derived1 : Base {  // public inheritance by default
};

class Derived2 : Base {   // private inheritance by default
};
```

**Otherwise identical:**
```cpp
struct S {
private:  // Can have private members
    int x;
public:
    void method();  // Can have methods
    virtual void vfunc();  // Can have virtual functions
};

class C {
public:  // Can have public members
    int x;
};
```

**Convention:**
```cpp
// Use struct for POD-like data
struct Point {
    int x, y;
};

// Use class for objects with behavior
class Widget {
public:
    void doSomething();
private:
    int data_;
};
```

**No performance difference:**
Both compile to identical code.

---

