# C++ Polymorphism

## Table of Contents

1. [What is Polymorphism?](#what-is-polymorphism)
2. [Polymorphism in C++ Programming](#polymorphism-in-c-programming)
3. [How Can We Achieve Polymorphism in C++?](#how-can-we-achieve-polymorphism-in-c)
4. [Static Polymorphism (Compile-Time Polymorphism)](#static-polymorphism-compile-time-polymorphism)
   - [Function Overloading](#function-overloading)
   - [Example: Static Polymorphism with Function Overloading](#example-static-polymorphism-with-function-overloading)
5. [Important: Function Overloading Cannot Be Achieved by Just Having Different Return Types](#important-function-overloading-cannot-be-achieved-by-just-having-different-return-types)
   - [What is a Function Signature?](#what-is-a-function-signature)
   - [Why Can't We Overload Based on Return Type Alone?](#why-cant-we-overload-based-on-return-type-alone)
   - [Const Overloading (Special Case for Member Functions)](#const-overloading-special-case-for-member-functions)
6. [When Static Polymorphism Is Not Enough](#when-static-polymorphism-is-not-enough)
7. [Dynamic Polymorphism (Runtime Polymorphism)](#dynamic-polymorphism-runtime-polymorphism)
   - [Function Overriding](#function-overriding)
   - [Virtual Functions](#virtual-functions)
   - [How Virtual Functions Work: The Mechanism](#how-virtual-functions-work-the-mechanism)
   - [How a Virtual Function Call Gets Resolved](#how-a-virtual-function-call-gets-resolved)
   - [The `override` Keyword (C++11)](#the-override-keyword-c11)
   - [The `final` Keyword (C++11)](#the-final-keyword-c11)

---

## What is Polymorphism?

Imagine the word **"play"** — it's the same word, but its meaning changes depending on the situation:

* When you say, **"Kids play in the park,"** it means they are having fun or playing games.
* When you say, **"Musicians play the guitar,"** it means they are performing music.
* When you say, **"Actors play a role,"** it means they are acting in a movie or play.

**Same word ("play") — different meanings depending on the context.**

That's what **polymorphism** means (having many forms):

> **"One thing (name or action) behaving differently based on the situation."**

## Polymorphism in C++ Programming

**Polymorphism** is the ability of a single method or function to behave differently depending on the situation. From a class and object perspective, it means:

> The same method can produce different behaviors depending on either the **type of object** it is called on, or the **type of data** it is given.

**Key Idea:** Client code can call a method on different kinds of objects or data, and the resulting behavior will differ — this is the essence of polymorphism.

[↑ Back to Table of Contents](#table-of-contents)

---

## How Can We Achieve Polymorphism in C++?

In C++, polymorphism can be achieved in two main ways:

1. **At compile time** → **Static Polymorphism**
2. **At runtime** → **Dynamic Polymorphism**

Let's first understand compile-time polymorphism.

[↑ Back to Table of Contents](#table-of-contents)

---

## Static Polymorphism (Compile-Time Polymorphism)

**Static polymorphism** is achieved when the behavior of a function is decided **at compile time**.

* The compiler determines which method to call based on the **data type** or **number of arguments** passed.
* This allows the same function name to work in multiple ways, depending on the inputs.

Common ways to achieve this are **function overloading**, **operator overloading**, and **templates** (will cover templates in a separate section).

### Function Overloading

**Function overloading** allows you to define multiple functions with the same name but with different parameter types or numbers of parameters.

The compiler automatically selects the appropriate function based on the arguments you pass.

#### Example: Static Polymorphism with Function Overloading

```cpp
#include <iostream>
#include <string>
using namespace std;

class Player {
public:
    void play(int minutes) {
        cout << "Kids are playing for " << minutes << " minutes.\n";
    }

    void play(const string& instrument) {
        cout << "Musician is playing the " << instrument << ".\n";
    }

    void play() {
        cout << "Actor is playing a role in a movie.\n";
    }
};

int main() {
    Player p;

    p.play();             // Actor
    p.play(30);           // Kids
    p.play("Guitar");     // Musician
}
```

#### Output:

```
Actor is playing a role in a movie.
Kids are playing for 30 minutes.
Musician is playing the Guitar.
```

#### Explanation:

* The same function name `play()` behaves differently depending on the arguments.
* The compiler decides which version to call — this is **static (compile-time) polymorphism**.

[↑ Back to Table of Contents](#table-of-contents)

---

## Important: Function Overloading Cannot Be Achieved by Just Having Different Return Types

You **cannot** overload functions based solely on their return type. The compiler uses the **function signature** to distinguish between overloaded functions, and the return type is **not** part of the function signature.

### What is a Function Signature?

A function signature consists of:
* The **function name**
* The **number of parameters**
* The **types of parameters**
* The **order of parameters**

**Note:** The return type is **NOT** included in the function signature.

### Why Can't We Overload Based on Return Type Alone?

When you call a function, the compiler needs to determine which version to execute based on how you're calling it. The compiler looks at:

* The function name
* The arguments you're passing

The compiler does **not** look at how you're using the return value to decide which function to call.

#### Example: Why This Won't Work

```cpp
class Calculator {
public:
    int compute(int a, int b) {
        return a + b;
    }

    double compute(int a, int b) {  // ❌ ERROR: Cannot overload
        return a + b + 0.5;
    }
};

int main() {
    Calculator calc;
    auto result = calc.compute(5, 3);  // Which function should be called?
}
```

**Problem:** When the compiler sees `calc.compute(5, 3)`, it looks at:
* Function name: `compute` ✓
* Arguments: `(int, int)` ✓

Both functions have the **exact same signature**: `compute(int, int)`

The compiler has **no way** to decide which function to call because:
* It doesn't know if you want an `int` or `double` result
* Function selection happens **before** the return value is considered
* Even if you write `int result = calc.compute(5, 3);`, the compiler resolves the function call **first**, then attempts the assignment

#### Symbol Perspective

In compiled code, functions are identified by **name mangling** (a technique where the compiler creates unique symbols for functions). The mangled name includes:

* Function name
* Parameter types
* (Sometimes) namespace/class name

For example, the compiler might create symbols like:
* `_ZN10Calculator7computeEii` → `Calculator::compute(int, int)`
* `_ZN10Calculator7computeEii` → `Calculator::compute(int, int)` returning double

**Both would have the same mangled symbol!** This creates a conflict.

#### Valid Overloading Examples

```cpp
class Calculator {
public:
    // ✓ Different number of parameters
    int compute(int a) {
        return a * 2;
    }

    int compute(int a, int b) {
        return a + b;
    }

    // ✓ Different parameter types
    double compute(double a, double b) {
        return a + b;
    }

    // ✓ Different order of parameter types
    void compute(int a, double b) {
        cout << "int, double\n";
    }

    void compute(double a, int b) {
        cout << "double, int\n";
    }
};
```

Each of these has a **unique signature**, so the compiler can distinguish between them.

### Const Overloading (Special Case for Member Functions)

In C++, you can overload member functions by making one `const` and the other non-`const`. This is called **const overloading**. But it works **only for member functions**, not for free (non-member) functions.

The `const` qualifier becomes part of the function signature for member functions because it affects the type of the implicit `this` pointer:
* Non-const member function: `this` is a pointer to non-const object
* Const member function: `this` is a pointer to const object

#### Example: Const Overloading

```cpp
#include <iostream>
using namespace std;

class Data {
private:
    int value;
public:
    Data(int v) : value(v) {}

    // Non-const version - can modify the object
    int& getValue() {
        cout << "Non-const getValue() called\n";
        return value;
    }

    // Const version - cannot modify the object
    const int& getValue() const {
        cout << "Const getValue() called\n";
        return value;
    }
};

int main() {
    Data d1(10);
    const Data d2(20);

    d1.getValue();      // Calls non-const version
    d2.getValue();      // Calls const version

    d1.getValue() = 50; // Can modify through non-const version
    // d2.getValue() = 60; // ❌ ERROR: Cannot modify through const version

    return 0;
}
```

#### Output:

```
Non-const getValue() called
Const getValue() called
```

**Key Points:**
* The compiler chooses the appropriate version based on whether the object is `const` or non-`const`
* This is useful when you want different behavior or return types for const and non-const objects
* The const version typically returns a const reference to prevent modification

[↑ Back to Table of Contents](#table-of-contents)

---

## When Static Polymorphism Is Not Enough

Static polymorphism works great when you know the **exact types at compile time**. But what if you don't know the exact type until the program is running?

### Real-World Scenario: A Drawing Application

Imagine you're building a drawing application that can draw different shapes: circles, rectangles, triangles, etc.

```cpp
#include <iostream>
#include <vector>
using namespace std;

class Circle {
public:
    void draw() {
        cout << "Drawing a Circle\n";
    }
};

class Rectangle {
public:
    void draw() {
        cout << "Drawing a Rectangle\n";
    }
};

class Triangle {
public:
    void draw() {
        cout << "Drawing a Triangle\n";
    }
};

int main() {
    vector<???> shapes;  // ❌ What type should this be?
    
    // User creates shapes at runtime based on input
    // How do we store different shape types in one collection?
    // How do we call draw() on each without knowing the exact type?
    
    return 0;
}
```

**The Problem:**

* You need to store different shape types in a single collection (like a vector)
* You want to call `draw()` on each shape without knowing its exact type
* The user decides which shapes to create at **runtime** (not compile time)
* Static polymorphism (function overloading) can't help here because the compiler needs to know exact types

**The Solution:** We need **Dynamic Polymorphism** (Runtime Polymorphism)!

[↑ Back to Table of Contents](#table-of-contents)

---

## Dynamic Polymorphism (Runtime Polymorphism)

**Dynamic polymorphism** is achieved when the behavior of a function is decided **at runtime** based on the actual object type, not the reference/pointer type.

Key characteristics:
* The decision of which function to call happens **during program execution**
* Allows you to write code that works with base class pointers/references but calls derived class functions
* Achieved through **inheritance**, **function overriding**, and **virtual functions**

### Function Overriding

**Function overriding** occurs when a derived class provides its own implementation of a function that is already defined in the base class.

Requirements for function overriding:
* Must have the same name
* Must have the same parameters (exact match)
* Must have the same return type (or covariant return type)
* The base class function must be declared as `virtual`

#### Example: Function Overriding

```cpp
#include <iostream>
using namespace std;

class Shape {
public:
    void draw() {  // Non-virtual function
        cout << "Drawing a generic Shape\n";
    }
};

class Circle : public Shape {
public:
    void draw() {  // Overriding the base class function
        cout << "Drawing a Circle\n";
    }
};

int main() {
    Circle circle;
    Shape* shapePtr = &circle;
    
    shapePtr->draw();  // What will this print?
    
    return 0;
}
```

**Output:**
```
Drawing a generic Shape
```

**Problem:** Even though `shapePtr` points to a `Circle` object, it calls the `Shape::draw()` function! This is because the function is **not virtual**, so the call is resolved at compile time based on the pointer type (`Shape*`), not the actual object type (`Circle`).

This is where **virtual functions** come to the rescue!

[↑ Back to Table of Contents](#table-of-contents)

---

### Virtual Functions

A **virtual function** is a member function in the base class that you expect to be overridden in derived classes. When you call a virtual function through a base class pointer or reference, C++ ensures that the correct derived class version is called based on the actual object type.

**Syntax:**
```cpp
class Base {
public:
    virtual void functionName() {
        // Base implementation
    }
};
```

#### Example: Virtual Functions in Action

```cpp
#include <iostream>
using namespace std;

class Shape {
public:
    virtual void draw() {  // Virtual function
        cout << "Drawing a generic Shape\n";
    }
    
    virtual ~Shape() {}  // Virtual destructor
};

class Circle : public Shape {
public:
    void draw() override {  // Overriding the virtual function
        cout << "Drawing a Circle\n";
    }
};

class Rectangle : public Shape {
public:
    void draw() override {
        cout << "Drawing a Rectangle\n";
    }
};

class Triangle : public Shape {
public:
    void draw() override {
        cout << "Drawing a Triangle\n";
    }
};

int main() {
    Shape* s1 = new Circle();
    Shape* s2 = new Rectangle();
    Shape* s3 = new Triangle();

    s1->draw();  // Calls Circle::draw()
    s2->draw();  // Calls Rectangle::draw()
    s3->draw();  // Calls Triangle::draw()

    delete s1;
    delete s2;
    delete s3;

    return 0;
}

```

**Output:**
```
Drawing a Circle
Drawing a Rectangle
Drawing a Triangle
Drawing a Circle
```

**Success!** Now each object calls its own `draw()` function, even though we're using base class pointers. This is **dynamic polymorphism**!

**Key Points:**
* The `virtual` keyword enables runtime polymorphism
* Always declare a virtual destructor in the base class when using polymorphism

[↑ Back to Table of Contents](#table-of-contents)

---

### How Virtual Functions Work: The Mechanism

Virtual functions work through a mechanism involving two key components:

1. **Virtual Pointer (vptr)** - A hidden pointer in each object
2. **Virtual Table (vtable)** - A table of function pointers for each class

#### Understanding vptr and vtable

When a class has at least one virtual function:

* **The compiler creates a vtable** (virtual table) for that class
  * The vtable is a static array of function pointers
  * Each entry points to the most-derived version of a virtual function
  * One vtable per class (not per object)

* **Each object gets a vptr** (virtual pointer)
  * The vptr is a hidden member variable added by the compiler
  * It points to the vtable of that object's class
  * Each object has its own vptr

#### Visual Representation

```cpp
class Shape {
public:
    virtual void draw() { cout << "Shape\n"; }
    virtual void area() { cout << "Shape area\n"; }
};

class Circle : public Shape {
public:
    void draw() override { cout << "Circle\n"; }
    void area() override { cout << "Circle area\n"; }
};
```

**Memory Layout:**

```
Shape Object:                    Circle Object:
+-----------------+              +-----------------+
| vptr (8 bytes)  |--+           | vptr (8 bytes)  |--+
+-----------------+  |           +-----------------+  |
                     |                                |
                     v                                v
Shape's vtable:              Circle's vtable:
+-----------------+          +-----------------+
| &Shape::draw    |          | &Circle::draw   |
| &Shape::area    |          | &Circle::area   |
+-----------------+          +-----------------+
```

**Key Observations:**
* The vptr is typically the first member of the object (8 bytes on 64-bit systems)
* Each class with virtual functions has its own vtable
* All objects of the same class share the same vtable but have their own vptr

#### Size Impact

```cpp
class WithoutVirtual {
    int x;  // 4 bytes
};

class WithVirtual {
    int x;  // 4 bytes
    virtual void func() {}
    // + vptr (8 bytes on 64-bit)
};

cout << sizeof(WithoutVirtual);  // Output: 4 bytes
cout << sizeof(WithVirtual);     // Output: 16 bytes (4 + 8 + padding)
```

[↑ Back to Table of Contents](#table-of-contents)

---

### How a Virtual Function Call Gets Resolved

When you call a virtual function through a pointer or reference, here's what happens:

#### Step-by-Step Process

```cpp
Shape* shapePtr = new Circle();
shapePtr->draw();  // How does this get resolved?
```

**Step 1: Dereference the vptr**
* The program accesses the object through `shapePtr`
* It reads the vptr from the object (first 8 bytes)
* The vptr points to Circle's vtable

**Step 2: Look up the function in the vtable**
* The compiler knows that `draw()` is the first virtual function (index 0)
* It accesses `vtable[0]` to get the address of the function

**Step 3: Call the function**
* The program jumps to the function address found in the vtable
* In this case, it calls `Circle::draw()`

#### Pseudo-code Representation

```cpp
// What you write:
shapePtr->draw();

// What actually happens (conceptually):
(*(shapePtr->vptr[0]))(shapePtr);
//  ^     ^      ^       ^
//  |     |      |       |
//  |     |      |       +-- Pass 'this' pointer
//  |     |      +---------- Index 0 for draw()
//  |     +----------------- Access vptr
//  +----------------------- Dereference function pointer and call
```

#### Performance Characteristics

**Virtual Function Call:**
* 2 memory accesses (vptr lookup + vtable lookup)
* 1 indirect function call
* Slightly slower than direct function calls
* Cannot be inlined by the compiler

**Non-Virtual Function Call:**
* Direct function call
* Can be inlined by the compiler
* Faster

**Benchmark (approximate):**
* Virtual function call: ~2-3 nanoseconds overhead
* For most applications, this overhead is negligible

#### Complete Example with Explanation

```cpp
#include <iostream>
using namespace std;

class Animal {
public:
    virtual void speak() {
        cout << "Animal speaks\n";
    }
    
    virtual void eat() {
        cout << "Animal eats\n";
    }
};

class Dog : public Animal {
public:
    void speak() override {
        cout << "Dog barks\n";
    }
    
    void eat() override {
        cout << "Dog eats bones\n";
    }
};

int main() {
    Animal* animalPtr = new Dog();
    
    animalPtr->speak();  
    // Step 1: Access animalPtr->vptr → Points to Dog's vtable
    // Step 2: Look up vtable[0] → &Dog::speak
    // Step 3: Call Dog::speak()
    // Output: "Dog barks"
    
    animalPtr->eat();
    // Step 1: Access animalPtr->vptr → Points to Dog's vtable
    // Step 2: Look up vtable[1] → &Dog::eat
    // Step 3: Call Dog::eat()
    // Output: "Dog eats bones"
    
    delete animalPtr;
    return 0;
}
```

**Output:**
```
Dog barks
Dog eats bones
```

**Why This Works:**
* Even though `animalPtr` is of type `Animal*`, the object it points to is a `Dog`
* The `Dog` object's vptr points to `Dog`'s vtable
* The vtable contains pointers to `Dog`'s overridden functions
* At runtime, the correct functions are called based on the actual object type

#### What If a Derived Class Doesn't Override All Virtual Functions?

When a derived class doesn't override a virtual function, the base class version is used in the derived class's vtable.

```cpp
#include <iostream>
using namespace std;

class Base {
public:
    virtual void func1() {
        cout << "Base::func1()\n";
    }
    
    virtual void func2() {
        cout << "Base::func2()\n";
    }
    
    virtual void func3() {
        cout << "Base::func3()\n";
    }
};

class Derived : public Base {
public:
    void func1() override {
        cout << "Derived::func1()\n";
    }
    
    // func2() is NOT overridden
    
    void func3() override {
        cout << "Derived::func3()\n";
    }
};

int main() {
    Base* basePtr = new Derived();
    
    basePtr->func1();  // Calls Derived::func1()
    basePtr->func2();  // Calls Base::func2() (not overridden)
    basePtr->func3();  // Calls Derived::func3()
    
    delete basePtr;
    return 0;
}
```

**Output:**
```
Derived::func1()
Base::func2()
Derived::func3()
```

**vtable Layout:**

```
Base's vtable:                Derived's vtable:
+-------------------+         +-------------------+
| &Base::func1      |         | &Derived::func1   | ← Overridden
| &Base::func2      |         | &Base::func2      | ← NOT overridden, inherits Base's
| &Base::func3      |         | &Derived::func3   | ← Overridden
+-------------------+         +-------------------+
```

**Key Insight:**
* When `Derived` doesn't override `func2()`, its vtable entry still points to `Base::func2()`
* The derived class "inherits" the base class function pointer in its vtable
* This is why calling `basePtr->func2()` executes `Base::func2()` even though the object is of type `Derived`
* The vtable ensures that each function call resolves to the most-derived version available

#### Why Virtual Destructors Are Critical

When using polymorphism, **always make the base class destructor virtual**. If you don't, deleting a derived class object through a base class pointer will only call the base class destructor, causing a memory leak!

```cpp
#include <iostream>
using namespace std;

class Base {
public:
    Base() { cout << "Base Constructor\n"; }
    ~Base() { cout << "Base Destructor\n"; }  // ❌ NOT virtual
};

class Derived : public Base {
    int* data;
public:
    Derived() { 
        data = new int[100];
        cout << "Derived Constructor\n"; 
    }
    ~Derived() { 
        delete[] data;
        cout << "Derived Destructor\n"; 
    }
};

int main() {
    Base* ptr = new Derived();
    delete ptr;  // ⚠️ Memory leak! Only Base destructor called
    return 0;
}
```

**Output:**
```
Base Constructor
Derived Constructor
Base Destructor
```

**Problem:** `Derived` destructor never called → `data` array leaked!

**Solution: Make Base Destructor Virtual**

```cpp
class Base {
public:
    Base() { cout << "Base Constructor\n"; }
    virtual ~Base() { cout << "Base Destructor\n"; }  // ✓ Virtual
};

// ... rest same ...

int main() {
    Base* ptr = new Derived();
    delete ptr;  // ✓ Both destructors called correctly
    return 0;
}
```

**Output:**
```
Base Constructor
Derived Constructor
Derived Destructor
Base Destructor
```

**Rule of Thumb:** If a class has any virtual functions, its destructor should be virtual too!

[↑ Back to Table of Contents](#table-of-contents)

---


### The `override` Keyword (C++11)


C++11 introduced the `override` keyword to make your code safer and more explicit when overriding virtual functions. It's not required, but it's highly recommended!

#### What Does `override` Do?

The `override` keyword tells the compiler: **"I intend to override a virtual function from the base class."**

If you make a mistake (wrong parameter types, misspelled name, forgot `const`, etc.), the compiler will give you an error instead of silently creating a new function.

#### Problem Without `override`

```cpp
class Base {
public:
    virtual void setValue(int val) {
        cout << "Base::setValue\n";
    }
};

class Derived : public Base {
public:
    // Oops! Typo: "vlaue" instead of "value"
    // Also wrong parameter type: double instead of int
    virtual void setValue(double val) {  // ❌ NOT overriding!
        cout << "Derived::setValue\n";
    }
};

int main() {
    Base* ptr = new Derived();
    ptr->setValue(10);  // Calls Base::setValue (unexpected!)
    delete ptr;
    return 0;
}
```

**Output:**
```
Base::setValue
```

**Problem:** The compiler doesn't warn you! It thinks you're creating a new overloaded function, not overriding the base class function.

#### Solution With `override`

```cpp
class Base {
public:
    virtual void setValue(int val) {
        cout << "Base::setValue\n";
    }
};

class Derived : public Base {
public:
    void setValue(double val) override {  // ✓ Compiler error!
        cout << "Derived::setValue\n";
    }
};
```

**Compiler Error:**
```
error: 'void Derived::setValue(double)' marked 'override', but does not override
```

**The compiler catches your mistake immediately!**

#### Correct Usage

```cpp
class Base {
public:
    virtual void setValue(int val) {
        cout << "Base::setValue\n";
    }
};

class Derived : public Base {
public:
    void setValue(int val) override {  // ✓ Correct override
        cout << "Derived::setValue\n";
    }
};

int main() {
    Base* ptr = new Derived();
    ptr->setValue(10);  // Calls Derived::setValue (as expected!)
    delete ptr;
    return 0;
}
```

**Output:**
```
Derived::setValue
```

#### Benefits of Using `override`

1. **Catches typos** - Misspelled function names
2. **Catches signature mismatches** - Wrong parameter types or count
3. **Catches const mismatches** - Forgot `const` qualifier
4. **Self-documenting** - Makes it clear you're overriding, not creating a new function
5. **Refactoring safety** - If the base class function signature changes, you'll get compilation errors

#### Complete Example

```cpp
#include <iostream>
using namespace std;

class Animal {
public:
    virtual void makeSound() const {
        cout << "Animal sound\n";
    }
    
    virtual ~Animal() {}
};

class Dog : public Animal {
public:
    void makeSound() const override {  // ✓ Correct
        cout << "Woof!\n";
    }
};

class Cat : public Animal {
public:
    void makeSound() override {  // ❌ Compiler error: missing 'const'
        cout << "Meow!\n";
    }
};

int main() {
    Animal* animal = new Dog();
    animal->makeSound();
    delete animal;
    return 0;
}
```

**Best Practice:** Always use `override` when overriding virtual functions in modern C++ (C++11 and later)!

[↑ Back to Table of Contents](#table-of-contents)

---


### The final Keyword (C++11)


The `final` keyword, introduced in C++11, is used to restrict inheritance and method overriding. It can be applied in two ways:

1. **Final Class** - Prevents a class from being inherited
2. **Final Method** - Prevents a virtual method from being overridden in derived classes

#### Example: Using final

```cpp
#include <iostream>
using namespace std;

// Base class with a final method
class Base {
public:
    virtual void canOverride() {
        cout << "Base: This can be overridden" << endl;
    }
    
    // This method cannot be overridden
    virtual void cannotOverride() final {
        cout << "Base: This is final - cannot be overridden" << endl;
    }
};

// This class cannot be inherited from
class FinalClass final {
public:
    void display() {
        cout << "This is a final class" << endl;
    }
};

// Derived class from Base
class Derived : public Base {
public:
    // Allowed - overriding non-final method
    void canOverride() override {
        cout << "Derived: Overridden successfully" << endl;
    }
    
    // ERROR: Cannot override final method
    // void cannotOverride() override {
    //     cout << "This will cause compilation error" << endl;
    // }
};

// ERROR: Cannot inherit from final class
// class AnotherClass : public FinalClass {
//     // Compilation error
// };

int main() {
    Derived d;
    d.canOverride();      // Calls overridden version
    d.cannotOverride();   // Calls Base's final version
    
    FinalClass fc;
    fc.display();
    
    return 0;
}
```

**Output:**
```
Derived: Overridden successfully
Base: This is final - cannot be overridden
This is a final class
```

**Key Points:**
- Use `final` on a class to prevent any inheritance from it
- Use `final` on a virtual method to prevent derived classes from overriding it
- Attempting to violate `final` restrictions results in a compile-time error
- The `final` keyword provides clear intent and compiler-enforced restrictions

*For in-depth details about the `final` keyword, refer to the [C++11 final Keyword](#) section.*

[↑ Back to Table of Contents](#table-of-contents)

---

## Overloading vs Overriding: Quick Comparison

Here's a side-by-side comparison to help you understand the key differences:

| **Aspect** | **Function Overloading** | **Function Overriding** |
|------------|-------------------------|------------------------|
| **Type of Polymorphism** | Static (Compile-time) | Dynamic (Runtime) |
| **When is it resolved?** | At compile time | At runtime |
| **Where does it occur?** | Same class (or across classes) | Base and derived classes (inheritance required) |
| **Function signature** | Must be different (different parameters) | Must be same (same name, parameters, return type) |
| **`virtual` keyword** | Not required | Required in base class |
| **`override` keyword** | Not applicable | Recommended (C++11+) |
| **Function name** | Same name, different parameters | Same name, same parameters |
| **Return type** | Can be same or different | Must be same (or covariant) |
| **Purpose** | Provide multiple ways to call same function name with different arguments | Provide specific implementation in derived class for base class behavior |
| **Example** | `print(int)`, `print(double)`, `print(string)` | Base: `virtual void draw()`, Derived: `void draw() override` |
| **Relationship** | Independent functions in same scope | Child class redefines parent class function |
| **Pointer/Reference type** | Not relevant (direct call) | Important (base pointer/reference to derived object) |

### Quick Example Comparison

```cpp
#include <iostream>
using namespace std;

// OVERLOADING (Static Polymorphism)
class Calculator {
public:
    int add(int a, int b) {
        return a + b;
    }
    
    double add(double a, double b) {  // Different parameter types
        return a + b;
    }
    
    int add(int a, int b, int c) {  // Different number of parameters
        return a + b + c;
    }
};

// OVERRIDING (Dynamic Polymorphism)
class Animal {
public:
    virtual void sound() {
        cout << "Animal makes a sound\n";
    }
};

class Dog : public Animal {
public:
    void sound() override {  // Same signature, different implementation
        cout << "Dog barks\n";
    }
};

int main() {
    // Overloading - Compiler decides which function to call
    Calculator calc;
    cout << calc.add(5, 3) << "\n";        // Calls add(int, int)
    cout << calc.add(5.5, 3.2) << "\n";    // Calls add(double, double)
    cout << calc.add(1, 2, 3) << "\n";     // Calls add(int, int, int)
    
    cout << "---\n";
    
    // Overriding - Runtime decides which function to call
    Animal* animalPtr = new Dog();
    animalPtr->sound();  // Calls Dog::sound() at runtime
    
    delete animalPtr;
    return 0;
}
```

**Output:**
```
8
8.7
6
---
Dog barks
```

**Key Takeaway:**
- **Overloading** = Same name, different signatures → Compile-time decision
- **Overriding** = Same name, same signature, inheritance → Runtime decision

[↑ Back to Table of Contents](#table-of-contents)
