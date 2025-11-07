# C++ Inheritance: Complete Tutorial

## What is Inheritance?

Imagine you work at a company. All employees share common properties like name, employee ID, and salary. But different roles have additional specific properties:

- **Developers** have programming languages they know
- **Managers** have a team size they manage
- **HR Staff** have recruitment targets

Instead of rewriting common properties for each role, inheritance lets you define them once in a base "Employee" class and extend it for specific roles. This is exactly how inheritance works in C++.

**In simple terms:** Inheritance is when a class (child/derived class) inherits properties and behaviors from another class (parent/base class), allowing you to reuse code and create a hierarchical relationship.

## Basic Syntax of Inheritance

```cpp
class BaseClassName {
    // Base class members
};

class DerivedClassName : access_specifier BaseClassName {
    // Derived class members
    // + Inherited members from BaseClassName
};
```

**Components:**
- `BaseClassName`: The class being inherited from (also called parent class or superclass)
- `DerivedClassName`: The class that inherits (also called child class or subclass)
- `access_specifier`: How inheritance is done (`public`, `protected`, or `private`)
- `:` (colon): Indicates inheritance relationship

### Simple Example

```cpp
// Base class
class Animal {
public:
    void eat() {
        cout << "Eating..." << endl;
    }
};

// Derived class inherits from Animal
class Dog : public Animal {
public:
    void bark() {
        cout << "Woof!" << endl;
    }
};

// Usage
Dog myDog;
myDog.eat();   // Inherited from Animal
myDog.bark();  // Dog's own method
```

## Understanding Base Class and Derived Class

### Base Class (Parent Class / Superclass)
The **base class** is the class that provides the common properties and behaviors to be inherited. It's the "general" class.

**Characteristics:**
- Contains common/shared functionality
- Defined first, independently
- Can exist and be used on its own
- Doesn't know about its derived classes

```cpp
class Employee {  // BASE CLASS
public:
    string name;
    int employeeID;
    void displayInfo() {
        cout << "Employee: " << name << endl;
    }
};
```

### Derived Class (Child Class / Subclass)
The **derived class** is the class that inherits from the base class and adds its own specific properties and behaviors. It's the "specialized" class.

**Characteristics:**
- Inherits all accessible members from base class
- Adds its own specific functionality
- Cannot exist without the base class definition
- Can override base class behaviors

```cpp
class Developer : public Employee {  // DERIVED CLASS
public:
    string programmingLanguage;  // Additional property
    void code() {                // Additional method
        cout << name << " is coding" << endl;  // Can use inherited 'name'
    }
};
```

### Visual Relationship

```
        ┌─────────────────┐
        │   Employee      │  ◄── BASE CLASS (Parent)
        │  (Base Class)   │
        └────────┬────────┘
                 │ inherits from
                 │
        ┌────────▼────────┐
        │   Developer     │  ◄── DERIVED CLASS (Child)
        │ (Derived Class) │
        └─────────────────┘
```

### What Gets Inherited?

```cpp
class Base {
public:
    int publicVar;      // ✓ Inherited and accessible
protected:
    int protectedVar;   // ✓ Inherited and accessible (in derived class only)
private:
    int privateVar;     // ✓ Inherited but NOT accessible
    
public:
    void publicMethod() { }     // ✓ Inherited and accessible
protected:
    void protectedMethod() { }  // ✓ Inherited and accessible (in derived class only)
private:
    void privateMethod() { }    // ✓ Inherited but NOT accessible
};

class Derived : public Base {
    // Has: publicVar, protectedVar, publicMethod(), protectedMethod()
    // Doesn't have access to: privateVar, privateMethod()
    // (but they exist in memory!)
};
```

**Key Point:** Private members ARE inherited (they exist in the derived object's memory), but the derived class cannot directly access them.

### Real-World Example: Company Employee System

```cpp
// Base class - Common properties for ALL employees
class Employee {
public:
    string name;
    int employeeID;
    double salary;
    
    void displayBasicInfo() {
        cout << "Name: " << name << endl;
        cout << "ID: " << employeeID << endl;
        cout << "Salary: $" << salary << endl;
    }
};

// Derived class - Specific to developers
class Developer : public Employee {
public:
    string programmingLanguage;
    
    void code() {
        cout << name << " is coding in " << programmingLanguage << endl;
    }
};

// Derived class - Specific to managers
class Manager : public Employee {
public:
    int teamSize;
    
    void conductMeeting() {
        cout << name << " is conducting a meeting with " << teamSize << " team members" << endl;
    }
};
```

**Usage:**
```cpp
Developer dev;
dev.name = "Alice";           // Inherited from Employee
dev.employeeID = 101;         // Inherited from Employee
dev.salary = 80000;           // Inherited from Employee
dev.programmingLanguage = "C++";  // Specific to Developer
dev.displayBasicInfo();       // Inherited method
dev.code();                   // Developer's own method
```

## Why Use Inheritance?

### Benefits of Inheritance

1. **Code Reusability**: Write common code once, use it everywhere
   - No need to repeat `name`, `employeeID`, `salary` in every employee type
   
2. **Easy Maintenance**: Fix bugs in one place
   - If you fix a bug in the `displayBasicInfo()` method, it's fixed for all employee types
   
3. **Logical Organization**: Models real-world relationships
   - Clearly shows that Developer "is-a" Employee
   
4. **Extensibility**: Easy to add new employee types
   - Adding a `SalesRep` class? Just inherit from `Employee`
   
5. **Polymorphism Support**: Treat different types uniformly (covered in later chapters)
   - Store all employees in one array, regardless of their specific type

## Protected Access Specifier

C++ has three access specifiers: `private`, `protected`, and `public`. The `protected` keyword is particularly important in inheritance.

### Access Specifier Comparison

| Access Specifier | Accessible in Same Class | Accessible in Derived Class | Accessible Outside Class |
|------------------|-------------------------|----------------------------|-------------------------|
| `private`        | ✓ Yes                   | ✗ No                        | ✗ No                    |
| `protected`      | ✓ Yes                   | ✓ Yes                       | ✗ No                    |
| `public`         | ✓ Yes                   | ✓ Yes                       | ✓ Yes                   |

### When to Use Protected

Use `protected` when you want derived classes to access members, but not outside code.

```cpp
class Employee {
protected:
    double baseSalary;      // Derived classes can access
    
private:
    string bankAccount;     // Only Employee class can access
    
public:
    string name;            // Everyone can access
    
    void setSalary(double salary) {
        baseSalary = salary;
    }
};

class Developer : public Employee {
public:
    void calculateBonus() {
        // Can access baseSalary (protected)
        double bonus = baseSalary * 0.15;
        cout << "Bonus: $" << bonus << endl;
        
        // Cannot access bankAccount (private)
        // bankAccount = "123456"; // ERROR!
    }
};
```

**Best Practice:** Use `protected` for data that derived classes need to access but should remain hidden from external code.

## Types of Inheritance: Private, Protected, and Public

The inheritance type controls how base class members are inherited.

### Syntax
```cpp
class Derived : access_specifier Base {
    // access_specifier can be private, protected, or public
};
```

### How Inheritance Types Affect Access

| Base Class Member | Public Inheritance | Protected Inheritance | Private Inheritance |
|-------------------|-------------------|----------------------|---------------------|
| `public`          | `public`          | `protected`          | `private`           |
| `protected`       | `protected`       | `protected`          | `private`           |
| `private`         | Not accessible    | Not accessible       | Not accessible      |

### 1. Public Inheritance (Most Common)

**"IS-A" relationship** - Developer IS-A Employee

```cpp
class Employee {
public:
    string name;
protected:
    double salary;
private:
    string ssn;
};

class Developer : public Employee {
    // name remains public
    // salary remains protected
    // ssn is not accessible
};

Developer dev;
dev.name = "Bob";  // OK - name is public
```

### 2. Protected Inheritance

**"Implemented-in-terms-of" relationship** - Less common

```cpp
class Employee {
public:
    string name;
protected:
    double salary;
};

class Developer : protected Employee {
    // name becomes protected (was public)
    // salary remains protected
};

Developer dev;
dev.name = "Bob";  // ERROR! name is now protected
```

### 3. Private Inheritance

**"Implemented-in-terms-of" relationship** - Hides the base class completely

```cpp
class Employee {
public:
    string name;
protected:
    double salary;
};

class Developer : private Employee {
    // name becomes private (was public)
    // salary becomes private (was protected)
};

Developer dev;
dev.name = "Bob";  // ERROR! name is now private
```

**Most Common:** Use **public inheritance** 99% of the time. Use protected/private inheritance only when you want to hide the base class interface.

## Object Size in Inheritance Hierarchy

When a class inherits from another, the derived class object contains **all members from both classes**.

### Memory Layout Diagram

```cpp
class Employee {
    string name;        // 32 bytes (typical string size)
    int employeeID;     // 4 bytes
    double salary;      // 8 bytes
};  // Total: ~44 bytes

class Developer : public Employee {
    string programmingLanguage;  // 32 bytes
    int yearsOfExperience;       // 4 bytes
};  // Total: ~80 bytes (44 + 36)
```

**Visual Representation:**

```
Employee Object:
┌─────────────────────────────┐
│ name (32 bytes)             │
├─────────────────────────────┤
│ employeeID (4 bytes)        │
├─────────────────────────────┤
│ salary (8 bytes)            │
└─────────────────────────────┘
Total: ~44 bytes


Developer Object:
┌─────────────────────────────┐
│ Employee Part:              │
│  - name (32 bytes)          │
│  - employeeID (4 bytes)     │
│  - salary (8 bytes)         │
├─────────────────────────────┤
│ Developer Part:             │
│  - programmingLanguage      │
│    (32 bytes)               │
│  - yearsOfExperience        │
│    (4 bytes)                │
└─────────────────────────────┘
Total: ~80 bytes
```

### Key Points About Object Size

1. **Derived objects are always larger** than base objects (or equal if no new members)
2. **Base class portion comes first** in memory
3. You can check sizes using `sizeof()`:

```cpp
cout << "Employee size: " << sizeof(Employee) << " bytes" << endl;
cout << "Developer size: " << sizeof(Developer) << " bytes" << endl;
```

## Casting Objects: Upcasting and Downcasting

### Upcasting (Safe) ✓

**Upcasting** = Converting derived class pointer/reference to base class pointer/reference

```cpp
Developer dev;
dev.name = "Charlie";
dev.programmingLanguage = "Python";

// Upcasting - ALWAYS SAFE
Employee* empPtr = &dev;  // Developer* → Employee*
empPtr->displayBasicInfo();  // Works fine

// But loses access to derived class members
// empPtr->code();  // ERROR! Employee doesn't have code()
```

**Why it's safe:** Every Developer IS-AN Employee, so treating it as Employee is always valid.

### Downcasting (Risky) ⚠️

**Downcasting** = Converting base class pointer/reference to derived class pointer/reference

```cpp
Employee* empPtr = new Employee();

// Downcasting - DANGEROUS without checking!
Developer* devPtr = (Developer*)empPtr;  // C-style cast - risky!
devPtr->code();  // RUNTIME ERROR! empPtr wasn't actually pointing to a Developer
```

### Safe Downcasting with dynamic_cast

```cpp
Employee* empPtr = new Developer();  // Actually points to Developer

// Safe downcasting using dynamic_cast
Developer* devPtr = dynamic_cast<Developer*>(empPtr);

if (devPtr != nullptr) {
    // Successfully casted - empPtr was really a Developer
    devPtr->code();
} else {
    // Cast failed - empPtr wasn't a Developer
    cout << "Not a Developer!" << endl;
}
```

**Requirements for `dynamic_cast`:**
- Base class must have at least one virtual function
- Only works with pointers and references
- Returns `nullptr` for pointers or throws `bad_cast` exception for references if cast fails

### Best Practices for Casting

1. **Prefer Upcasting:** It's safe and natural
2. **Avoid Downcasting when possible:** Design your code to minimize need for downcasting
3. **Use `dynamic_cast` for Downcasting:** Never use C-style casts for downcasting
4. **Always check `dynamic_cast` results:** Handle the case where it returns `nullptr`
5. **Consider virtual functions instead:** Often better than downcasting

### Common Casting Failures at Runtime

```cpp
// Failure Case 1: Casting to wrong derived class
Employee* emp = new Manager();
Developer* dev = dynamic_cast<Developer*>(emp);  // Returns nullptr - emp is Manager, not Developer

// Failure Case 2: Slicing problem
Developer dev;
Employee emp = dev;  // Copies only Employee part, loses Developer data (object slicing)

// Failure Case 3: Casting without virtual functions
class Base { int x; };  // No virtual functions
class Derived : public Base { int y; };
Base* b = new Derived();
Derived* d = dynamic_cast<Derived*>(b);  // Compile error! Need virtual functions
```

## Coming Up Next: Advanced Inheritance Concepts

In the following chapters, we'll explore concepts that are deeply related to and build upon inheritance:

### 1. **Constructors and Destructors in Inheritance**
- How derived class constructors call base class constructors
- Order of construction and destruction
- Passing arguments to base class constructors

### 2. **Virtual Functions and Polymorphism**
- Runtime polymorphism through virtual functions
- Virtual function tables (vtables)
- Pure virtual functions and abstract classes

### 3. **Function Overriding**
- How derived classes override base class methods
- The `override` keyword
- Difference between overriding and overloading

### 4. **Multiple Inheritance**
- Inheriting from multiple base classes
- The diamond problem
- Virtual inheritance

### 5. **Virtual Destructors**
- Why destructors should be virtual in base classes
- Memory leak prevention
- Proper cleanup in inheritance hierarchies

### 6. **Access Control in Inheritance**
- Using `using` declarations to change access
- Friend functions and inheritance
- Protected inheritance use cases

### 7. **Object Slicing**
- What happens when you assign derived to base
- How to avoid slicing problems
- Using pointers and references

### 8. **Composition vs Inheritance**
- "Has-A" vs "IS-A" relationships
- When to use composition instead
- Design guidelines

### 9. **Abstract Classes and Interfaces**
- Creating interfaces using pure virtual functions
- Designing flexible, extensible systems
- Interface segregation principles

Each of these topics expands on the foundation of inheritance and helps you build robust, maintainable object-oriented systems in C++!