# Class Templates

## Table of Contents

1. [What is a Class Template?](#1-what-is-a-class-template)
2. [Syntax for Class Templates](#2-syntax-for-class-templates)
3. [A Simple Class Template Example](#3-a-simple-class-template-example)
4. [Defining Member Functions Outside the Class](#4-defining-member-functions-outside-the-class)
5. [Instantiating Class Templates](#5-instantiating-class-templates)
6. [Understanding Template vs Type](#6-understanding-template-vs-type)

---

## 1. What is a Class Template?

A **class template** is a blueprint for creating classes that work with **generic types**. Instead of writing separate classes for `int`, `double`, `string`, etc., you write one template that works with any type.

### Definition

> A class template is a class that is **parameterized** over one or more types. It contains member variables and functions that use these generic types.

### Why Use Class Templates?

**Without templates:**
```cpp
class IntBox {
    int value;
};

class DoubleBox {
    double value;
};

class StringBox {
    string value;
};
// ... and so on for every type!
```

**With templates:**
```cpp
template<typename T>
class Box {
    T value;  // Works with ANY type!
};
```

[↑ Back to Table of Contents](#table-of-contents)

---

## 2. Syntax for Class Templates

### Basic Syntax

```cpp
template<typename T>
class ClassName {
    // Member variables using type T
    T member;
    
    // Member functions using type T
    void setMember(T value);
    T getMember();
};
```

### Syntax Breakdown

```cpp
template<typename T>
│         │        │
│         │        └─→ Template parameter name (can be any identifier)
│         └─────────→ Keyword (can also use 'class' instead)
└──────────────────→ Keyword introducing template
```

### Multiple Template Parameters

```cpp
template<typename T, typename U>
class Pair {
    T first;
    U second;
};

template<typename T, int SIZE>
class Array {
    T data[SIZE];  // SIZE is a non-type parameter
};
```

### Common Conventions

| Convention | Example | Notes |
|------------|---------|-------|
| Single letter | `template<typename T>` | Most common for simple cases |
| Descriptive name | `template<typename ValueType>` | Better for complex templates |
| Multiple params | `template<typename K, typename V>` | Key-Value pairs |

[↑ Back to Table of Contents](#table-of-contents)

---

## 3. A Simple Class Template Example

Let's create a `Box` class that can hold any type of value.

### Simple Box Template

```cpp
template<typename T>
class Box {
private:
    T value;  // Generic type member variable
    
public:
    // Constructor
    Box(T val) : value(val) {}
    
    // Getter
    T getValue() const {
        return value;
    }
    
    // Setter
    void setValue(T val) {
        value = val;
    }
};
```

### Understanding the Example

```cpp
template<typename T>  // ← Declares this is a template
class Box {
private:
    T value;          // ← T can be int, double, string, etc.
    
public:
    Box(T val) : value(val) {}  // ← Constructor takes type T
    
    T getValue() const {         // ← Returns type T
        return value;
    }
    
    void setValue(T val) {       // ← Parameter is type T
        value = val;
    }
};
```

### Usage Example

```cpp
// Create a Box for integers
Box<int> intBox(42);
cout << intBox.getValue();  // Output: 42

// Create a Box for doubles
Box<double> doubleBox(3.14);
cout << doubleBox.getValue();  // Output: 3.14

// Create a Box for strings
Box<string> stringBox("Hello");
cout << stringBox.getValue();  // Output: Hello
```

[↑ Back to Table of Contents](#table-of-contents)

---

## 4. Defining Member Functions Outside the Class

For better code organization, you can define member functions outside the class body.

### Syntax for External Definition

```cpp
template<typename T>
ReturnType ClassName<T>::functionName(parameters) {
    // Function body
}
```

### Complete Example: Box with External Definitions

```cpp
// ============================================
// Class Declaration
// ============================================
template<typename T>
class Box {
private:
    T value;
    bool isEmpty;
    
public:
    // Constructor declarations
    Box();
    Box(T val);
    
    // Member function declarations
    void store(T val);
    T retrieve() const;
    bool empty() const;
    void clear();
    void display() const;
};

// ============================================
// Member Function Definitions (Outside Class)
// ============================================

// Default constructor
template<typename T>
Box<T>::Box() : isEmpty(true) {
    // Empty constructor body
}

// Parameterized constructor
template<typename T>
Box<T>::Box(T val) : value(val), isEmpty(false) {
    // Initialize with a value
}

// Store function
template<typename T>
void Box<T>::store(T val) {
    value = val;
    isEmpty = false;
}

// Retrieve function
template<typename T>
T Box<T>::retrieve() const {
    if (isEmpty) {
        throw runtime_error("Box is empty!");
    }
    return value;
}

// Empty check function
template<typename T>
bool Box<T>::empty() const {
    return isEmpty;
}

// Clear function
template<typename T>
void Box<T>::clear() {
    isEmpty = true;
}

// Display function
template<typename T>
void Box<T>::display() const {
    if (isEmpty) {
        cout << "Box is empty" << endl;
    } else {
        cout << "Box contains: " << value << endl;
    }
}
```

### Anatomy of External Definition

```cpp
template<typename T>        // ← Template declaration (required!)
│
└─→ T Box<T>::retrieve() const {
      │   │  │
      │   │  └─→ Scope resolution with template parameter
      │   └────→ Class name
      └────────→ Return type using template parameter
```

### Key Points for External Definitions

**Must include:**
- `template<typename T>` before each function
- Class name with template parameter: `Box<T>::`
- Same signature as declaration

**Common mistakes:**
```cpp
// WRONG: Missing template declaration
T Box<T>::retrieve() const { }

// WRONG: Missing template parameter on class name
template<typename T>
T Box::retrieve() const { }

// CORRECT
template<typename T>
T Box<T>::retrieve() const { }
```

[↑ Back to Table of Contents](#table-of-contents)

---

## 5. Instantiating Class Templates

### Basic Instantiation

```cpp
// Syntax: ClassName<Type> objectName;
Box<int> integerBox;           // Box that holds int
Box<double> doubleBox;         // Box that holds double
Box<string> stringBox;         // Box that holds string
```

### Instantiation with Initialization

```cpp
// Using parameterized constructor
Box<int> box1(42);
Box<double> box2(3.14159);
Box<string> box3("Hello, Templates!");

// Using default constructor then storing
Box<char> box4;
box4.store('A');
```

### Complete Usage Example

```cpp
#include <iostream>
#include <string>
using namespace std;

// ... (Box class template definition here) ...

int main() {
    // ========================================
    // Example 1: Integer Box
    // ========================================
    cout << "=== Integer Box ===" << endl;
    Box<int> intBox(100);
    intBox.display();                    // Box contains: 100
    
    int value = intBox.retrieve();
    cout << "Value: " << value << endl;  // Value: 100
    
    intBox.clear();
    intBox.display();                    // Box is empty
    
    // ========================================
    // Example 2: Double Box
    // ========================================
    cout << "\n=== Double Box ===" << endl;
    Box<double> doubleBox;
    cout << "Empty? " << doubleBox.empty() << endl;  // Empty? 1
    
    doubleBox.store(2.71828);
    doubleBox.display();                 // Box contains: 2.71828
    
    // ========================================
    // Example 3: String Box
    // ========================================
    cout << "\n=== String Box ===" << endl;
    Box<string> stringBox("C++ Templates");
    stringBox.display();                 // Box contains: C++ Templates
    
    string text = stringBox.retrieve();
    cout << "Retrieved: " << text << endl;  // Retrieved: C++ Templates
    
    // ========================================
    // Example 4: Custom Type
    // ========================================
    struct Point {
        int x, y;
        friend ostream& operator<<(ostream& os, const Point& p) {
            return os << "(" << p.x << ", " << p.y << ")";
        }
    };
    
    cout << "\n=== Point Box ===" << endl;
    Box<Point> pointBox({10, 20});
    pointBox.display();                  // Box contains: (10, 20)
    
    return 0;
}
```

### Output

```
=== Integer Box ===
Box contains: 100
Value: 100
Box is empty

=== Double Box ===
Empty? 1
Box contains: 2.71828

=== String Box ===
Box contains: C++ Templates
Retrieved: C++ Templates

=== Point Box ===
Box contains: (10, 20)
```

### Multiple Objects of Same Type

```cpp
// You can create multiple objects with the same template type
Box<int> scores[3];
scores[0].store(85);
scores[1].store(92);
scores[2].store(78);

for (int i = 0; i < 3; i++) {
    cout << "Score " << i << ": " << scores[i].retrieve() << endl;
}
```

[↑ Back to Table of Contents](#table-of-contents)

---

## 6. Understanding Template vs Type

Now that you've seen how to create and use class templates, it's important to understand the distinction between a **template** and a **type**.

### Template vs Type Table

| Concept | Code Example | Description |
|---------|--------------|-------------|
| **Template** | `template<typename T> class Box` | The generic blueprint/pattern with parameter `T` |
| **Type** | `Box<int>` | A specific instantiation of the template with `T = int` |
| **Type** | `Box<double>` | A specific instantiation of the template with `T = double` |
| **Type** | `Box<string>` | A specific instantiation of the template with `T = string` |

### Key Point

- `Box` (with template parameter) is **not a type** — it's a template
- `Box<int>`, `Box<double>`, etc. **are types** — they are instantiated from the template

```cpp
// This is the TEMPLATE (not a type)
template<typename T>
class Box { /* ... */ };

// These are TYPES (instantiated from the template)
Box<int> myIntBox;        // Box<int> is a type
Box<double> myDoubleBox;  // Box<double> is a type
Box<string> myStringBox;  // Box<string> is a type

// Each type is distinct and independent
```

### Visual Representation

```
                Template
                   │
        template<typename T>
             class Box
                   │
         ┌─────────┼─────────┐
         │         │         │
         ▼         ▼         ▼
    Box<int>  Box<double> Box<string>
      Type      Type        Type
```

### Why This Matters

Understanding this distinction is crucial because:

1. **You cannot declare a variable of type `Box`** — you must specify the type parameter
   ```cpp
   Box myBox;        // ERROR: Template parameter missing
   Box<int> myBox;   // CORRECT: Specific type
   ```

2. **Each instantiated type is independent**
   ```cpp
   Box<int> intBox;
   Box<double> doubleBox;
   // These are completely different types!
   // You cannot assign one to the other
   ```

3. **The compiler generates separate code for each type**
   ```cpp
   Box<int> b1;      // Generates Box code for int
   Box<double> b2;   // Generates Box code for double
   // Two separate classes in the compiled code
   ```

[↑ Back to Table of Contents](#table-of-contents)

---

## Summary

### What You've Learned

1. **Class templates** are blueprints for creating generic classes
2. **Syntax**: `template<typename T>` before the class declaration
3. **Member functions** can use the template parameter `T`
4. **External definitions** require `template<typename T>` and `ClassName<T>::`
5. **Instantiation**: `ClassName<Type> object;`
6. The **template** (`Box`) is different from instantiated **types** (`Box<int>`, `Box<double>`)

### Quick Reference

```cpp
// Declaration
template<typename T>
class Container {
    T data;
public:
    Container(T val);
    void set(T val);
    T get() const;
};

// External definition
template<typename T>
Container<T>::Container(T val) : data(val) {}

template<typename T>
void Container<T>::set(T val) { data = val; }

template<typename T>
T Container<T>::get() const { return data; }

// Usage
Container<int> c1(42);
Container<string> c2("Hello");
```

[↑ Back to Table of Contents](#table-of-contents)
