# Operator Overloading

## Table of Contents
- [What is Operator Overloading?](#what-is-operator-overloading)
- [Overloadable vs Non-Overloadable Operators](#overloadable-vs-non-overloadable-operators)
- [Why Use Operator Overloading?](#why-use-operator-overloading)
- [Ways to Overload Operators](#ways-to-overload-operators)
  - [Member Function Overloading](#member-function-overloading)
  - [Non-Member Function Overloading](#non-member-function-overloading)
- [Binary vs Unary Operators](#binary-vs-unary-operators)
- [More Operator Overloading Examples](#more-operator-overloading-examples)
- [Best Practices](#best-practices)

---

## What is Operator Overloading?

Operator overloading is a feature in C++ that allows you to define custom behavior for operators (like `+`, `-`, `<`, `<<`, etc.) when they are used with your own classes. This tells C++ what it means to use an operator on a class you've written yourself.

### Basic Syntax

There are two ways to overload an operator:

#### 1. As a Member Function
```cpp
ReturnType operator@(parameters) const;
```

#### 2. As a Non-Member Function
```cpp
ReturnType operator@(ClassName& lhs, ClassName& rhs);
```

Where `@` represents the operator you want to overload (e.g., `+`, `-`, `<`, `==`, etc.)

### Example Prototype

```cpp
class MyClass {
public:
    // Member function operator overloading
    MyClass operator+(const MyClass& other) const;
    bool operator<(const MyClass& other) const;
    MyClass& operator=(const MyClass& other);
};

// Non-member function operator overloading
MyClass operator*(const MyClass& lhs, const MyClass& rhs);
ostream& operator<<(ostream& out, const MyClass& obj);
```

### Overloadable Operators

C++ allows you to overload most operators:
- Arithmetic: `+`, `-`, `*`, `/`, `%`
- Comparison: `<`, `>`, `<=`, `>=`, `==`, `!=`
- Logical: `&&`, `||`, `!`
- Assignment: `=`, `+=`, `-=`, `*=`, `/=`
- Stream: `<<`, `>>`
- And many more!

[↑ Back to Table of Contents](#table-of-contents)

---

## Overloadable vs Non-Overloadable Operators

Not all operators in C++ can be overloaded. Here's a comprehensive table:

### Operators That CAN Be Overloaded

| Category | Operators |
|----------|-----------|
| **Arithmetic** | `+` `-` `*` `/` `%` |
| **Bitwise** | `^` `&` `|` `~` `<<` `>>` |
| **Comparison** | `<` `>` `<=` `>=` `==` `!=` |
| **Logical** | `!` `&&` `||` |
| **Assignment** | `=` `+=` `-=` `*=` `/=` `%=` `^=` `&=` `|=` `<<=` `>>=` |
| **Increment/Decrement** | `++` `--` |
| **Member Access** | `->` `->*` |
| **Subscript** | `[]` |
| **Function Call** | `()` |
| **Memory Management** | `new` `new[]` `delete` `delete[]` |
| **Other** | `,` (comma operator) |

### Operators That CANNOT Be Overloaded

| Operator | Name | Reason |
|----------|------|--------|
| `::` | Scope resolution | Fundamental to C++ structure |
| `.` | Member access | Direct member access must remain fixed |
| `.*` | Pointer-to-member access | Core language feature |
| `?:` | Ternary conditional | Requires special evaluation rules |
| `sizeof` | Size-of operator | Compile-time operator |
| `typeid` | Type identification | RTTI operator |
| `#` | Preprocessor stringification | Preprocessor directive |
| `##` | Preprocessor concatenation | Preprocessor directive |

[↑ Back to Table of Contents](#table-of-contents)

---

## Why Use Operator Overloading?

Let's say we have a simple `Number` class that wraps an integer value. Without operator overloading, adding two numbers looks awkward:

### Without Operator Overloading

```cpp
class Number {
public:
    Number(int val) : value(val) {}
    
    Number add(const Number& other) const {
        return Number(value + other.value);
    }
    
    int getValue() const { return value; }
    
private:
    int value;
};

// Usage
Number a(5);
Number b(3);
Number c = a.add(b);  // Awkward syntax
cout << c.getValue() << endl;  // Output: 8
```

### With Operator Overloading

```cpp
class Number {
public:
    Number(int val) : value(val) {}
    
    Number operator+(const Number& other) const {
        return Number(value + other.value);
    }
    
    int getValue() const { return value; }
    
private:
    int value;
};

// Usage
Number a(5);
Number b(3);
Number c = a + b;  // Natural and intuitive!
cout << c.getValue() << endl;  // Output: 8
```

**Benefits:**
- More intuitive and readable code
- Makes custom classes behave like built-in types
- Follows the principle of least surprise for users of your class

[↑ Back to Table of Contents](#table-of-contents)

---

## Ways to Overload Operators

There are two primary ways to overload operators in C++:

### Member Function Overloading

When you overload an operator as a member function, it's declared inside the class. The left-hand side of the operation becomes `this`, and the right-hand side is passed as a parameter.

#### Syntax

```cpp
class ClassName {
public:
    ReturnType operator@(const ClassName& rhs) const;
};
```

Where `@` is the operator you want to overload (e.g., `+`, `-`, `<`, etc.)

#### Example: Time Class

```cpp
class Time {
public:
    Time(int h, int m, int s) : hours(h), minutes(m), seconds(s) {}
    
    // Overload < operator as a member function
    bool operator<(const Time& rhs) const {
        if (hours < rhs.hours) return true;
        if (rhs.hours < hours) return false;
        
        if (minutes < rhs.minutes) return true;
        if (rhs.minutes < minutes) return false;
        
        return seconds < rhs.seconds;
    }
    
    // Overload + operator as a member function
    Time operator+(const Time& rhs) const {
        int totalSeconds = seconds + rhs.seconds;
        int totalMinutes = minutes + rhs.minutes + totalSeconds / 60;
        int totalHours = hours + rhs.hours + totalMinutes / 60;
        
        return Time(totalHours % 24, totalMinutes % 60, totalSeconds % 60);
    }
    
private:
    int hours;
    int minutes;
    int seconds;
};

// Usage
Time morning(9, 30, 0);
Time duration(2, 45, 30);

if (morning < duration) {
    cout << "Morning comes before duration" << endl;
}

Time result = morning + duration;  // Calls morning.operator+(duration)
```

#### How It Works

When you write `a + b`, C++ translates it to `a.operator+(b)`:
- `a` becomes `this` (the left-hand side)
- `b` is passed as the `rhs` parameter (right-hand side)

**Advantages:**
- Direct access to private members without getters
- Clearly belongs to the class

**Limitations:**
- Left-hand side must be an instance of your class
- Cannot overload operators where your class is on the right-hand side with a built-in type on the left

[↑ Back to Table of Contents](#table-of-contents)

---

### Non-Member Function Overloading

When you overload an operator as a non-member function, it's declared outside the class. Both the left-hand side and right-hand side are passed as parameters.

#### Syntax

```cpp
ReturnType operator@(const ClassName& lhs, const ClassName& rhs);
```

#### Example: Time Class

```cpp
class Time {
public:
    Time(int h, int m, int s) : hours(h), minutes(m), seconds(s) {}
    
    int getHours() const { return hours; }
    int getMinutes() const { return minutes; }
    int getSeconds() const { return seconds; }
    
    // Declare as friend to access private members
    friend bool operator<(const Time& lhs, const Time& rhs);
    friend ostream& operator<<(ostream& out, const Time& t);
    
private:
    int hours;
    int minutes;
    int seconds;
};

// Define outside the class
bool operator<(const Time& lhs, const Time& rhs) {
    if (lhs.hours < rhs.hours) return true;
    if (rhs.hours < lhs.hours) return false;
    
    if (lhs.minutes < rhs.minutes) return true;
    if (rhs.minutes < lhs.minutes) return false;
    
    return lhs.seconds < rhs.seconds;
}

// Overload << for easy printing
ostream& operator<<(ostream& out, const Time& t) {
    out << t.hours << ":" << t.minutes << ":" << t.seconds;
    return out;  // Return stream for chaining
}

// Usage
Time morning(9, 30, 0);
Time evening(17, 45, 30);

if (morning < evening) {
    cout << "Morning comes first!" << endl;
}

cout << "Morning time: " << morning << endl;  // Output: Morning time: 9:30:0
```

#### The `friend` Keyword

If your non-member operator function needs to access private members, declare it as a `friend` inside the class:

```cpp
class Person {
public:
    friend bool operator==(const Person& lhs, const Person& rhs);
private:
    int secretID;
};

bool operator==(const Person& lhs, const Person& rhs) {
    return lhs.secretID == rhs.secretID;  // Can access private members
}
```

#### Stream Operator `<<`

The stream insertion operator is commonly overloaded as a non-member function:

```cpp
ostream& operator<<(ostream& out, const Time& time) {
    out << time.hours << ":" << time.minutes << ":" << time.seconds;
    return out;  // Must return the stream for chaining
}

// This enables chaining:
cout << "The time is " << myTime << " right now" << endl;
```

**Why non-member?** Because `cout` (an `ostream`) is on the left side, not your custom class!

**Advantages:**
- Allows the left-hand side to be a different type (e.g., `ostream` for `<<`)
- Works when you can't modify the left-hand side class
- Preferred by the C++ Standard Library for symmetry

**Considerations:**
- Needs `friend` declaration to access private members
- Or must use public getters if not declared as friend

[↑ Back to Table of Contents](#table-of-contents)

---

## Binary vs Unary Operators

Understanding the difference between binary and unary operators is crucial for proper operator overloading.

### Binary Operators

Binary operators work with **two operands** (left-hand side and right-hand side).

**Examples:** `+`, `-`, `*`, `/`, `<`, `==`, `+=`

#### As Member Functions
- Takes **one parameter** (the right-hand side)
- `this` is the left-hand side

```cpp
class Number {
public:
    Number operator+(const Number& rhs) const {  // rhs = right-hand side
        return Number(value + rhs.value);
    }
private:
    int value;
};

// Usage: a + b  →  a.operator+(b)
```

#### As Non-Member Functions
- Takes **two parameters** (both left and right sides)

```cpp
Number operator+(const Number& lhs, const Number& rhs) {
    return Number(lhs.getValue() + rhs.getValue());
}

// Usage: a + b  →  operator+(a, b)
```

### Unary Operators

Unary operators work with **one operand** only.

**Examples:** `!`, `~`, `++`, `--`, `-` (negation), `+` (positive)

#### As Member Functions
- Takes **no parameters**
- `this` is the only operand

```cpp
class Time {
public:
    bool operator!() const {  // No parameters!
        // Returns true if time is "empty" or zero
        return (hours == 0 && minutes == 0 && seconds == 0);
    }
    
    Time operator-() const {  // Unary minus (negation)
        return Time(-hours, -minutes, -seconds);
    }
private:
    int hours, minutes, seconds;
};

// Usage
Time t(0, 0, 0);
if (!t) {  // Calls t.operator!()
    cout << "Time is zero!" << endl;
}

Time negative = -t;  // Calls t.operator-()
```

#### As Non-Member Functions
- Takes **one parameter** (the operand)

```cpp
bool operator!(const Time& t) {
    return (t.getHours() == 0 && t.getMinutes() == 0 && t.getSeconds() == 0);
}

// Usage: !t  →  operator!(t)
```

### Special Case: Increment and Decrement

The `++` and `--` operators come in two forms: prefix and postfix.

```cpp
class Counter {
public:
    Counter(int val = 0) : value(val) {}
    
    // Prefix: ++counter
    Counter& operator++() {  // No parameter
        ++value;
        return *this;  // Return reference to modified object
    }
    
    // Postfix: counter++
    Counter operator++(int) {  // Dummy int parameter to distinguish
        Counter temp = *this;  // Save current value
        ++value;
        return temp;  // Return old value
    }
    
    int getValue() const { return value; }
    
private:
    int value;
};

// Usage
Counter c(5);
++c;  // c.operator++()    → c is now 6
c++;  // c.operator++(0)   → returns 6, c becomes 7
```

**Key Difference:**
- **Prefix (`++c`)**: Increments first, then returns reference to the object
- **Postfix (`c++`)**: Returns copy of original value, then increments
- The dummy `int` parameter distinguishes postfix from prefix

[↑ Back to Table of Contents](#table-of-contents)

---

## More Operator Overloading Examples

Let's explore various operators and how to overload them in real-world scenarios.

### Example 1: Complex Number Class

A comprehensive example showing multiple operators:

```cpp
class Complex {
public:
    Complex(double r = 0, double i = 0) : real(r), imag(i) {}
    
    // Binary arithmetic operators
    Complex operator+(const Complex& rhs) const {
        return Complex(real + rhs.real, imag + rhs.imag);
    }
    
    Complex operator-(const Complex& rhs) const {
        return Complex(real - rhs.real, imag - rhs.imag);
    }
    
    Complex operator*(const Complex& rhs) const {
        return Complex(
            real * rhs.real - imag * rhs.imag,
            real * rhs.imag + imag * rhs.real
        );
    }
    
    // Unary operators
    Complex operator-() const {  // Negation
        return Complex(-real, -imag);
    }
    
    // Comparison operator
    bool operator==(const Complex& rhs) const {
        return (real == rhs.real && imag == rhs.imag);
    }
    
    bool operator!=(const Complex& rhs) const {
        return !(*this == rhs);
    }
    
    // Compound assignment
    Complex& operator+=(const Complex& rhs) {
        real += rhs.real;
        imag += rhs.imag;
        return *this;  // Return reference for chaining
    }
    
    // Stream output
    friend ostream& operator<<(ostream& out, const Complex& c) {
        out << c.real;
        if (c.imag >= 0) out << "+";
        out << c.imag << "i";
        return out;
    }
    
private:
    double real;
    double imag;
};

// Usage
Complex a(3, 4);   // 3 + 4i
Complex b(1, -2);  // 1 - 2i

Complex sum = a + b;        // 4 + 2i
Complex product = a * b;    // 11 - 2i
Complex negative = -a;       // -3 - 4i

cout << "Sum: " << sum << endl;
a += b;  // a is now 4 + 2i
```

### Example 2: Boolean Logic Class

```cpp
class BoolExpr {
public:
    BoolExpr(bool val) : value(val) {}
    
    // Logical operators
    BoolExpr operator&&(const BoolExpr& rhs) const {
        return BoolExpr(value && rhs.value);
    }
    
    BoolExpr operator||(const BoolExpr& rhs) const {
        return BoolExpr(value || rhs.value);
    }
    
    BoolExpr operator!() const {
        return BoolExpr(!value);
    }
    
    // Conversion to bool
    operator bool() const {
        return value;
    }
    
    friend ostream& operator<<(ostream& out, const BoolExpr& expr) {
        out << (expr.value ? "true" : "false");
        return out;
    }
    
private:
    bool value;
};

// Usage
BoolExpr a(true);
BoolExpr b(false);

BoolExpr result = a && b;  // false
BoolExpr negation = !a;     // false

if (a || b) {
    cout << "At least one is true" << endl;
}
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Best Practices

### 1. Make Operators Obvious
The operation should be intuitive when reading the code. If someone sees `a + b`, they should have a good idea what it means.

### 2. Stay Consistent with Built-in Types
Operators should behave similarly to how they work with built-in types:
- `+` should perform addition-like operations
- `<` should perform comparisons
- Don't make `+` do subtraction!

### 3. When In Doubt, Use a Named Function
If the meaning isn't obvious, use a descriptive function name instead:

```cpp
// 🚫 Confusing
MyString a("hello");
MyString b("world");
MyString c = a * b;  // What does this even mean?

// ✅ Clear
MyString a("hello");
MyString b("world");
MyString c = a.charsInCommon(b);  // Much better!
```

### 4. Choose Member vs Non-Member Appropriately
- Use **member functions** when the operator logically belongs to the class
- Use **non-member functions** when:
  - You need a different type on the left-hand side
  - You want symmetry between operands
  - Overloading stream operators (`<<`, `>>`)

### 5. Return Appropriate Types
- Comparison operators (`<`, `==`, etc.) should return `bool`
- Arithmetic operators (`+`, `-`, etc.) should return a new object
- Assignment operators (`=`, `+=`, etc.) should return a reference to `*this`

[↑ Back to Table of Contents](#table-of-contents)