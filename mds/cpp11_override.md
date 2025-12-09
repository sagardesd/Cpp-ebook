# C++11 Override Keyword

## Table of Contents
1. [What is the Override Keyword?](#what-is-the-override-keyword)
2. [Definition](#definition)
3. [The Problem Without Override](#the-problem-without-override)
   - [Example 1: Typo in Function Name](#example-1-typo-in-function-name)
   - [Example 2: Wrong Parameter Types](#example-2-wrong-parameter-types)
   - [Example 3: Missing const Qualifier](#example-3-missing-const-qualifier)
4. [The Solution: Using Override Keyword](#the-solution-using-override-keyword)
   - [Correct Usage](#correct-usage)
   - [Catching Errors at Compile Time](#catching-errors-at-compile-time)
5. [Benefits of Override Keyword](#benefits-of-override-keyword)
6. [Best Practices](#best-practices)

---

## What is the Override Keyword?

The **override** keyword is a C++11 feature that explicitly indicates that a member function in a derived class is intended to override a virtual function from the base class. It provides compile-time checking to ensure the override is valid.

[↑ Back to Table of Contents](#table-of-contents)

---

## Definition

The `override` keyword is placed after the function signature in a derived class to explicitly declare that the function overrides a virtual function from the base class.

**Syntax:**
```cpp
class Base {
public:
    virtual void functionName() {
        // base implementation
    }
};

class Derived : public Base {
public:
    void functionName() override {  // Explicitly marks as override
        // derived implementation
    }
};
```

[↑ Back to Table of Contents](#table-of-contents)

---

## The Problem Without Override

Without the `override` keyword, subtle mistakes in function signatures can lead to bugs that are difficult to detect. The compiler won't warn you if you accidentally create a new function instead of overriding the base class function.

### Example 1: Typo in Function Name

```cpp
#include <iostream>

class Shape {
public:
    virtual void draw() {
        std::cout << "Drawing a shape" << std::endl;
    }
    
    virtual ~Shape() = default;
};

class Circle : public Shape {
public:
    void darw() {  // Typo: 'darw' instead of 'draw'
        std::cout << "Drawing a circle" << std::endl;
    }
};

int main() {
    Shape* shape = new Circle();
    shape->draw();  // Calls Shape::draw(), not Circle::darw()
    delete shape;
    return 0;
}
```

**Output:**
```
Drawing a shape
```

**Problem:** The typo `darw()` creates a new function instead of overriding `draw()`. The compiler doesn't warn you, and the base class function is called instead of the derived class function.

---

### Example 2: Wrong Parameter Types

```cpp
#include <iostream>

class Animal {
public:
    virtual void makeSound(int volume) {
        std::cout << "Animal sound at volume " << volume << std::endl;
    }
    
    virtual ~Animal() = default;
};

class Dog : public Animal {
public:
    void makeSound(double volume) {  // Wrong parameter type: double instead of int
        std::cout << "Woof at volume " << volume << std::endl;
    }
};

int main() {
    Animal* animal = new Dog();
    animal->makeSound(5);  // Calls Animal::makeSound(int), not Dog::makeSound(double)
    delete animal;
    return 0;
}
```

**Output:**
```
Animal sound at volume 5
```

**Problem:** The parameter type doesn't match (`double` vs `int`), so this creates a new function instead of overriding. The base class function is called.

---

### Example 3: Missing const Qualifier

```cpp
#include <iostream>

class Vehicle {
public:
    virtual void getInfo() const {
        std::cout << "Vehicle info" << std::endl;
    }
    
    virtual ~Vehicle() = default;
};

class Car : public Vehicle {
public:
    void getInfo() {  // Missing 'const' qualifier
        std::cout << "Car info" << std::endl;
    }
};

int main() {
    Vehicle* vehicle = new Car();
    vehicle->getInfo();  // Calls Vehicle::getInfo(), not Car::getInfo()
    delete vehicle;
    return 0;
}
```

**Output:**
```
Vehicle info
```

**Problem:** Missing `const` qualifier means the signature doesn't match, creating a new function instead of overriding.

[↑ Back to Table of Contents](#table-of-contents)

---

## The Solution: Using Override Keyword

### Correct Usage

```cpp
#include <iostream>

class Shape {
public:
    virtual void draw() {
        std::cout << "Drawing a shape" << std::endl;
    }
    
    virtual void area() const {
        std::cout << "Calculating shape area" << std::endl;
    }
    
    virtual ~Shape() = default;
};

class Circle : public Shape {
public:
    void draw() override {  // Correctly overrides Shape::draw()
        std::cout << "Drawing a circle" << std::endl;
    }
    
    void area() const override {  // Correctly overrides Shape::area()
        std::cout << "Calculating circle area" << std::endl;
    }
};

int main() {
    Shape* shape = new Circle();
    shape->draw();   // Calls Circle::draw()
    shape->area();   // Calls Circle::area()
    delete shape;
    return 0;
}
```

**Output:**
```
Drawing a circle
Calculating circle area
```

**Success:** The derived class functions are correctly called because they properly override the base class functions.

---

### Catching Errors at Compile Time

**Example 1: Typo Caught by Override**
```cpp
class Shape {
public:
    virtual void draw() {
        std::cout << "Drawing a shape" << std::endl;
    }
};

class Circle : public Shape {
public:
    void darw() override {  // Compilation Error!
        std::cout << "Drawing a circle" << std::endl;
    }
};
```

**Compiler Error:**
```
error: 'void Circle::darw()' marked 'override', but does not override
```

---

**Example 2: Wrong Parameter Type Caught**
```cpp
class Animal {
public:
    virtual void makeSound(int volume) {
        std::cout << "Animal sound" << std::endl;
    }
};

class Dog : public Animal {
public:
    void makeSound(double volume) override {  // Compilation Error!
        std::cout << "Woof" << std::endl;
    }
};
```

**Compiler Error:**
```
error: 'void Dog::makeSound(double)' marked 'override', but does not override
```

---

**Example 3: Missing const Caught**
```cpp
class Vehicle {
public:
    virtual void getInfo() const {
        std::cout << "Vehicle info" << std::endl;
    }
};

class Car : public Vehicle {
public:
    void getInfo() override {  // Compilation Error!
        std::cout << "Car info" << std::endl;
    }
};
```

**Compiler Error:**
```
error: 'void Car::getInfo()' marked 'override', but does not override
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Benefits of Override Keyword

1. **Compile-time Error Detection**: Catches mistakes early when the function signature doesn't match the base class
2. **Self-documenting Code**: Makes it clear that a function is intended to override a base class function
3. **Refactoring Safety**: If the base class function signature changes, the compiler will catch all derived classes that need updating
4. **Prevents Silent Bugs**: Eliminates bugs caused by accidentally creating new functions instead of overriding
5. **Better Code Maintenance**: Easier to understand class hierarchies and relationships
6. **No Runtime Overhead**: It's a compile-time feature with zero runtime cost

[↑ Back to Table of Contents](#table-of-contents)

---

## Best Practices

1. **Always use `override`** when you intend to override a virtual function
2. **Use `virtual` only in base classes** for the initial declaration
3. **Don't use both `virtual` and `override`** in derived classes (redundant)
4. **Mark base class destructors as `virtual`** when using inheritance
5. **Consider using `final`** to prevent further overriding if needed

**Example of Best Practices:**
```cpp
class Base {
public:
    virtual void foo() { }
    virtual void bar() { }
    virtual ~Base() = default;  // Virtual destructor
};

class Derived : public Base {
public:
    void foo() override { }      // Good: uses override
    void bar() override final { } // Good: override and prevent further overriding
};

class FurtherDerived : public Derived {
public:
    void foo() override { }      // Good: overrides Derived::foo()
    // void bar() override { }   // Error: bar is final in Derived
};
```

[↑ Back to Table of Contents](#table-of-contents)