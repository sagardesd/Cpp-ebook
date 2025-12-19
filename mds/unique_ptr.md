# std::unique_ptr<T>

## Table of Contents
1. [What is std::unique_ptr<T>?](#what-is-uniqueptrt)
2. [Declaration in C++ Standard](#declaration-in-c-standard)
3. [Creating a std::unique_ptr<T>](#creating-a-uniqueptrt)
4. [Non-Copyable Semantics](#non-copyable-semantics)
5. [Move Semantics](#move-semantics)
6. [Custom Deleters](#custom-deleters)
7. [Array Allocation](#array-allocation)
8. [std::make_unique<T> (C++14)](#stdmake_uniquet-c14)
9. [Limitations of std::make_unique](#limitations-of-stdmake_unique)

---

## What is std::unique_ptr<T>?

`std::unique_ptr<T>` is a smart pointer that manages a resource (which may be memory, a file handle, a socket, or a hardware mutex) through **exclusive ownership**. It acts as an RAII (Resource Acquisition Is Initialization) wrapper that guarantees the resource is released—via a deleter—exactly once: either when the `unique_ptr<T>` object goes out of scope or when it is reassigned.

Key characteristics:
- **Exclusive Ownership**: Only one `unique_ptr` can own a given resource at any time
- **Resource Management**: Manages any resource, not just dynamically allocated memory (files, sockets, hardware resources, etc.)
- **Guaranteed Cleanup**: The resource is released exactly once through the deleter when the `unique_ptr` is destroyed or reassigned
- **Zero Overhead**: No reference counting; essentially a wrapper around a raw pointer with minimal overhead
- **Move-Only Semantics**: Cannot be copied (to enforce exclusive ownership), but can be moved to transfer ownership
- **RAII Principle**: Follows the Resource Acquisition Is Initialization pattern, binding resource lifetime to object lifetime

---

## Declaration in C++ Standard

According to the C++11 standard (and refined in later standards), `std::unique_ptr` is defined in the `<memory>` header:

```cpp
#include <memory>

// Basic declaration
template<class T, class D = std::default_delete<T>> class unique_ptr;

// Partial specialization for array types
template<class T, class D> class unique_ptr<T[], D>;
```

The template has two parameters:
- **T**: The type of the object being managed
- **D**: The deleter (defaults to `std::default_delete<T>`, which calls `delete` or `delete[]`)

---

## Creating a std::unique_ptr<T>

### Method 1: Using `new` (C++11)

```cpp
#include <memory>
#include <iostream>

class Dog {
public:
    Dog(const std::string& name) : name_(name) {
        std::cout << "Dog " << name_ << " created\n";
    }
    ~Dog() {
        std::cout << "Dog " << name_ << " destroyed\n";
    }
private:
    std::string name_;
};

int main() {
    // Create a unique_ptr using new
    std::unique_ptr<Dog> dog1(new Dog("Buddy"));
    
    // Access the object
    dog1->name();
    
    // When dog1 goes out of scope, the Dog is automatically deleted
    return 0;
}
```

### Method 2: Using `std::make_unique<T>` (C++14)

```cpp
int main() {
    // More safe and concise
    auto dog2 = std::make_unique<Dog>("Max");
    
    return 0;
}
```

### Explicit Type Declaration

```cpp
int main() {
    std::unique_ptr<Dog> dog3 = std::make_unique<Dog>("Charlie");
    std::unique_ptr<Dog> dog4{new Dog("Daisy")};
    
    return 0;
}
```

---

## Non-Copyable Semantics

`std::unique_ptr` **cannot be copied** because it enforces exclusive ownership. Only one `unique_ptr` should manage a given resource.

### What This Means

When you try to copy a `unique_ptr`, the compiler will complain and wont allow to copy.

```cpp
// Compilation ERROR!
std::unique_ptr<Dog> dog1 = std::make_unique<Dog>("Buddy");
std::unique_ptr<Dog> dog2 = dog1;  // COMPILER ERROR: copy constructor deleted

std::unique_ptr<Dog> dog3(dog1);   // COMPILER ERROR: copy constructor deleted

std::unique_ptr<Dog> dog4 = dog1;  // COMPILER ERROR: copy constructor deleted

std::vector<std::unique_ptr<Dog>> dogs;
dogs.push_back(dog1);              // COMPILER ERROR: cannot copy
```

### Why This Restriction Exists

```cpp
// Without this restriction, this would be problematic:
std::unique_ptr<Dog> dog1 = std::make_unique<Dog>("Buddy");
std::unique_ptr<Dog> dog2 = dog1;  // If copying were allowed...

// Now which one "owns" the Dog? Both?
// When dog1 goes out of scope, it deletes the Dog.
// When dog2 goes out of scope, it tries to delete the already-deleted Dog.
// Result: DOUBLE DELETE - memory corruption and crash!
```

### The Deleted Copy Operations

This is achieved by deleteing the copy constructor and copy assignment operator of `std::unique_ptr` class.

```cpp
// Simplified view of unique_ptr definition:
template<class T>
class unique_ptr {
public:
    // Copy operations are explicitly deleted
    unique_ptr(const unique_ptr&) = delete;
    unique_ptr& operator=(const unique_ptr&) = delete;
    
    // Move operations are available
    unique_ptr(unique_ptr&&) noexcept;
    unique_ptr& operator=(unique_ptr&&) noexcept;
    
    // ... rest of implementation
};
```

---

## Move Semantics

`std::unique_ptr` **can be moved**, which transfers ownership from one `unique_ptr` to another. When you move a `std::unique_ptr` object it invokes the `move semantic` special member functions.

### Basic Move Example

```cpp
int main() {
    std::unique_ptr<Dog> dog1 = std::make_unique<Dog>("Buddy");
    
    // Transfer ownership from dog1 to dog2
    std::unique_ptr<Dog> dog2 = std::move(dog1);
    
    // Now dog2 owns the Dog, dog1 is nullptr
    if (dog1 == nullptr) {
        std::cout << "dog1 is now null\n";  // This prints
    }
    
    // dog2 still owns the Dog
    // When dog2 goes out of scope, the Dog is deleted
    return 0;
}
```

### Using `std::move` Explicitly

```cpp
void processDog(std::unique_ptr<Dog> dog) {
    // Function takes ownership
    std::cout << "Processing dog...\n";
    // Dog is deleted when function returns
}

int main() {
    std::unique_ptr<Dog> myDog = std::make_unique<Dog>("Max");
    
    // Transfer ownership to the function
    processDog(std::move(myDog));
    
    // myDog is now nullptr
    std::cout << "myDog after transfer: " 
              << (myDog ? "valid" : "null") << "\n";  // Prints "null"
    
    return 0;
}
```

### Move in Return Values

```cpp
std::unique_ptr<Dog> createDog() {
    auto dog = std::make_unique<Dog>("NewDog");
    return dog;  // Automatically moved (RVO or move semantics)
}

int main() {
    std::unique_ptr<Dog> myDog = createDog();
    // No copy, no extra allocations - just a move
    
    return 0;
}
```

### Move with Containers

```cpp
int main() {
    std::vector<std::unique_ptr<Dog>> dogs;
    
    dogs.push_back(std::make_unique<Dog>("Buddy"));  // Moved into vector
    
    auto dog = std::make_unique<Dog>("Max");
    dogs.push_back(std::move(dog));                   // Explicitly moved
    
    // All dogs are automatically cleaned up when vector is destroyed
    return 0;
}
```

---

## Custom Deleters

By default, `std::unique_ptr<T>` uses `std::default_delete<T>`, which simply calls `delete` for pointers and `delete[]` for arrays. However, you can provide a custom deleter for specialized cleanup needs.

### Why Custom Deleters Are Needed

Custom deleters are necessary when:
1. **Resource management differs from `delete`**: File handles, database connections, memory allocated with `malloc`, etc.
2. **Cleanup requires additional operations**: Logging, reference counting, resource pool management
3. **Third-party library resources**: APIs that require specific deallocation functions

### Syntax for Custom Deleters

```cpp
// Template parameter specifies the deleter type
std::unique_ptr<T, DeleterType> ptr;
```

### Example 1: File Handle Wrapper

```cpp
#include <cstdio>
#include <memory>

// Custom deleter for FILE*
struct FileDeleter {
    void operator()(FILE* file) const {
        if (file) {
            std::cout << "Closing file...\n";
            std::fclose(file);
        }
    }
};

int main() {
    // FILE* requires fclose, not delete
    std::unique_ptr<FILE, FileDeleter> file(
        std::fopen("data.txt", "r")
    );
    
    if (file) {
        // Use the file
        char buffer[100];
        std::fgets(buffer, sizeof(buffer), file.get());
    }
    
    // FileDeleter is called automatically, closing the file
    return 0;
}
```

### Example 2: C API Resource

```cpp
#include <memory>
#include <iostream>

// Simulated C library
extern "C" {
    typedef struct {
        int* data;
        int size;
    } DataBuffer;
    
    DataBuffer* createBuffer(int size);
    void destroyBuffer(DataBuffer* buffer);
}

// Custom deleter for C API
auto bufferDeleter = [](DataBuffer* buf) {
    std::cout << "Destroying buffer via C API...\n";
    destroyBuffer(buf);
};

int main() {
    using BufferPtr = std::unique_ptr<DataBuffer, decltype(bufferDeleter)>;
    
    BufferPtr buffer(createBuffer(100), bufferDeleter);
    
    // Use buffer
    std::cout << "Buffer size: " << buffer->size << "\n";
    
    // destroyBuffer is called automatically
    return 0;
}
```

### Example 3: Lambda Deleter

```cpp
#include <memory>
#include <iostream>

class Resource {
public:
    Resource() { std::cout << "Resource acquired\n"; }
    ~Resource() { std::cout << "Resource destroyed\n"; }
};

int main() {
    // Lambda as custom deleter
    auto customDeleter = [](Resource* res) {
        std::cout << "Custom cleanup before deletion\n";
        delete res;
    };
    
    using ResourcePtr = std::unique_ptr<Resource, decltype(customDeleter)>;
    
    ResourcePtr res(new Resource(), customDeleter);
    
    // Output:
    // Resource acquired
    // Custom cleanup before deletion
    // Resource destroyed
    
    return 0;
}
```

### Example 4: Stateful Deleter

```cpp
#include <memory>
#include <iostream>

class MemoryPool {
private:
    int allocated_ = 0;
public:
    void* allocate(int size) {
        allocated_ += size;
        std::cout << "Allocated " << size << " bytes (total: " 
                  << allocated_ << ")\n";
        return new char[size];
    }
    
    void deallocate(void* ptr, int size) {
        allocated_ -= size;
        std::cout << "Deallocated " << size << " bytes (total: " 
                  << allocated_ << ")\n";
        delete[] static_cast<char*>(ptr);
    }
};

struct PoolDeleter {
    MemoryPool* pool;
    int size;
    
    void operator()(char* ptr) const {
        pool->deallocate(ptr, size);
    }
};

int main() {
    MemoryPool pool;
    
    const int SIZE = 256;
    char* raw = static_cast<char*>(pool.allocate(SIZE));
    
    std::unique_ptr<char[], PoolDeleter> buffer(
        raw,
        PoolDeleter{&pool, SIZE}
    );
    
    // Use buffer...
    
    // Deleter tracks deallocation through the pool
    return 0;
}
```

---

## Array Allocation

`std::unique_ptr` has a **partial specialization for arrays** (`unique_ptr<T[]>`), which uses `delete[]` instead of `delete`:

```cpp
#include <memory>
#include <iostream>

int main() {
    // Single object
    std::unique_ptr<int> single(new int(42));
    
    // Array of objects - use T[]
    std::unique_ptr<int[]> array(new int[100]);
    
    // Access via operator[]
    array[0] = 10;
    array[99] = 20;
    
    // Use make_unique for arrays (C++20)
    auto modern_array = std::make_unique<double[]>(50);
    modern_array[0] = 3.14;
    
    // Automatic cleanup with delete[]
    return 0;
}
```

### Array with Custom Deleter

```cpp
struct ArrayDeleter {
    void operator()(int* array) const {
        std::cout << "Deleting array with custom deleter...\n";
        delete[] array;
    }
};

int main() {
    std::unique_ptr<int[], ArrayDeleter> array(
        new int[100],
        ArrayDeleter{}
    );
    
    array[0] = 42;
    
    return 0;
}
```

---

## The Problem with Naked `new` and the Need for `std::make_unique<T>`

When using `new` directly with `unique_ptr`, there's a critical exception safety issue that can lead to resource leaks.

### Example: Exception Safety Problem

Consider this function that takes multiple `unique_ptr` parameters:

```cpp
#include <memory>
#include <iostream>

class Data {
public:
    Data() { std::cout << "Data created\n"; }
    ~Data() { std::cout << "Data destroyed\n"; }
};

class Config {
public:
    Config() { std::cout << "Config created\n"; }
    ~Config() { std::cout << "Config destroyed\n"; }
};

void processData(
    std::unique_ptr<Data> data,
    std::unique_ptr<Config> config
) {
    std::cout << "Processing...\n";
    // Process data and config
}

int main() {
    // UNSAFE: Can leak memory!
    processData(
        std::unique_ptr<Data>(new Data()),      // First allocation
        std::unique_ptr<Config>(new Config())   // Second allocation
    );
    
    return 0;
}
```

### Why This Is Dangerous

The C++ standard( < C++17) does **not guarantee** the order of evaluation of function arguments. Here's what could happen:

1. `new Data()` is called → allocates memory
2. `new Config()` is called → allocates memory
3. **Exception is thrown** (in Config constructor or elsewhere)
4. The `Data` object is deleted successfully (unique_ptr destructor runs)
5. But the `Config` allocation was partial → **MEMORY LEAK**

Or worse:

1. `new Data()` is called → allocates memory
2. **Exception is thrown** (in Data constructor)
3. No `unique_ptr` is constructed yet → **MEMORY LEAK** (raw pointer lost)

The problem is that memory allocation and `unique_ptr` construction are **not atomic**. Multiple intermediate states exist where resources can leak.

### The Solution: `std::make_unique<T>` (C++14)

`std::make_unique<T>` is a factory function that creates and wraps the object **atomically**. Either the entire operation succeeds and you have a fully constructed `unique_ptr`, or an exception is thrown before any allocation happens. There is no intermediate state where a resource can leak.

```cpp
#include <memory>

int main() {
    // SAFE: Atomic operation
    processData(
        std::make_unique<Data>(),
        std::make_unique<Config>()
    );
    
    return 0;
}
```

**Why it's atomic:**
- `std::make_unique` creates the object and immediately wraps it in a `unique_ptr`
- Either both succeed together, or nothing succeeds
- No raw pointers exist in intermediate states
- No possibility of a leak between allocation and `unique_ptr` construction

### Detailed Example: Demonstrating the Safety

```cpp
#include <memory>
#include <iostream>

class FailingObject {
public:
    FailingObject() {
        std::cout << "FailingObject constructor started\n";
        throw std::runtime_error("Constructor failed!");
        std::cout << "FailingObject constructor completed\n";
    }
    ~FailingObject() { std::cout << "FailingObject destroyed\n"; }
};

class SafeObject {
public:
    SafeObject() { std::cout << "SafeObject created\n"; }
    ~SafeObject() { std::cout << "SafeObject destroyed\n"; }
};

void unsafeWay() {
    std::cout << "\n=== UNSAFE WAY (with new) ===\n";
    try {
        auto obj1 = std::make_unique<SafeObject>();
        // If exception occurs here, obj1 is properly cleaned up
        // But if we had: ptr(new SafeObject()), ptr(new FailingObject())
        // We could have a leak
        auto obj2 = std::make_unique<FailingObject>();
    } catch (const std::exception& e) {
        std::cout << "Exception caught: " << e.what() << "\n";
    }
    std::cout << "End of unsafeWay\n";
}

void safeWay() {
    std::cout << "\n=== SAFE WAY (with make_unique) ===\n";
    try {
        auto obj1 = std::make_unique<SafeObject>();
        auto obj2 = std::make_unique<FailingObject>();
    } catch (const std::exception& e) {
        std::cout << "Exception caught: " << e.what() << "\n";
    }
    std::cout << "End of safeWay\n";
}

int main() {
    unsafeWay();
    safeWay();
    return 0;
}
```

---

## std::make_unique<T> (C++14)

`std::make_unique<T>` (introduced in C++14) is a factory function that creates a `unique_ptr` **atomically** and more safely than using `new` directly.

### Syntax

```cpp
// Single object
auto ptr = std::make_unique<T>(args...);

// Array (C++20)
auto arr = std::make_unique<T[]>(size);
```

### Key Advantage: Atomic Construction

```cpp
#include <memory>
#include <string>
#include <iostream>

class Person {
public:
    Person(const std::string& name, int age)
        : name_(name), age_(age) {
        std::cout << "Person created: " << name_ << "\n";
    }
    ~Person() {
        std::cout << "Person destroyed: " << name_ << "\n";
    }
    
    void display() const {
        std::cout << name_ << " is " << age_ << " years old\n";
    }
    
private:
    std::string name_;
    int age_;
};

void processPersons(
    std::unique_ptr<Person> person1,
    std::unique_ptr<Person> person2
) {
    // Process persons
}

int main() {
    // SAFE: Each make_unique is atomic
    // Either person is created or an exception is thrown
    // No intermediate state with leaked resources
    processPersons(
        std::make_unique<Person>("Alice", 30),
        std::make_unique<Person>("Bob", 25)
    );
    
    // In containers
    std::vector<std::unique_ptr<Person>> people;
    people.push_back(std::make_unique<Person>("Carol", 28));
    people.push_back(std::make_unique<Person>("Dave", 35));
    
    for (const auto& p : people) {
        p->display();
    }
    
    return 0;
}
```

### Benefits of std::make_unique

1. **Exception Safety**: Atomic operation - either succeeds completely or fails without leaking
2. **Less Typing**: More concise than `std::unique_ptr<T>(new T(...))`
3. **Type Deduction**: `auto` can deduce the full type
4. **Consistency**: Encourages uniform resource management patterns

---

## Limitations of std::make_unique

`std::make_unique` uses the **default deleter** and has several limitations:

### 1. Cannot Use Custom Deleters

```cpp
// ERROR: make_unique doesn't support custom deleters
FILE* file = std::fopen("data.txt", "r");

// This won't compile:
auto filePtr = std::make_unique<FILE, FileDeleter>(file);  // ERROR!

// Must use new directly:
std::unique_ptr<FILE, FileDeleter> filePtr(file, FileDeleter{});
```

### 2. Cannot Use with Pre-existing Pointers

```cpp
int* raw = new int(42);

// ERROR: make_unique creates a new object, can't wrap existing pointer
auto ptr = std::make_unique<int>(raw);  // Creates new int, not what we want

// Must use new directly:
auto ptr = std::unique_ptr<int>(raw);
```

### 3. Private Constructors (Indirect Limitation)

```cpp
class Secret {
private:
    Secret(int value) : value_(value) {}
    friend class SecretFactory;
    int value_;
};

// ERROR: make_unique can't access private constructor
auto secret = std::make_unique<Secret>(42);  // Won't compile

// Workaround: Use new with a friend function
class SecretFactory {
public:
    static std::unique_ptr<Secret> create(int value) {
        return std::unique_ptr<Secret>(new Secret(value));
    }
};
```

### 4. Inherited Classes Requiring Base Constructor Conversion

```cpp
class Base {
public:
    virtual ~Base() = default;
};

class Derived : public Base {
public:
    Derived(int x) {}
};

// This works:
std::unique_ptr<Base> base = std::make_unique<Derived>(42);

// But if you need the deleter to be specific:
struct BaseDeleter {
    void operator()(Base* ptr) const { delete ptr; }
};

// ERROR: Can't specify custom deleter with make_unique
auto base2 = std::make_unique<Derived, BaseDeleter>(42);  // Won't compile

// Must use new:
auto base2 = std::unique_ptr<Base, BaseDeleter>(
    new Derived(42),
    BaseDeleter{}
);
```

### 5. Array Specialization Not Available Until C++20

```cpp
// C++14 and C++17: NOT AVAILABLE
auto arr = std::make_unique<int[]>(100);  // Compiler error

// Workaround for C++14/C++17:
std::unique_ptr<int[]> arr(new int[100]);

// C++20 and later: Available
auto arr = std::make_unique<int[]>(100);  // Works!
```


## Reassigning with `reset()`

The `reset()` method allows you to reassign a `unique_ptr` to a new resource. When you reassign, the old resource is automatically deleted via the deleter, then the new resource is stored.

### Basic `reset()` Usage

```cpp
#include <memory>
#include <iostream>

class Animal {
public:
    Animal(const std::string& name) : name_(name) {
        std::cout << "Animal " << name_ << " created\n";
    }
    ~Animal() {
        std::cout << "Animal " << name_ << " destroyed\n";
    }
private:
    std::string name_;
};

int main() {
    auto animal = std::make_unique<Animal>("Dog");
    
    // Reset to a new resource
    // First, the Dog is destroyed
    // Then, the new Cat is stored
    animal = std::make_unique<Animal>("Cat");
    
    // Reset to nullptr (releases the resource without assigning new one)
    animal.reset();
    // Cat is destroyed
    
    // animal is now nullptr
    if (!animal) {
        std::cout << "animal is now null\n";
    }
    
    return 0;
}

// Output:
// Animal Dog created
// Animal Dog destroyed
// Animal Cat created
// Animal Cat destroyed
// animal is now null
```

### `reset()` with a Raw Pointer

```cpp
#include <memory>
#include <iostream>

class Resource {
public:
    Resource(int id) : id_(id) {
        std::cout << "Resource " << id_ << " acquired\n";
    }
    ~Resource() {
        std::cout << "Resource " << id_ << " released\n";
    }
private:
    int id_;
};

int main() {
    std::unique_ptr<Resource> resource = std::make_unique<Resource>(1);
    
    std::cout << "\nRessigning with reset()...\n";
    // Reset with a new raw pointer
    // Old resource (1) is destroyed first
    resource.reset(new Resource(2));
    
    std::cout << "\nCalling reset() with no arguments...\n";
    // Reset with nullptr (default argument)
    resource.reset();
    
    std::cout << "\nEnd of main\n";
    return 0;
}

// Output:
// Resource 1 acquired
// 
// Reassigning with reset()...
// Resource 1 released
// Resource 2 acquired
//
// Calling reset() with no arguments...
// Resource 2 released
//
// End of main
```

### `reset()` with Custom Deleter

```cpp
#include <memory>
#include <cstdio>
#include <iostream>

struct FileDeleter {
    void operator()(FILE* file) const {
        if (file) {
            std::cout << "Closing file with custom deleter\n";
            std::fclose(file);
        }
    }
};

int main() {
    std::unique_ptr<FILE, FileDeleter> file(
        std::fopen("data1.txt", "r")
    );
    
    if (file) {
        std::cout << "Opened data1.txt\n";
    }
    
    // Reset to a different file
    // data1.txt is closed with the custom deleter
    // data2.txt is opened
    file.reset(std::fopen("data2.txt", "r"));
    
    if (file) {
        std::cout << "Opened data2.txt\n";
    }
    
    // Close the file explicitly
    file.reset();
    
    return 0;
}

// Output:
// Opened data1.txt
// Closing file with custom deleter
// Opened data2.txt
// Closing file with custom deleter
```

### `reset()` in Practice: Resource Replacement

```cpp
#include <memory>
#include <iostream>
#include <vector>

class DatabaseConnection {
public:
    DatabaseConnection(const std::string& server) : server_(server) {
        std::cout << "Connecting to " << server_ << "\n";
    }
    ~DatabaseConnection() {
        std::cout << "Disconnecting from " << server_ << "\n";
    }
    void query() const {
        std::cout << "Executing query on " << server_ << "\n";
    }
private:
    std::string server_;
};

int main() {
    std::unique_ptr<DatabaseConnection> db = 
        std::make_unique<DatabaseConnection>("server1.example.com");
    
    db->query();
    
    std::cout << "\nReconnecting to different server...\n";
    // Old connection is closed, new one is opened
    db.reset(new DatabaseConnection("server2.example.com"));
    
    db->query();
    
    std::cout << "\nCalling reset() to close connection...\n";
    db.reset();  // Closes the connection
    
    // Try to query after reset
    if (db) {
        db->query();
    } else {
        std::cout << "No active connection\n";
    }
    
    return 0;
}

// Output:
// Connecting to server1.example.com
// Executing query on server1.example.com
//
// Reconnecting to different server...
// Disconnecting from server1.example.com
// Connecting to server2.example.com
// Executing query on server2.example.com
//
// Calling reset() to close connection...
// Disconnecting from server2.example.com
// No active connection
```

### Key Points About `reset()`

- **Deletes old resource first**: When you reassign, the old resource is deleted via the deleter before the new one is stored
- **Safe with nullptr**: Calling `reset()` without arguments (or `reset(nullptr)`) safely releases the resource
- **Works with custom deleters**: The deleter is applied when the old resource is destroyed
- **Useful for resource replacement**: Allows you to cleanly switch from one resource to another
- **Enables cleanup without destruction**: You can explicitly release a resource before the `unique_ptr` goes out of scope

---

### Best Practices

1. **Use `std::make_unique` by default** when possible
2. **Use `new` with `std::unique_ptr` when**:
   - You need a custom deleter
   - Wrapping a pre-existing pointer
   - Working with C APIs
   - Need to call private constructors (through friend mechanisms)
   - Supporting C++14/C++17 with array types
3. **Never mix approaches** in the same codebase without clear reasoning

---

## Summary

| Feature | Details |
|---------|---------|
| **Ownership** | Exclusive, single owner |
| **Copyable** | No (copy constructor/assignment deleted) |
| **Movable** | Yes (transfer ownership) |
| **Overhead** | Zero - just a pointer wrapper |
| **Default Deleter** | `delete` (or `delete[]` for arrays) |
| **Custom Deleter** | Supported via template parameter |
| **Factory Function** | `std::make_unique<T>` (C++14) |
| **Array Support** | `unique_ptr<T[]>` or `make_unique<T[]>` (C++20) |

`std::unique_ptr` is the best choice for exclusive ownership of dynamically allocated objects in modern C++.