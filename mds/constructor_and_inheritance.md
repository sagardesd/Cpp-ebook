# Constructor Execution in Inheritance - C++

## Table of Contents
1. [Understanding Constructor Execution Order](#understanding-constructor-execution-order)
2. [Default Constructor Behavior](#default-constructor-behavior)
3. [Execution Sequence Analysis](#execution-sequence-analysis)
4. [Calling Parameterized Base Constructors](#calling-parameterized-base-constructors)
5. [Complete Example with Explanation](#complete-example-with-explanation)
6. [Inheriting constructors - C++11](#inheriting-constructors-cpp11)
7. [Limitation of inherited construtors - C++11](#limitations-of-inherited-constructors)
8. [Understanding Destructor Execution Order](#destructors-order-inheritance)

---

<a id="understanding-constructor-execution-order"></a>
## 1. Understanding Constructor Execution Order

When creating an object of a derived class, constructors are called in a specific order:

### Order of Construction:
1. **Base class constructor** (top of hierarchy) - **First**
2. **Intermediate class constructors** (if any)
3. **Derived class constructor** (bottom of hierarchy) - **Last**

### Order of Destruction: (Reverse order)
1. **Derived class destructor** - **First**
2. **Intermediate class destructors**
3. **Base class destructor** - **Last**

### Why This Order?

The derived class depends on the base class being fully constructed first. You can't build a house's roof before building its foundation!

```
Construction:  Base → Intermediate → Derived (Bottom-up)
Destruction:   Derived → Intermediate → Base (Top-down)
```

[↑ Back to Table of Contents](#table-of-contents)

---

<a id="default-constructor-behavior"></a>
## 2. Default Constructor Behavior

### Original Example

```cpp
#include <iostream>

class A {
public:
    A() : a(1) {
        std::cout << "A(): a = " << a << std::endl;
    }
    A(int a) : a(a) {
        std::cout << "A(int): a = " << a << std::endl;
    }
private:
    int a;
};

class B : public A {
public:
    B() : b(2) {
        std::cout << "B(): b = " << b << std::endl;
    }
    B(int b) : b(b) {
        std::cout << "B(int): b = " << b << std::endl;
    }
private:
    int b;
};

class C : public B {
public:
    C() : c(3) {
        std::cout << "C(): c = " << c << std::endl;
    }
    C(int c) : c(c) {
        std::cout << "C(int): c = " << c << std::endl;
    }
private:
    int c;
};

int main(int argc, char* argv[]) {
    std::cout << "Without parameter:" << std::endl;
    C c_obj{};
    std::cout << "\nWith parameter:" << std::endl;
    C c_obj_param{30};
    return 0;
}
```

### Output
```
Without parameter:
A(): a = 1
B(): b = 2
C(): c = 3

With parameter:
A(): a = 1
B(): b = 2
C(int): c = 30
```

### Key Observation

Notice that even when we call `C(int)` with a parameter, the base classes `A` and `B` still use their **default constructors**!

[↑ Back to Table of Contents](#table-of-contents)

---

<a id="execution-sequence-analysis"></a>
## 3. Execution Sequence Analysis

### Case 1: `C c_obj{};` (Default Constructor)

#### What the Compiler Sees:

```cpp
C() : c(3) {
    std::cout << "C(): c = " << c << std::endl;
}
```

#### What the Compiler Does (Implicit):

```cpp
C() : B(),     // ← Implicitly calls B's default constructor
      c(3) {
    std::cout << "C(): c = " << c << std::endl;
}
```

And `B()` does the same:

```cpp
B() : A(),     // ← Implicitly calls A's default constructor
      b(2) {
    std::cout << "B(): b = " << b << std::endl;
}
```

#### Execution Flow:

```
Step 1: C() constructor called
   │
   ├──> Step 2: Compiler sees no explicit base constructor call
   │            Automatically calls B() (default)
   │              │
   │              ├──> Step 3: B() constructor starts
   │              │           Compiler sees no explicit base constructor call
   │              │           Automatically calls A() (default)
   │              │              │
   │              │              ├──> Step 4: A() constructor starts
   │              │              │           Initializes: a = 1
   │              │              │           Prints: "A(): a = 1"
   │              │              └──> A() constructor completes
   │              │
   │              ├──> Step 5: B() constructor continues
   │              │           Initializes: b = 2
   │              │           Prints: "B(): b = 2"
   │              └──> B() constructor completes
   │
   ├──> Step 6: C() constructor continues
   │           Initializes: c = 3
   │           Prints: "C(): c = 3"
   └──> C() constructor completes
```

**Visual Timeline:**

```
Time →
[A() starts] → [a=1] → [Print "A()"] → [A() done]
                                          ↓
                       [B() starts] → [b=2] → [Print "B()"] → [B() done]
                                                                 ↓
                                              [C() starts] → [c=3] → [Print "C()"] → [C() done]
```

### Case 2: `C c_obj_param{30};` (Parameterized Constructor)

#### What the Compiler Sees:

```cpp
C(int c) : c(c) {
    std::cout << "C(int): c = " << c << std::endl;
}
```

#### What the Compiler Does (Implicit):

```cpp
C(int c) : B(),    // ← Still implicitly calls B's DEFAULT constructor!
           c(c) {
    std::cout << "C(int): c = " << c << std::endl;
}
```

#### Execution Flow:

```
Step 1: C(int) constructor called with c = 30
   │
   ├──> Step 2: Compiler sees no explicit base constructor call
   │            Automatically calls B() (default, not B(int)!)
   │              │
   │              ├──> Step 3: B() constructor starts
   │              │           Automatically calls A() (default)
   │              │              │
   │              │              ├──> Step 4: A() constructor
   │              │              │           Initializes: a = 1
   │              │              │           Prints: "A(): a = 1"
   │              │              └──> A() completes
   │              │
   │              ├──> Step 5: B() constructor continues
   │              │           Initializes: b = 2
   │              │           Prints: "B(): b = 2"
   │              └──> B() completes
   │
   ├──> Step 6: C(int) constructor continues
   │           Initializes: c = 30 (uses the parameter!)
   │           Prints: "C(int): c = 30"
   └──> C(int) completes
```

### Important Rule

> **If a derived class constructor doesn't EXPLICITLY call a base class constructor in its initializer list, the compiler AUTOMATICALLY calls the base class's DEFAULT constructor.**

This means:
- You wrote: `C(int c) : c(c) { }`
- Compiler executes: `C(int c) : B(), c(c) { }`
- `B()` then executes: `B() : A(), b(2) { }`

[↑ Back to Table of Contents](#table-of-contents)

---

<a id="calling-parameterized-base-constructors"></a>
## 4. Calling Parameterized Base Constructors

To use parameterized constructors of base classes, you must **explicitly call them** in the initializer list.

### Modified Code

```cpp
#include <iostream>

class A {
public:
    A() : a(1) {
        std::cout << "A(): a = " << a << std::endl;
    }
    A(int a) : a(a) {
        std::cout << "A(int): a = " << a << std::endl;
    }
private:
    int a;
};

class B : public A {
public:
    B() : A(), b(2) {  // Explicitly call A() (though it's implicit)
        std::cout << "B(): b = " << b << std::endl;
    }
    B(int b) : A(), b(b) {  // Explicitly call A()
        std::cout << "B(int): b = " << b << std::endl;
    }
    // New: Constructor that takes parameters for both B and A
    B(int a_val, int b_val) : A(a_val), b(b_val) {
        std::cout << "B(int, int): b = " << b << std::endl;
    }
private:
    int b;
};

class C : public B {
public:
    C() : B(), c(3) {  // Explicitly call B()
        std::cout << "C(): c = " << c << std::endl;
    }
    C(int c) : B(), c(c) {  // Explicitly call B()
        std::cout << "C(int): c = " << c << std::endl;
    }
    // New: Constructor that takes parameters for C and B
    C(int b_val, int c_val) : B(b_val), c(c_val) {
        std::cout << "C(int, int): c = " << c << std::endl;
    }
    // New: Constructor that takes parameters for all classes
    C(int a_val, int b_val, int c_val) : B(a_val, b_val), c(c_val) {
        std::cout << "C(int, int, int): c = " << c << std::endl;
    }
private:
    int c;
};

int main(int argc, char* argv[]) {
    std::cout << "=== Case 1: Default constructors ===" << std::endl;
    C obj1{};
    
    std::cout << "\n=== Case 2: Only C parameter ===" << std::endl;
    C obj2{30};
    
    std::cout << "\n=== Case 3: B and C parameters ===" << std::endl;
    C obj3{20, 30};
    
    std::cout << "\n=== Case 4: A, B, and C parameters ===" << std::endl;
    C obj4{10, 20, 30};
    
    return 0;
}
```

### Output

```
=== Case 1: Default constructors ===
A(): a = 1
B(): b = 2
C(): c = 3

=== Case 2: Only C parameter ===
A(): a = 1
B(): b = 2
C(int): c = 30

=== Case 3: B and C parameters ===
A(): a = 1
B(int): b = 20
C(int, int): c = 30

=== Case 4: A, B, and C parameters ===
A(int): a = 10
B(int, int): b = 20
C(int, int, int): c = 30
```

[↑ Back to Table of Contents](#table-of-contents)

---

<a id="complete-example-with-explanation"></a>
## 5. Complete Example with Explanation

### Detailed Analysis of Each Case

#### Case 1: `C obj1{};` (All Default Constructors)

```cpp
C() : B(), c(3) {
    std::cout << "C(): c = " << c << std::endl;
}
```

**Execution:**
```
A() called → a = 1
B() called → b = 2
C() called → c = 3
```

**Explanation:**
- `C()` calls `B()` (explicit in modified code, implicit in original)
- `B()` calls `A()` (explicit in modified code, implicit in original)
- Each constructor uses default values

---

#### Case 2: `C obj2{30};` (Only C Gets Parameter)

```cpp
C(int c) : B(), c(c) {
    std::cout << "C(int): c = " << c << std::endl;
}
```

**Execution:**
```
A() called → a = 1
B() called → b = 2
C(int) called → c = 30
```

**Explanation:**
- `C(int)` explicitly calls `B()` (default constructor)
- `B()` implicitly calls `A()` (default constructor)
- Only `c` gets the parameter value
- `a` and `b` still use defaults

**Key Point:** Passing a parameter to C doesn't automatically pass it to B or A!

---

#### Case 3: `C obj3{20, 30};` (B and C Get Parameters)

```cpp
C(int b_val, int c_val) : B(b_val), c(c_val) {
    std::cout << "C(int, int): c = " << c << std::endl;
}
```

Which calls:

```cpp
B(int b) : A(), b(b) {
    std::cout << "B(int): b = " << b << std::endl;
}
```

**Execution:**
```
A() called → a = 1
B(int) called → b = 20
C(int, int) called → c = 30
```

**Explanation:**
- `C(int, int)` explicitly calls `B(int)` with `b_val = 20`
- `B(int)` implicitly calls `A()` (default constructor)
- `a` still uses default, but `b` and `c` get parameters

---

#### Case 4: `C obj4{10, 20, 30};` (All Get Parameters)

```cpp
C(int a_val, int b_val, int c_val) : B(a_val, b_val), c(c_val) {
    std::cout << "C(int, int, int): c = " << c << std::endl;
}
```

Which calls:

```cpp
B(int a_val, int b_val) : A(a_val), b(b_val) {
    std::cout << "B(int, int): b = " << b << std::endl;
}
```

Which calls:

```cpp
A(int a) : a(a) {
    std::cout << "A(int): a = " << a << std::endl;
}
```

**Execution:**
```
A(int) called → a = 10
B(int, int) called → b = 20
C(int, int, int) called → c = 30
```

**Explanation:**
- `C(int, int, int)` explicitly calls `B(int, int)` with `a_val = 10, b_val = 20`
- `B(int, int)` explicitly calls `A(int)` with `a_val = 10`
- All classes get their respective parameter values

**This is the proper way to initialize the entire hierarchy with custom values!**

---

### Visual Representation of Constructor Calls

```
Case 1: C obj1{}
   C() 
    └─> B()
         └─> A()
              └─> a=1
         └─> b=2
    └─> c=3

Case 2: C obj2{30}
   C(int)  [param: 30]
    └─> B()
         └─> A()
              └─> a=1
         └─> b=2
    └─> c=30  ← Uses parameter

Case 3: C obj3{20, 30}
   C(int, int)  [params: 20, 30]
    └─> B(int)  [param: 20]
         └─> A()
              └─> a=1
         └─> b=20  ← Uses parameter
    └─> c=30  ← Uses parameter

Case 4: C obj4{10, 20, 30}
   C(int, int, int)  [params: 10, 20, 30]
    └─> B(int, int)  [params: 10, 20]
         └─> A(int)  [param: 10]
              └─> a=10  ← Uses parameter
         └─> b=20  ← Uses parameter
    └─> c=30  ← Uses parameter
```

---

## Key Takeaways

### 1. Automatic Default Constructor Call
- If you don't explicitly call a base class constructor, the compiler calls the **default constructor** automatically
- This happens even if you call a parameterized constructor of the derived class

### 2. Explicit Base Constructor Call
- To use a parameterized base constructor, you MUST explicitly call it in the initializer list:
  ```cpp
  DerivedClass(params) : BaseClass(params), members(values) {
      // constructor body
  }
  ```

### 3. Constructor Execution Order
- **Always** executes from base to derived (top-down in hierarchy)
- Base class is fully constructed before derived class constructor body runs

### 4. Passing Parameters Up the Hierarchy
- Parameters don't automatically propagate to base classes
- You must explicitly pass them through constructor calls:
  ```cpp
  C(int a, int b, int c) : B(a, b), c(c) { }
  ```

### 5. Initializer List Order
- Base class constructors are called **before** member initialization
- Even if you write members first in the list:
  ```cpp
  C() : c(3), B() { }  // B() still called before c initialization
  ```

### Best Practice

✓ **DO:**
- Explicitly call base constructors when you need specific initialization
- Pass parameters through the hierarchy when needed
- Use initializer lists for all initialization

✗ **DON'T:**
- Rely on implicit default constructor calls when you need specific values
- Try to initialize base class members in derived class constructor body
- Forget that base constructors run first

---

## Summary

**Constructor execution in inheritance** follows a strict order:
1. Base class constructor (outermost first)
2. Member variable initialization
3. Constructor body execution

**If not explicitly called**, the compiler automatically invokes the **default constructor** of the base class. To use parameterized base constructors, you must explicitly call them in the initializer list.

This ensures that the base class is fully constructed before the derived class tries to use it, maintaining the integrity of the inheritance hierarchy.

**C++11 introduced constructor inheritance** using the `using` keyword, which allows derived classes to inherit base class constructors, reducing boilerplate code. However, there are important limitations when constructors with the same signature exist in both base and derived classes.

[↑ Back to Table of Contents](#table-of-contents)

---

<a id="inheriting-constructors-cpp11"></a>
## 6. Inheriting Constructors (C++11)

### The Problem Before C++11

Before C++11, if you wanted to use base class constructors in a derived class, you had to write forwarding constructors manually:

```cpp
class Base {
public:
    Base(int x) { }
    Base(int x, int y) { }
    Base(int x, int y, int z) { }
};

class Derived : public Base {
public:
    // Manually forward each constructor - tedious!
    Derived(int x) : Base(x) { }
    Derived(int x, int y) : Base(x, y) { }
    Derived(int x, int y, int z) : Base(x, y, z) { }
};
```

**Problems:**
- Lots of boilerplate code
- Error-prone (easy to forget a constructor)
- Hard to maintain (every base constructor needs forwarding)
- Repetitive and tedious

### The Solution: `using` to Inherit Constructors (C++11)

C++11 introduced the `using` declaration to inherit base class constructors:

```cpp
class Base {
public:
    Base(int x) { std::cout << "Base(int): " << x << "\n"; }
    Base(int x, int y) { std::cout << "Base(int, int): " << x << ", " << y << "\n"; }
    Base(int x, int y, int z) { std::cout << "Base(int, int, int)\n"; }
};

class Derived : public Base {
public:
    using Base::Base;  // ✓ Inherit ALL base constructors!
};

int main() {
    Derived d1(10);           // Calls Base(int)
    Derived d2(10, 20);       // Calls Base(int, int)
    Derived d3(10, 20, 30);   // Calls Base(int, int, int)
}
```

### Output
```
Base(int): 10
Base(int, int): 10, 20
Base(int, int, int)
```

### How It Eases Development

#### Before C++11 (Manual Forwarding)
```cpp
class Base {
public:
    Base() { }
    Base(int x) { }
    Base(int x, double y) { }
    Base(std::string s) { }
};

class Derived : public Base {
    int member;
public:
    // Must manually write ALL of these!
    Derived() : Base(), member(0) { }
    Derived(int x) : Base(x), member(0) { }
    Derived(int x, double y) : Base(x, y), member(0) { }
    Derived(std::string s) : Base(s), member(0) { }
};
```

#### After C++11 (Inheriting Constructors)
```cpp
class Base {
public:
    Base() { }
    Base(int x) { }
    Base(int x, double y) { }
    Base(std::string s) { }
};

class Derived : public Base {
    int member = 0;  // Default member initialization
public:
    using Base::Base;  // ✓ One line instead of four constructors!
};
```

**Benefits:**
- ✓ **Less code** - One line vs multiple constructors
- ✓ **Less maintenance** - Add base constructor, automatically available
- ✓ **Fewer errors** - No chance of forgetting to forward a constructor
- ✓ **Cleaner code** - Intent is clear and concise

### Complete Example

```cpp
#include <iostream>
#include <string>

class Person {
protected:
    std::string name;
    int age;
    
public:
    Person(std::string n) : name(n), age(0) {
        std::cout << "Person(string): " << name << "\n";
    }
    
    Person(std::string n, int a) : name(n), age(a) {
        std::cout << "Person(string, int): " << name << ", " << age << "\n";
    }
    
    void display() const {
        std::cout << "Name: " << name << ", Age: " << age << "\n";
    }
};

class Employee : public Person {
    int employeeId = 0;  // Default member initialization
    
public:
    // Inherit all Person constructors
    using Person::Person;
    
    // Can still add derived-specific constructors
    Employee(std::string n, int a, int id) : Person(n, a), employeeId(id) {
        std::cout << "Employee(string, int, int): " << name << ", " << age << ", " << id << "\n";
    }
    
    void display() const {
        Person::display();
        std::cout << "Employee ID: " << employeeId << "\n";
    }
};

int main() {
    std::cout << "=== Using inherited constructor ===" << std::endl;
    Employee emp1("Alice", 30);
    emp1.display();
    
    std::cout << "\n=== Using derived-specific constructor ===" << std::endl;
    Employee emp2("Bob", 25, 1001);
    emp2.display();
    
    return 0;
}
```

### Output
```
=== Using inherited constructor ===
Person(string, int): Alice, 30
Name: Alice, Age: 30
Employee ID: 0

=== Using derived-specific constructor ===
Person(string, int): Bob, 25
Employee(string, int, int): Bob, 25, 1001
Name: Bob, Age: 25
Employee ID: 1001
```

[↑ Back to Table of Contents](#table-of-contents)

---

<a id="limitations-of-inherited-constructors"></a>
## 7. Limitations of Inherited Constructors

### Limitation 1: Constructor Hiding (Same Signature Conflict)

**Important Rule:** If a derived class defines a constructor with the **same signature** as an inherited base constructor, the derived class constructor **hides** (overrides) the inherited one.

#### Example: Constructor Hiding

```cpp
#include <iostream>

class Base {
public:
    Base(int x) {
        std::cout << "Base(int): " << x << "\n";
    }
    
    Base(int x, int y) {
        std::cout << "Base(int, int): " << x << ", " << y << "\n";
    }
};

class Derived : public Base {
public:
    using Base::Base;  // Inherit all Base constructors
    
    // This HIDES the inherited Base(int) constructor!
    Derived(int x) {
        std::cout << "Derived(int): " << x << "\n";
    }
};

int main() {
    Derived d1(10);        // Calls Derived(int), NOT Base(int)
    Derived d2(10, 20);    // Calls inherited Base(int, int)
    
    return 0;
}
```

### Output
```
Derived(int): 10
Base(int, int): 10, 20
```

### Analysis

```
using Base::Base;  // Brings in:
                   // - Base(int)        ← HIDDEN by Derived(int)
                   // - Base(int, int)   ← Still available

Derived(int x) { }  // This HIDES Base(int)
```

**What Happens:**
1. `Derived d1(10)` - Calls `Derived(int)`, **not** the inherited `Base(int)`
2. `Derived d2(10, 20)` - Calls inherited `Base(int, int)` (no conflict)

**Key Point:** The derived class constructor with matching signature takes precedence and completely hides the inherited base constructor.

### Detailed Example with Multiple Scenarios

```cpp
#include <iostream>

class Base {
public:
    Base() {
        std::cout << "Base()\n";
    }
    
    Base(int x) {
        std::cout << "Base(int): " << x << "\n";
    }
    
    Base(int x, int y) {
        std::cout << "Base(int, int): " << x << ", " << y << "\n";
    }
    
    Base(double d) {
        std::cout << "Base(double): " << d << "\n";
    }
};

class Derived : public Base {
    int member;
    
public:
    using Base::Base;  // Inherit ALL Base constructors
    
    // Scenario 1: Same signature - HIDES Base(int)
    Derived(int x) : Base(x * 2), member(x) {
        std::cout << "Derived(int): " << x << ", member = " << member << "\n";
    }
    
    // Scenario 2: Different signature - coexists with inherited constructors
    Derived(int x, int y, int z) : Base(x, y), member(z) {
        std::cout << "Derived(int, int, int): member = " << z << "\n";
    }
};

int main() {
    std::cout << "=== Test 1: Derived(int) - Hidden ===" << std::endl;
    Derived d1(5);  // Calls Derived(int), Base(int) is hidden
    
    std::cout << "\n=== Test 2: Base(int, int) - Inherited ===" << std::endl;
    Derived d2(10, 20);  // Calls inherited Base(int, int)
    
    std::cout << "\n=== Test 3: Base(double) - Inherited ===" << std::endl;
    Derived d3(3.14);  // Calls inherited Base(double)
    
    std::cout << "\n=== Test 4: Derived(int, int, int) - Derived-specific ===" << std::endl;
    Derived d4(1, 2, 3);  // Calls Derived(int, int, int)
    
    std::cout << "\n=== Test 5: Base() - Inherited ===" << std::endl;
    Derived d5;  // Calls inherited Base()
    
    return 0;
}
```

### Output
```
=== Test 1: Derived(int) - Hidden ===
Base(int): 10
Derived(int): 5, member = 5

=== Test 2: Base(int, int) - Inherited ===
Base(int, int): 10, 20

=== Test 3: Base(double) - Inherited ===
Base(double): 3.14

=== Test 4: Derived(int, int, int) - Derived-specific ===
Base(int, int): 1, 2
Derived(int, int, int): member = 3

=== Test 5: Base() - Inherited ===
Base()
```

### Analysis of Each Test Case

| Test | Constructor Called | Explanation |
|------|-------------------|-------------|
| `Derived d1(5)` | `Derived(int)` | Derived class has `Derived(int)` which **hides** inherited `Base(int)` |
| `Derived d2(10, 20)` | Inherited `Base(int, int)` | No conflict, uses inherited constructor |
| `Derived d3(3.14)` | Inherited `Base(double)` | No conflict, uses inherited constructor |
| `Derived d4(1, 2, 3)` | `Derived(int, int, int)` | Derived-specific constructor (not inherited) |
| `Derived d5` | Inherited `Base()` | No conflict, uses inherited constructor |

### Limitation 2: Cannot Inherit from Multiple Bases with Same Signature

If multiple base classes have constructors with the same signature, you cannot inherit them:

```cpp
class Base1 {
public:
    Base1(int x) { }
};

class Base2 {
public:
    Base2(int x) { }
};

class Derived : public Base1, public Base2 {
public:
    using Base1::Base1;  // Brings Base1(int)
    using Base2::Base2;  // ERROR: Ambiguous - both have (int)
};
```

**Solution:** Define your own constructor to resolve ambiguity:

```cpp
class Derived : public Base1, public Base2 {
public:
    Derived(int x) : Base1(x), Base2(x) { }
};
```

### Limitation 3: Private and Protected Constructors

Inherited constructors maintain their access level:

```cpp
class Base {
protected:
    Base(int x) { }  // Protected constructor
};

class Derived : public Base {
public:
    using Base::Base;  // Base(int) is still PROTECTED in Derived
};

int main() {
    // Derived d(10);  // ERROR: Base(int) is protected
}
```

### Limitation 4: Default Member Initialization

When using inherited constructors, derived class members must use **default member initialization**:

```cpp
class Base {
public:
    Base(int x) { }
};

class Derived : public Base {
    int member;  // Uninitialized when using inherited constructors!
    
public:
    using Base::Base;
};

// Better:
class Derived : public Base {
    int member = 0;  // ✓ Default member initialization
    
public:
    using Base::Base;
};
```

### When NOT to Use Inherited Constructors

**Don't use inherited constructors when:**
- Derived class needs to initialize its own members in specific ways
- You need different behavior than just forwarding to base
- Multiple bases have constructors with same signature
- You need to perform additional initialization logic

✓ **DO use inherited constructors when:**
- Derived class doesn't add new data members (or they have defaults)
- You simply want to forward all base constructors
- No special initialization logic is needed
- You want to reduce boilerplate code

### Best Practices Summary

```cpp
class Base {
public:
    Base(int x) { }
    Base(int x, int y) { }
};

// ✓ GOOD: Simple forwarding, members have defaults
class Derived1 : public Base {
    int member = 0;
public:
    using Base::Base;  // Clean and simple
};

// ✓ GOOD: Mix inherited and custom constructors
class Derived2 : public Base {
    int member = 0;
public:
    using Base::Base;  // Inherit most constructors
    
    // Add custom constructor when needed
    Derived2(int x, int y, int z) : Base(x, y), member(z) { }
};

// ✓ GOOD: Override when you need different behavior
class Derived3 : public Base {
    int member;
public:
    using Base::Base;  // Inherit Base(int, int)
    
    // Override Base(int) with custom behavior
    Derived3(int x) : Base(x * 2), member(x) { }
};

// BAD: Inherited constructors can't initialize this properly
class Derived4 : public Base {
    int member;  // No default, will be uninitialized!
public:
    using Base::Base;  // member not initialized
};
```

[↑ Back to Table of Contents](#table-of-contents)

<a id="destructors-order-inheritance"></a>
## 7.Understanding Destructor Execution Order

When an object of a **derived class** is destroyed, destructors are called in the **reverse order of construction**.

### Order of Destruction:
1. **Derived class destructor** — called **first**  
2. **Base class destructor** — called **last**

This ensures that the derived class cleans up its resources before the base class is destroyed.

---

### Example

```cpp
#include <iostream>

class Parent {
public:
    Parent() { std::cout << "Inside base class constructor\n"; }
    ~Parent() { std::cout << "Inside base class destructor\n"; }
};

class Child : public Parent {
public:
    Child() { std::cout << "Inside derived class constructor\n"; }
    ~Child() { std::cout << "Inside derived class destructor\n"; }
};

int main() {
    Child obj;
    return 0;
}
```

---

### Expected Output

```
Inside base class constructor
Inside derived class constructor
Inside derived class destructor
Inside base class destructor
```

---

### Why Destructors Are Called in Reverse Order

- During **construction**, the base class is created **first**, forming a foundation for the derived class.  
- During **destruction**, the **derived destructor** runs first to clean up resources that might depend on the base class still being valid.  
- After that, the **base class destructor** runs to finalize the cleanup.

This reverse order:
- Prevents undefined behavior caused by destroying the base while derived resources still exist.  
- Maintains **symmetry and safety** — the base’s lifetime always outlasts the derived part.  
- Applies similarly to **data members**, which are also destroyed in the reverse order of their construction.

---

[↑ Back to Table of Contents](#table-of-contents)


---

## Complete Summary

### Constructor Execution Rules

1. **Execution Order**: Base → Derived (construction), Derived → Base (destruction)
2. **Default Constructor**: Automatically called if not explicitly specified
3. **Explicit Calls**: Use initializer list to call specific base constructors
4. **C++11 Inheritance**: Use `using Base::Base;` to inherit all base constructors

### Inheriting Constructors (C++11)

**Advantages:**
- ✓ Reduces boilerplate code
- ✓ Automatic forwarding of base constructors
- ✓ Easier maintenance
- ✓ Less error-prone

**Limitations:**
- Same signature in derived class hides inherited constructor
- Cannot inherit from multiple bases with same signature
- Access levels are preserved
- Derived members need default initialization

**Golden Rule:** Inherited constructors are a convenience feature for simple cases. When you need custom initialization logic, write explicit constructors.

### Destructor execution order
The **reverse order of destructor calls** ensures:
- Consistent and safe cleanup  
- Proper handling of dependencies  
- No premature destruction of essential components  

In short, **destruction happens bottom-up**, mirroring the **top-down** order of construction.

[↑ Back to Table of Contents](#table-of-contents)
