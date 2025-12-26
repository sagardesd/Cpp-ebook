# C++11 Advanced Constructor Features

A comprehensive guide to modern constructor features introduced in C++11.

---

## Table of Contents

1. [Delegating Constructors](#delegating-constructors)
2. [Defaulted Constructors](#defaulted-constructors)
3. [Deleted Constructors](#deleted-constructors)
4. [Non-static Data Member Initializers](#non-static-data-member-initializers)
5. [Inheriting Constructors](#inheriting-constructors)

---

## Delegating Constructors

### Why Needed?

Before C++11, multiple constructors with different parameters often duplicated initialization logic, leading to code repetition and maintenance issues.

### How It's Beneficial

Delegating constructors allow one constructor to call another constructor in the same class, reducing code duplication and centralizing initialization logic.

### Example

```cpp
class Rectangle {
private:
    int width;
    int height;
    
public:
    // Main constructor with initialization logic
    Rectangle(int w, int h) : width(w), height(h) {
        std::cout << "Creating rectangle: " << width << "x" << height << "\n";
    }
    
    // Delegating constructor - calls the main constructor
    Rectangle() : Rectangle(10, 10) {
        // Delegates to Rectangle(int, int)
    }
    
    // Another delegating constructor
    Rectangle(int size) : Rectangle(size, size) {
        // Creates a square by delegating
    }
};

// Usage
Rectangle r1;           // Calls Rectangle() -> Rectangle(10, 10)
Rectangle r2(5);        // Calls Rectangle(int) -> Rectangle(5, 5)
Rectangle r3(8, 12);    // Calls Rectangle(int, int) directly
```

**Before C++11 (Code Duplication):**
```cpp
class Rectangle {
    int width, height;
public:
    Rectangle(int w, int h) : width(w), height(h) {
        std::cout << "Creating rectangle\n";  // Duplicated
    }
    
    Rectangle() : width(10), height(10) {
        std::cout << "Creating rectangle\n";  // Duplicated
    }
    
    Rectangle(int size) : width(size), height(size) {
        std::cout << "Creating rectangle\n";  // Duplicated
    }
};
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Defaulted Constructors

### Why Needed?

Sometimes you want the compiler-generated default constructor even when you've defined other constructors. Before C++11, you had to write an empty constructor body if you have declared a parameterized constructor, which is unnecessary work.

### How It's Beneficial

Using `= default` explicitly requests the compiler to generate the default implementation, making code clearer and potentially more efficient.

### Example

```cpp
class Point {
private:
    int x, y;
    
public:
    // Explicitly request compiler-generated default constructor
    Point() = default;
    
    // Custom constructor
    Point(int xVal, int yVal) : x(xVal), y(yVal) {}
    
    // Explicitly defaulted copy constructor
    Point(const Point&) = default;
    
    // Explicitly defaulted copy assignment
    Point& operator=(const Point&) = default;
};

// Usage
Point p1;              // Default constructor (x and y uninitialized)
Point p2(5, 10);       // Custom constructor
Point p3 = p2;         // Copy constructor
```

**Why it matters:**
```cpp
class Data {
    int value;
public:
    Data(int v) : value(v) {}
    // Without = default, no default constructor exists
    // Data d;  // ERROR: no default constructor
};

class BetterData {
    int value;
public:
    BetterData() = default;  // Now we have both!
    BetterData(int v) : value(v) {}
};

BetterData d1;        // OK: uses defaulted constructor
BetterData d2(42);    // OK: uses custom constructor
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Deleted Constructors

### Why Needed?

Sometimes you want to prevent certain operations (like copying) or specific implicit conversions. Before C++11, you had to declare constructors as private without implementation.

### What `= delete` Means

Using `= delete` means the particular constructor is not available and is deleted. The compiler will generate an error if anyone attempts to use it.

### How It's Beneficial

Using `= delete` explicitly states intent, provides better error messages, and prevents unwanted operations at compile time.

### Example

```cpp
class UniqueResource {
private:
    int* data;
    
public:
    UniqueResource(int value) : data(new int(value)) {}
    
    // Delete copy constructor - prevent copying
    UniqueResource(const UniqueResource&) = delete;
    
    // Delete copy assignment - prevent assignment
    UniqueResource& operator=(const UniqueResource&) = delete;
    
    // Move operations are still allowed
    UniqueResource(UniqueResource&& other) noexcept : data(other.data) {
        other.data = nullptr;
    }
    
    ~UniqueResource() { delete data; }
};

// Usage
UniqueResource r1(42);
// UniqueResource r2 = r1;       // ERROR: copy constructor deleted
// UniqueResource r3(r1);        // ERROR: copy constructor deleted
UniqueResource r4 = std::move(r1); // OK: move constructor
```

**Preventing Implicit Conversions:**
```cpp
class SafeInt {
    int value;
public:
    SafeInt(int v) : value(v) {}
    
    // Prevent construction from double
    SafeInt(double) = delete;
};

SafeInt s1(42);        // OK
// SafeInt s2(3.14);   // ERROR: constructor deleted
// SafeInt s3 = 2.5;   // ERROR: constructor deleted
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Non-static Data Member Initializers

### Why Needed?

Before C++11, you had to initialize member variables in the constructor initializer list or constructor body, leading to duplication across multiple constructors.

### How It's Beneficial

You can provide default values directly in the class definition, reducing code duplication and ensuring members always have a valid initial value.

### Example

```cpp
class Configuration {
private:
    // Direct member initialization
    int maxConnections = 100;
    double timeout = 30.0;
    bool useSSL = true;
    std::string serverName = "localhost";
    
public:
    // Default constructor uses the member initializers
    Configuration() = default;
    
    // This constructor overrides only specific values
    Configuration(int connections) : maxConnections(connections) {
        // timeout, useSSL, serverName use their default values
    }
    
    // This overrides multiple values
    Configuration(int connections, double time) 
        : maxConnections(connections), timeout(time) {
        // useSSL and serverName use their default values
    }
    
    void display() const {
        std::cout << "Max Connections: " << maxConnections << "\n"
                  << "Timeout: " << timeout << "\n"
                  << "Use SSL: " << useSSL << "\n"
                  << "Server: " << serverName << "\n";
    }
};

// Usage
Configuration c1;           // All defaults: 100, 30.0, true, "localhost"
Configuration c2(200);      // 200, 30.0, true, "localhost"
Configuration c3(150, 60.0); // 150, 60.0, true, "localhost"
```

**Before C++11 (Code Duplication):**
```cpp
class OldConfiguration {
    int maxConnections;
    double timeout;
    bool useSSL;
    std::string serverName;
    
public:
    OldConfiguration() 
        : maxConnections(100), timeout(30.0), 
          useSSL(true), serverName("localhost") {}
    
    OldConfiguration(int connections) 
        : maxConnections(connections), timeout(30.0),  // Duplicated!
          useSSL(true), serverName("localhost") {}      // Duplicated!
    
    OldConfiguration(int connections, double time) 
        : maxConnections(connections), timeout(time), 
          useSSL(true), serverName("localhost") {}      // Duplicated!
};
```

**Combined with Delegating Constructors:**
```cpp
class SmartConfig {
    int value = 42;           // Default value
    std::string name = "default";
    
public:
    SmartConfig() = default;  // Uses member initializers
    
    SmartConfig(int v) : SmartConfig() {
        value = v;  // Override just one value
    }
};
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Inheriting Constructors

**Note:** This topic has been covered in detail in previous chapters on inheritance and derived classes.

### Brief Overview

C++11 allows derived classes to inherit base class constructors using the `using` declaration:

```cpp
class Base {
public:
    Base(int x) { }
    Base(int x, double y) { }
};

class Derived : public Base {
public:
    // Inherit all Base constructors
    using Base::Base;
    
    // Can still add new constructors
    Derived(std::string s) : Base(0) { }
};

// Usage
Derived d1(42);          // Uses inherited Base(int)
Derived d2(10, 3.14);    // Uses inherited Base(int, double)
Derived d3("hello");     // Uses Derived(std::string)
```

For comprehensive coverage of inheriting constructors, refer to the inheritance chapters.

[↑ Back to Table of Contents](#table-of-contents)

---

## Summary

C++11 constructor features provide powerful tools for writing cleaner, safer, and more maintainable code:

- **Delegating Constructors**: Reduce code duplication by reusing constructor logic
- **Defaulted Constructors**: Explicitly request compiler-generated implementations
- **Deleted Constructors**: Prevent unwanted operations and conversions
- **Explicit Constructors**: Avoid implicit conversions and potential bugs
- **Member Initializers**: Provide default values directly in class definitions
- **Inheriting Constructors**: Simplify derived class constructor declarations

These features work together to make C++ code more expressive and less error-prone.