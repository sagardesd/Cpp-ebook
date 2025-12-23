# CRTP (Curiously Recurring Template Pattern) in C++

---

## The One-Way Knowledge of Inheritance

Standard inheritance creates a hierarchy where knowledge only flows downward:

**Derived Knows Base:**
When you derive a class, it inherits all members (functions and variables) from its parent. It can see and use `public` and `protected` members of the Base class directly.

```cpp
class Base {
protected:
    int value;
public:
    void baseFunction() { }
};

class Derived : public Base {
public:
    void derivedFunction() {
        value = 10;          // Can access Base's protected member
        baseFunction();      // Can call Base's function
    }
};
```

**Base Has No Knowledge of Derived:**
The Base class is defined independently. It has absolutely no idea which classes will inherit from it later.

```cpp
class Base {
public:
    void callDerived() {
        // ERROR: Base doesn't know about Derived
        // derivedFunction();  // Won't compile!
    }
};

class Derived : public Base {
public:
    void derivedFunction() {
        std::cout << "Derived function called\n";
    }
};
```

### The "Access" Problem

If you have a `Base` object or pointer, you cannot access functions that only exist in `Derived`:

```cpp
int main() {
    Derived d;
    Base* ptr = &d;
    
    // ptr->derivedFunction();  // ERROR: Base doesn't know this function
}
```

In standard OOP, to allow the Base class to "call" a derived function, you must use **Virtual Functions**:

```cpp
class Base {
public:
    virtual void process() = 0;  // Pure virtual
};

class Derived : public Base {
public:
    void process() override {
        std::cout << "Processing in Derived\n";
    }
};

int main() {
    Base* ptr = new Derived();
    ptr->process();  // Works! Calls Derived::process()
    delete ptr;
}
```

This function dispatch at runtime is **dynamic polymorphism**.
Its really neat feature and very usefull.
But in performance critical system it can be an overhead as well.
How ?

### The Cost of Virtual Functions

However, virtual functions come with a "hidden" cost:

1. **V-Table Lookups:** The program must look up the correct function at runtime through a virtual table (vtable).
2. **Inlining Failure:** Compilers often cannot optimize or "inline" virtual calls, making them slower.
3. **Memory Overhead:** Each object with virtual functions carries a hidden pointer to the vtable (typically 8 bytes on 64-bit systems).

```cpp
class Base {
public:
    virtual void foo() { }
};

// Behind the scenes, compiler generates something like:
// - Global vtable for Base
// - Each Base object contains a hidden vptr (pointer to vtable)
// - Function calls: obj->vptr->vtable[index]()
```

**Performance Impact:**
- Direct call: ~1-2 CPU cycles
- Virtual call: ~5-10 CPU cycles (vtable lookup + indirect jump)
- Lost inlining opportunities mean further optimizations are blocked


Let's revisit what is our goal here:
- **Derived class objects can invoke base class functions**
- **Base class functions need to call derived class methods** (without virtuals) (How ?)

**Question:** How can we make the base class function (invoked using derive class object) able to call derived class methods ?

**Answer:** The base class function needs to have **knowledge of the exact derived class type at compile time**.

If we can achieve that, we get **static polymorphism** – polymorphic behavior resolved at compile time with zero runtime overhead!

### What about Templates ? Can we use it for to solve this problem ?

What if we create a base class that **takes the derived class type as its template parameter**?

This way the Base class has idea about the exact type of the Derived class at compile time and our problem solved.

This is exactly what **CRTP (Curiously Recurring Template Pattern)** is!

---

## What is CRTP?

CRTP is a pattern that achieves **static polymorphism** by passing the type of a derived class to a base class. In the Curiously Recurring Template Pattern, a class (let's call it `Derived`) inherits from a class template (let's call it `Base`) that has been specialized specifically with `Derived` as its template argument. This enables the base class to have knowledge of the derived class type at compile time, eliminating the runtime overhead of virtual function calls while maintaining polymorphic behavior.

```cpp
// Step 1: Define a template base class that takes a type T
template<typename T>
class Base {
    // Base is a template - T will be the derived class type
};

// Step 2: Derived class inherits from Base, passing its own type as T
class Derived : public Base<Derived> {
    //                      ^^^^^^^^
    // Derived passes itself as the template argument!
};
```

This creates a **curious recursive relationship** where:
1. `Derived` inherits from `Base<Derived>`
2. Inside `Base`, the template parameter `T` is actually `Derived`
3. `Base<Derived>` knows the exact type `Derived` at compile time
4. `Base<Derived>` can call `Derived`'s methods using `static_cast<T*>(this)`

### The Magic is Compile-Time Downcasting

```cpp
// Base class template - T will be the derived class type
template<typename T>
class Base {
public:
    void interface() {
        // Cast 'this' to T* (which is Derived*) at compile time
        static_cast<T*>(this)->implementation();
    }
};

// Derived inherits from Base<Derived>
// So inside Base, T is Derived
class Derived : public Base<Derived> {
public:
    void implementation() {
        std::cout << "Derived implementation called!\n";
    }
};

int main() {
    Derived d;
    d.interface();  // Base::interface() calls Derived::implementation()
}
```

**Output:**
```
Derived implementation called!
```

**What happened?**
1. We call `d.interface()` which is defined in `Base<Derived>`
2. Inside `Base`, the template parameter `T` is `Derived`
3. We do `static_cast<T*>(this)` which becomes `static_cast<Derived*>(this)` 
4. This cast is resolved at **compile time** (zero runtime cost!)
5. We call `implementation()` on the derived class directly
6. Compiler can **inline** the entire call chain for maximum performance

---

## Basic CRTP Implementation Example: Template Method Pattern

The Template Method Pattern is a behavioral design pattern where the base class defines the skeleton of an algorithm, and derived classes provide specific implementations for certain steps. While this pattern can be achieved using dynamic polymorphism with virtual functions, CRTP offers a compile-time alternative with zero runtime overhead.
Lets look at both.

### Comparison: Virtual Functions vs CRTP

Let's first look at the traditional approach using dynamic polymorphism, then explore how CRTP implements the same pattern.

### Approach 1: Virtual Functions (Dynamic Polymorphism)

```cpp
#include <iostream>
#include <memory>
#include <vector>

// Abstract base class with virtual functions
class DataProcessor {
public:
    // The template method - defines the algorithm structure
    void process() {
        read();
        processImpl();  // Virtual dispatch
        write();
    }
    
    virtual ~DataProcessor() = default;
    
private:
    void read() {
        std::cout << "[DataProcessor] Reading data...\n";
    }
    
    void write() {
        std::cout << "[DataProcessor] Writing results...\n";
    }
    
protected:
    virtual void processImpl() = 0;  // Pure virtual - customization point
};

// Concrete implementation for CSV data
class CSVProcessor : public DataProcessor {
protected:
    void processImpl() override {
        std::cout << "[CSVProcessor] Processing CSV format\n";
        std::cout << "  - Parsing comma-separated values\n";
    }
};

// Concrete implementation for JSON data
class JSONProcessor : public DataProcessor {
protected:
    void processImpl() override {
        std::cout << "[JSONProcessor] Processing JSON format\n";
        std::cout << "  - Parsing JSON structure\n";
    }
};

int main() {
    // Can store different types via base class pointers
    std::vector<std::unique_ptr<DataProcessor>> processors;
    processors.push_back(std::make_unique<CSVProcessor>());
    processors.push_back(std::make_unique<JSONProcessor>());
    
    for (auto& proc : processors) {
        proc->process();  // Virtual dispatch at runtime
        std::cout << "\n";
    }
}
```

**Output:**
```
[DataProcessor] Reading data...
[CSVProcessor] Processing CSV format
  - Parsing comma-separated values
[DataProcessor] Writing results...

[DataProcessor] Reading data...
[JSONProcessor] Processing JSON format
  - Parsing JSON structure
[DataProcessor] Writing results...
```

**Characteristics:**
- Uses virtual function for runtime dispatch
- Can store heterogeneous types in containers
- Runtime overhead from vtable lookup
- More flexible for dynamic scenarios

### Approach 2: CRTP (Static Polymorphism)

Now let's implement the same pattern using CRTP. The key difference is that the base class knows the derived type at compile time:

```cpp
#include <iostream>

// CRTP Base class that defines the template method pattern
template<typename Derived>
class DataProcessor {
public:
    // The template method - defines the algorithm structure
    void process() {
        read();
        static_cast<Derived*>(this)->processImpl();  // Compile-time dispatch
        write();
    }
    
private:
    void read() {
        std::cout << "[DataProcessor] Reading data...\n";
    }
    
    void write() {
        std::cout << "[DataProcessor] Writing results...\n";
    }
};

// Concrete implementation for CSV data
class CSVProcessor : public DataProcessor<CSVProcessor> {
public:
    void processImpl() {
        std::cout << "[CSVProcessor] Processing CSV format\n";
        std::cout << "  - Parsing comma-separated values\n";
    }
};

// Concrete implementation for JSON data
class JSONProcessor : public DataProcessor<JSONProcessor> {
public:
    void processImpl() {
        std::cout << "[JSONProcessor] Processing JSON format\n";
        std::cout << "  - Parsing JSON structure\n";
    }
};

// Concrete implementation for XML data
class XMLProcessor : public DataProcessor<XMLProcessor> {
public:
    void processImpl() {
        std::cout << "[XMLProcessor] Processing XML format\n";
        std::cout << "  - Parsing XML tags\n";
    }
};

int main() {
    CSVProcessor csv;
    csv.process();
    
    std::cout << "\n";
    
    JSONProcessor json;
    json.process();
    
    std::cout << "\n";
    
    XMLProcessor xml;
    xml.process();
}
```

**Output:**
```
[DataProcessor] Reading data...
[CSVProcessor] Processing CSV format
  - Parsing comma-separated values
[DataProcessor] Writing results...

[DataProcessor] Reading data...
[JSONProcessor] Processing JSON format
  - Parsing JSON structure
[DataProcessor] Writing results...

[DataProcessor] Reading data...
[XMLProcessor] Processing XML format
  - Parsing XML tags
[DataProcessor] Writing results...
```

**Characteristics:**
- No virtual functions required
- Compile-time type resolution via `static_cast`
- Cannot store different types in a single container (trade-off)
- Zero runtime overhead for polymorphic dispatch
- Compiler can fully inline the call chain

### Key Differences

| Aspect | Virtual Functions | CRTP |
|--------|------------------|------|
| **Dispatch Type** | Runtime (dynamic) | Compile-time (static) |
| **Polymorphic Containers** | Yes | No |
| **Virtual Function Overhead** | Yes (~5-10 CPU cycles) | None |
| **Inlining** | Limited | Full inlining possible |
| **Type Flexibility** | High (runtime types) | Low (compile-time types) |
| **Memory Overhead** | vtable pointer per object | None |

### When to Use Each

**Use Virtual Functions when:**
- You need to store different derived types in containers
- Types are determined at runtime
- Flexibility is more important than peak performance
- You're building extensible plugin systems

**Use CRTP when:**
- You know all types at compile time
- Performance-critical code needs zero overhead
- You don't need heterogeneous containers
- You want compiler optimization benefits

---

## CRTP Limitations and Hybrid Approach

### The Container Problem with CRTP

One of the most significant limitations of CRTP is its inability to store different derived types in the same container. This stems from a fundamental constraint: each specialization of a CRTP base template is a completely different type.

```cpp
template<typename T>
class Shape {
public:
    void draw() {
        static_cast<T*>(this)->drawImpl();
    }
};

class Circle : public Shape<Circle> {
public:
    void drawImpl() { std::cout << "Drawing Circle\n"; }
};

class Square : public Shape<Square> {
public:
    void drawImpl() { std::cout << "Drawing Square\n"; }
};

int main() {
    std::vector<Shape<Circle>> shapes;  // Can only store Circles
    
    // This doesn't work:
    // std::vector<Shape<??>> shapes;  // What type goes here?
    // shapes.push_back(Circle());  // Type mismatch
    // shapes.push_back(Square());  // Different type!
}
```

The problem is that `Shape<Circle>` and `Shape<Square>` are entirely different types. There's no common base class between them, making it impossible to store both in a single container. CRTP achieves zero-overhead abstraction at the cost of losing the flexibility to work with different derived types polymorphically through a single pointer or reference.

### Dynamic Polymorphism: The Container Advantage

Dynamic polymorphism with virtual functions solves this problem elegantly:

```cpp
class Shape {  // Non-template base
public:
    virtual void draw() = 0;
    virtual ~Shape() = default;
};

class Circle : public Shape {
public:
    void draw() override { std::cout << "Drawing Circle\n"; }
};

class Square : public Shape {
public:
    void draw() override { std::cout << "Drawing Square\n"; }
};

int main() {
    // Works perfectly! Can store any Shape derived class
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Circle>());
    shapes.push_back(std::make_unique<Square>());
    shapes.push_back(std::make_unique<Circle>());
    
    for (auto& shape : shapes) {
        shape->draw();  // Calls correct derived implementation
    }
}
```

**Output:**
```
Drawing Circle
Drawing Square
Drawing Circle
```

This is the classic trade-off: dynamic polymorphism gives flexibility and runtime behavior variation at the cost of vtable lookups and lost inlining opportunities.

### The Hybrid Approach: Best of Both Worlds

The hybrid approach combines CRTP with dynamic polymorphism to gain the benefits of both. The idea is to create a non-template abstract base class that uses virtual functions, while individual implementations use CRTP internally for performance-critical operations.

```cpp
#include <iostream>
#include <vector>
#include <memory>

// Abstract base class using virtual functions
class Shape {
public:
    virtual void draw() = 0;
    virtual ~Shape() = default;
};

// CRTP base for common functionality
template<typename Derived>
class ShapeImpl : public Shape {
protected:
    // CRTP allows static polymorphism for internal operations
    void internalSetup() {
        static_cast<Derived*>(this)->setup();
    }
    
    void internalCleanup() {
        static_cast<Derived*>(this)->cleanup();
    }
};

class Circle : public ShapeImpl<Circle> {
private:
    double radius_;
    
    void setup() {
        std::cout << "  [Circle] Initializing radius\n";
    }
    
    void cleanup() {
        std::cout << "  [Circle] Cleaning up radius\n";
    }
    
    friend class ShapeImpl<Circle>;
    
public:
    Circle(double r) : radius_(r) { }
    
    void draw() override {
        std::cout << "Drawing Circle with radius " << radius_ << "\n";
        internalSetup();
        internalCleanup();
    }
};

class Square : public ShapeImpl<Square> {
private:
    double side_;
    
    void setup() {
        std::cout << "  [Square] Initializing side\n";
    }
    
    void cleanup() {
        std::cout << "  [Square] Cleaning up side\n";
    }
    
    friend class ShapeImpl<Square>;
    
public:
    Square(double s) : side_(s) { }
    
    void draw() override {
        std::cout << "Drawing Square with side " << side_ << "\n";
        internalSetup();
        internalCleanup();
    }
};

int main() {
    // Can store different types in container
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Circle>(5.0));
    shapes.push_back(std::make_unique<Square>(4.0));
    shapes.push_back(std::make_unique<Circle>(3.5));
    
    // Virtual dispatch for external interface
    for (auto& shape : shapes) {
        shape->draw();
    }
}
```

**Output:**
```
Drawing Circle with radius 5
  [Circle] Initializing radius
  [Circle] Cleaning up radius
Drawing Square with side 4
  [Square] Initializing side
  [Square] Cleaning up side
Drawing Circle with radius 3.5
  [Circle] Initializing radius
  [Circle] Cleaning up radius
```

### Hybrid Approach Benefits

The hybrid strategy provides several advantages:

**Container Flexibility:** You can store different derived types in the same container through the abstract base class pointer, solving CRTP's fundamental limitation.

**Internal Performance:** Within each derived class, CRTP enables zero-overhead abstraction for internal operations that don't need polymorphic behavior. The compiler can inline and optimize these calls completely.

**Selective Virtuality:** Only the interface methods that truly need runtime polymorphism are virtual. Internal helper methods can use CRTP for maximum performance.

**Type Safety:** The CRTP base ensures compile-time type checking for the specific derived class operations, catching errors at compile time rather than runtime.

### When to Use Each Approach

Use **pure CRTP** when you have a known set of types that never need to be stored heterogeneously and performance is critical.

Use **pure virtual functions** when you need maximum flexibility with runtime-determined types and polymorphic behavior is the primary requirement.

Use the **hybrid approach** when you need both container flexibility AND performance-critical internal operations, or when your system has a stable interface with performance-sensitive implementations.

---

## Summary

This tutorial covers the fundamentals of CRTP and how it enables static polymorphism. 

In a separate dedicated chapter, we explore comprehensive performance benchmarks comparing CRTP against traditional virtual functions.
Also we will discuss some common use cases of CRTP.


### Real-World Projects Using CRTP

Several high-performance and widely-used open-source projects leverage CRTP extensively:

**ClickHouse (Column-oriented Database)**
- One of the fastest open-source analytical databases
- Uses CRTP extensively for query optimization and column processing
- Achieves extreme performance through compile-time specialization
- CRTP enables efficient data type handling without virtual function overhead

**Eigen (Linear Algebra Library)**
- Industry-standard C++ library for matrices and vectors
- Uses expression templates with CRTP for lazy evaluation
- Avoids temporary object creation in complex mathematical expressions
- Powers machine learning frameworks like TensorFlow

**High-Frequency Trading (HFT) Systems**
- Critical latency-sensitive applications in financial markets
- CRTP is a core pattern in order routing and risk management systems
- Eliminates vtable overhead for microsecond-critical operations
- Used in systems that process millions of orders per second

**Boost C++ Libraries**
- Various Boost libraries use CRTP for type-safe, zero-overhead abstractions
- Examples include Boost.Asio (networking) and Boost.Range

**Apache Arrow**
- Data processing framework used in big data ecosystems
- Uses CRTP for efficient memory layout and data type handling

These projects demonstrate that CRTP is not just an academic pattern but a proven technique used in the most demanding, performance-critical applications in the industry.
