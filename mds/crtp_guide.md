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

```
                     Base<T>
                  (Template Class)
                        △
                        │ T = Derived
                        │
                     Derived
              (Inherits from Base<Derived>)
                        │
                        ├─→ static_cast<Derived*>(this)
                        │
                   Compile-time
                  Polymorphism
                  (Zero overhead)
```

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

**Virtual Functions - Runtime Dispatch:**
```
    Base class (with virtual methods)
           △
           │ Virtual call
           │ (Runtime decision)
           │
    ┌──────┴──────┐
    │             │
Derived1      Derived2
    │             │
    └──────┬──────┘
           │ vtable lookup at runtime
           │ Indirect jump (5-10 cycles)
           ▼
     Actual Method
```

**CRTP - Compile-time Dispatch:**
```
    Base<T> (Template class)
           △
           │ T = Concrete type
           │ (Known at compile time)
           │
    ┌──────┴──────┐
    │             │
  Base<      Base<
 Derived1>  Derived2>
    │             │
    └──────┬──────┘
           │ static_cast<T*>(this)
           │ Resolved at compile time
           │ Full inlining possible
           ▼
     Actual Method
   (Zero overhead)
```

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

## Performance Comparison: CRTP vs Virtual Functions

One of the key advantages of CRTP is its superior performance characteristics compared to virtual functions. Let's examine this with concrete benchmarks.

### Benchmark Setup

The following benchmark compares the performance of CRTP against traditional virtual functions. The test scenario involves:

- **3,000 shapes** (1,000 circles, 1,000 rectangles, 1,000 triangles)
- **10,000 iterations** performing calculations
- **60,000,000 total function calls** (`area()` and `perimeter()` on each shape)
- **5 runs** for statistical accuracy

The code implements both approaches identically in terms of logic, with the only difference being the polymorphism mechanism: virtual functions vs. CRTP.

### Benchmark Code

```cpp
#include <iostream>
#include <chrono>
#include <vector>
#include <memory>
#include <cmath>
#include <iomanip>

// ============================================================================
// APPROACH 1: Virtual Functions (Polymorphism with Runtime Dispatch)
// ============================================================================

class ShapeVirtual {
public:
    virtual ~ShapeVirtual() = default;
    virtual double area() const = 0;
    virtual double perimeter() const = 0;
};

class CircleVirtual : public ShapeVirtual {
private:
    double radius_;
public:
    CircleVirtual(double r) : radius_(r) {}
    double area() const override { 
        return 3.14159 * radius_ * radius_; 
    }
    double perimeter() const override { 
        return 2 * 3.14159 * radius_; 
    }
};

class RectangleVirtual : public ShapeVirtual {
private:
    double width_, height_;
public:
    RectangleVirtual(double w, double h) : width_(w), height_(h) {}
    double area() const override { 
        return width_ * height_; 
    }
    double perimeter() const override { 
        return 2 * (width_ + height_); 
    }
};

class TriangleVirtual : public ShapeVirtual {
private:
    double a_, b_, c_;
public:
    TriangleVirtual(double a, double b, double c) : a_(a), b_(b), c_(c) {}
    double area() const override {
        double s = (a_ + b_ + c_) / 2;
        return std::sqrt(s * (s - a_) * (s - b_) * (s - c_));
    }
    double perimeter() const override { 
        return a_ + b_ + c_; 
    }
};

// ============================================================================
// APPROACH 2: CRTP (Curiously Recurring Template Pattern)
// ============================================================================

template <typename Derived>
class ShapeCRTP {
public:
    double area() const {
        return static_cast<const Derived*>(this)->area_impl();
    }
    
    double perimeter() const {
        return static_cast<const Derived*>(this)->perimeter_impl();
    }
    
    ~ShapeCRTP() = default;
};

class CircleCRTP : public ShapeCRTP<CircleCRTP> {
private:
    double radius_;
public:
    CircleCRTP(double r) : radius_(r) {}
    
    double area_impl() const { 
        return 3.14159 * radius_ * radius_; 
    }
    
    double perimeter_impl() const { 
        return 2 * 3.14159 * radius_; 
    }
};

class RectangleCRTP : public ShapeCRTP<RectangleCRTP> {
private:
    double width_, height_;
public:
    RectangleCRTP(double w, double h) : width_(w), height_(h) {}
    
    double area_impl() const { 
        return width_ * height_; 
    }
    
    double perimeter_impl() const { 
        return 2 * (width_ + height_); 
    }
};

class TriangleCRTP : public ShapeCRTP<TriangleCRTP> {
private:
    double a_, b_, c_;
public:
    TriangleCRTP(double a, double b, double c) : a_(a), b_(b), c_(c) {}
    
    double area_impl() const {
        double s = (a_ + b_ + c_) / 2;
        return std::sqrt(s * (s - a_) * (s - b_) * (s - c_));
    }
    
    double perimeter_impl() const { 
        return a_ + b_ + c_; 
    }
};

// ============================================================================
// BENCHMARK FUNCTIONS
// ============================================================================

long long benchmarkVirtual() {
    std::vector<std::unique_ptr<ShapeVirtual>> shapes;
    
    // Create mixed shapes
    for (int i = 0; i < 1000; ++i) {
        shapes.push_back(std::make_unique<CircleVirtual>(5.0 + i*0.001));
        shapes.push_back(std::make_unique<RectangleVirtual>(4.0 + i*0.001, 6.0));
        shapes.push_back(std::make_unique<TriangleVirtual>(3.0, 4.0, 5.0 + i*0.001));
    }
    
    auto start = std::chrono::high_resolution_clock::now();
    
    volatile double totalArea = 0;
    volatile double totalPerimeter = 0;
    
    for (int iteration = 0; iteration < 10000; ++iteration) {
        double scale = 1.0 + iteration * 0.0001;
        for (const auto& shape : shapes) {
            totalArea += shape->area() * scale;
            totalPerimeter += shape->perimeter() * scale;
        }
    }
    
    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
}

long long benchmarkCRTP() {
    std::vector<CircleCRTP> circles; circles.reserve(1000);
    std::vector<RectangleCRTP> rects; rects.reserve(1000);
    std::vector<TriangleCRTP> tris; tris.reserve(1000);

    for (int i = 0; i < 1000; ++i) {
        circles.emplace_back(5.0 + i*0.001);
        rects.emplace_back(4.0 + i*0.001, 6.0);
        tris.emplace_back(3.0, 4.0, 5.0 + i*0.001);
    }

    auto start = std::chrono::high_resolution_clock::now();
    volatile double totalArea = 0, totalPerimeter = 0;

    for (int iteration = 0; iteration < 10000; ++iteration) {
        double scale = 1.0 + iteration * 0.0001;

        for (const auto& c : circles) {
            totalArea += c.area() * scale;
            totalPerimeter += c.perimeter() * scale;
        }
        for (const auto& r : rects) {
            totalArea += r.area() * scale;
            totalPerimeter += r.perimeter() * scale;
        }
        for (const auto& t : tris) {
            totalArea += t.area() * scale;
            totalPerimeter += t.perimeter() * scale;
        }
    }

    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
}

// ============================================================================
// MAIN - RUN BENCHMARKS
// ============================================================================

int main() {
    std::cout << "\n";
    std::cout << "╔════════════════════════════════════════════════════════╗\n";
    std::cout << "║     CRTP vs Virtual Functions - Performance Benchmark  ║\n";
    std::cout << "╚════════════════════════════════════════════════════════╝\n";
    std::cout << "\n";
    
    std::cout << "Scenario: 3000 shapes (1000 of each type), 10000 iterations\n";
    std::cout << "Each iteration calls area() and perimeter() on all shapes\n";
    std::cout << "Total function calls: 3000 × 2 × 10000 = 60,000,000 calls\n";
    std::cout << "\n";
    
    std::cout << "Running benchmarks 5 times each for accuracy...\n\n";
    
    long long virtualTotal = 0;
    long long crptTotal = 0;
    
    for (int run = 1; run <= 5; ++run) {
        long long vTime = benchmarkVirtual();
        long long cTime = benchmarkCRTP();
        
        virtualTotal += vTime;
        crptTotal += cTime;
        
        std::cout << "Run " << run << ": Virtual=" << std::setw(3) << vTime 
                  << "ms | CRTP=" << std::setw(3) << cTime << "ms\n";
    }
    
    double avgVirtual = virtualTotal / 5.0;
    double avgCRTP = crptTotal / 5.0;
    double improvement = ((avgVirtual - avgCRTP) / avgVirtual) * 100.0;
    double speedup = avgVirtual / avgCRTP;
    
    std::cout << "\n";
    std::cout << "╔════════════════════════════════════════════════════════╗\n";
    std::cout << "║                    RESULTS SUMMARY                     ║\n";
    std::cout << "╚════════════════════════════════════════════════════════╝\n";
    std::cout << "\n";
    
    std::cout << "Virtual Functions (Average):  " << std::fixed << std::setprecision(1) 
              << avgVirtual << " ms\n";
    std::cout << "CRTP (Average):               " << avgCRTP << " ms\n";
    std::cout << "\n";
    
    std::cout << "Performance Improvement:      " << improvement << "%\n";
    std::cout << "Speedup Factor:               " << std::setprecision(2) << speedup << "x faster\n";
    std::cout << "\n";
    
    std::cout << "═══════════════════════════════════════════════════════════\n\n";
    
    return 0;
}
```

### Benchmark Results

```
╔════════════════════════════════════════════════════════╗
║     CRTP vs Virtual Functions - Performance Benchmark  ║
╚════════════════════════════════════════════════════════╝

Scenario: 3000 shapes (1000 of each type), 10000 iterations
Each iteration calls area() and perimeter() on all shapes
Total function calls: 3000 × 2 × 10000 = 60,000,000 calls

Running benchmarks 5 times each for accuracy...

Run 1: Virtual= 55ms | CRTP= 29ms
Run 2: Virtual= 35ms | CRTP= 25ms
Run 3: Virtual= 34ms | CRTP= 25ms
Run 4: Virtual= 34ms | CRTP= 25ms
Run 5: Virtual= 35ms | CRTP= 25ms

╔════════════════════════════════════════════════════════╗
║                    RESULTS SUMMARY                     ║
╚════════════════════════════════════════════════════════╝

Virtual Functions (Average):  38.6 ms
CRTP (Average):               25.8 ms

Performance Improvement:      33.2%
Speedup Factor:               1.50x faster

═══════════════════════════════════════════════════════════
```

### Analysis

**Important Note:** Benchmark results are highly dependent on the compiler, optimization flags, CPU architecture, and runtime environment. This benchmark was compiled with:
```
g++ -O3 -std=c++17 benchmark.cpp -o benchmark
```

These results are provided to showcase that CRTP can offer significant performance advantages in certain scenarios, particularly with modern optimizers and aggressive optimization levels like `-O3`. Your actual results may vary significantly depending on your compiler version, optimization flags, CPU architecture, and runtime environment. It's always recommended to profile your own code with your specific toolchain and hardware.

The benchmark results clearly demonstrate CRTP's performance advantage in this scenario:

**33.2% Performance Improvement** over virtual functions on 60 million function calls. The CRTP approach achieves a **1.50x speedup**, completing the same workload in approximately two-thirds the time of the virtual function approach.

This significant performance gain stems from several factors:

1. **No vtable lookups:** CRTP eliminates runtime table lookups entirely, replacing them with compile-time type resolution.

2. **Full inlining:** The compiler can aggressively inline CRTP calls since the target function is known at compile time. Virtual function calls are typically not inlined due to the indirection involved.

3. **Better cache locality:** Without vtable pointers, objects have smaller memory footprints and better cache behavior.

4. **Optimizer friendly:** The compiler has complete visibility into the call chain and can apply more aggressive optimizations.

**Performance Comparison Visualization:**
```
    Virtual Functions  │  CRTP
    ─────────────────────────────
    38.6 ms (100%)     │  25.8 ms (67%)
    ██████████         │  ███████
    
    Slower ←────────────┼────────→ Faster
                       │
              33.2% improvement
              1.50x speedup
```

The results show that while the theoretical overhead per virtual call is 5-10 CPU cycles, in real-world scenarios with modern optimizers, the actual impact can be even more substantial when considering inlining opportunities and cache effects.

---

## CRTP Limitations

One of the most significant limitations of CRTP is its inability to store different derived types in the same container. This stems from a fundamental constraint: each specialization of a CRTP base template is a completely different type. For example, `Shape<Circle>` and `Shape<Square>` are entirely different types with no common base class, making it impossible to store both in a single container through polymorphic pointers or references.

Additionally, CRTP requires all types to be known at compile time, limiting its use in scenarios where types are determined dynamically at runtime, such as plugin systems or highly extensible architectures.

These limitations will be explored in detail in a separate chapter on CRTP limitations and hybrid approaches that combine CRTP with dynamic polymorphism for maximum flexibility and performance.

---

## Summary

This tutorial covers the fundamentals of CRTP and how it enables static polymorphism with concrete performance advantages.

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