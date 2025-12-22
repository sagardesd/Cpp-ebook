# Data Types, Variables, and Input/Output in C++

## Table of Contents
1. [Introduction](#introduction)
2. [Variables - Your Data Containers](#variables)
3. [Data Types in C++](#data-types)
4. [Input and Output](#input-output)
5. [Choosing the Right Data Type](#choosing-right-type)
6. [Common Mistakes and Best Practices](#best-practices)

---

<a id="introduction"></a>
## Introduction

Think of C++ programming like cooking. Before you start cooking, you need containers (variables) to store your ingredients (data), and you need to know what type of container to use - you wouldn't store soup in a sieve! Similarly, in C++, we need to understand what kind of data we're working with and choose the appropriate "container" for it.

---

<a id="variables"></a>
## Variables - Your Data Containers

### What is a Variable?

A **variable** is a named storage location in your computer's memory that holds a value. Think of it as a labeled box where you can store information and retrieve it later.

### Variable Declaration Syntax

```cpp
dataType variableName = value;
```

**Example:**
```cpp
int age = 25;           // 'int' is the type, 'age' is the name, '25' is the value
double price = 19.99;   // Storing a decimal number
char grade = 'A';       // Storing a single character
```

### Variable Naming Rules

**Allowed:**
- Start with a letter (a-z, A-Z) or underscore (_)
- Contain letters, digits, and underscores
- Examples: `age`, `student_name`, `price2`, `_count`

**Not Allowed:**
- Start with a digit: `2names`
- Contain spaces: `student name`
- Use C++ keywords: `int`, `return`, `class`
- Special characters: `price$`, `name@`

### Best Naming Practices

```cpp
// Good - descriptive names
int studentAge = 18;
double accountBalance = 1500.50;
char firstInitial = 'J';

// Bad - unclear names
int x = 18;      // What does x represent?
double a = 1500.50;  // What is 'a'?
char c = 'J';    // What does 'c' mean?
```

---

<a id="data-types"></a>
## Data Types in C++

C++ has several built-in data types. Let's explore each category:

### 1. Integer Types (Whole Numbers)

These store whole numbers without decimal points.

**Important Note:** The size of integer types can vary depending on your platform (32-bit vs 64-bit system, compiler, operating system). The table below shows typical sizes, but always verify on your system using `sizeof()`.

| Type | Typical Size | Typical Range | When to Use |
|------|--------------|---------------|-------------|
| `short` | 2 bytes | -32,768 to 32,767 | Small numbers, save memory |
| `int` | 4 bytes (most common) | -2,147,483,648 to 2,147,483,647 | General purpose counting, IDs, ages |
| `long` | 4 or 8 bytes* | Platform dependent | Large calculations, timestamps |
| `long long` | 8 bytes (guaranteed) | Very large numbers | Scientific calculations, guaranteed 64-bit |

*Note: `long` is 4 bytes on Windows (32/64-bit) and most 32-bit systems, but 8 bytes on 64-bit Linux/Mac.

**Examples:**

```cpp
int studentCount = 30;           // Number of students in class
short temperature = -15;         // Temperature in Celsius
long worldPopulation = 8000000000L;  // World population
long long distanceToSun = 149600000000LL;  // Distance in meters
```

#### Unsigned Integers (Only Positive Numbers)

If you know your number will **never be negative**, use `unsigned` to double the positive range:

```cpp
unsigned int age = 25;           // Age is never negative
unsigned short score = 100;      // Score is always positive
unsigned long fileSize = 5000000;  // File sizes are positive
```

### 2. Floating-Point Types (Decimal Numbers)

These store numbers with decimal points.

| Type | Typical Size | Precision | When to Use |
|------|--------------|-----------|-------------|
| `float` | 4 bytes | ~7 decimal digits | Basic decimals, graphics |
| `double` | 8 bytes | ~15 decimal digits | Scientific calculations (MOST COMMON) |
| `long double` | 8-16 bytes* | ~19 decimal digits | Extreme precision needed |

*Note: `long double` size varies: 8 bytes (some systems), 12 bytes (Linux x86), 16 bytes (some 64-bit systems).

**Examples:**

```cpp
float pi = 3.14159f;              // 'f' suffix for float
double accountBalance = 1234.56;  // Most commonly used
double scientificValue = 3.14159265358979;
long double preciseValue = 3.141592653589793238L;
```

**💡 Key Point:** Use `double` by default for decimal numbers. Only use `float` if memory is critical (like in games with thousands of objects).

### 3. Character Type

Stores a **single character** enclosed in single quotes `' '`.

```cpp
char grade = 'A';
char symbol = '$';
char digit = '5';        // This is a character, not a number!
char newline = '\n';     // Special character for new line
```

**Special (Escape) Characters:**

```cpp
'\n'  // New line
'\t'  // Tab
'\\'  // Backslash
'\''  // Single quote
'\"'  // Double quote
```

### 4. Boolean Type

Stores only two values: `true` or `false`.

```cpp
bool isStudent = true;
bool hasLicense = false;
bool isPassing = (grade >= 60);  // Result of comparison
```

**💡 Use Case:** Perfect for yes/no situations, flags, conditions.

### 5. String Type (Text)

Stores sequences of characters (words, sentences). **Note:** You need to include `<string>` header.

```cpp
#include <string>

string name = "John Doe";
string message = "Hello, World!";
string empty = "";           // Empty string
```

**String vs Char:**
```cpp
char singleLetter = 'A';      // Single character - single quotes
string word = "A";            // String - double quotes
string fullName = "Alice";    // Multiple characters
```

---

## Checking Data Type Sizes

Since data type sizes can vary by platform, C++ provides the `sizeof()` operator to check the actual size on your system.

### The `sizeof()` Operator

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    cout << "=== Data Type Sizes on This System ===" << endl;
    cout << "Note: Size is shown in bytes (1 byte = 8 bits)\n" << endl;
    
    // Integer types
    cout << "INTEGER TYPES:" << endl;
    cout << "short          : " << sizeof(short) << " bytes" << endl;
    cout << "int            : " << sizeof(int) << " bytes" << endl;
    cout << "long           : " << sizeof(long) << " bytes" << endl;
    cout << "long long      : " << sizeof(long long) << " bytes" << endl;
    cout << "unsigned int   : " << sizeof(unsigned int) << " bytes" << endl;
    
    // Floating-point types
    cout << "\nFLOATING-POINT TYPES:" << endl;
    cout << "float          : " << sizeof(float) << " bytes" << endl;
    cout << "double         : " << sizeof(double) << " bytes" << endl;
    cout << "long double    : " << sizeof(long double) << " bytes" << endl;
    
    // Character and boolean
    cout << "\nCHARACTER & BOOLEAN:" << endl;
    cout << "char           : " << sizeof(char) << " bytes" << endl;
    cout << "bool           : " << sizeof(bool) << " bytes" << endl;
    
    // String (note: string size varies based on content)
    cout << "\nSTRING:" << endl;
    string emptyStr = "";
    string shortStr = "Hi";
    string longStr = "This is a longer string";
    cout << "string (empty) : " << sizeof(emptyStr) << " bytes (object overhead)" << endl;
    cout << "string (short) : " << sizeof(shortStr) << " bytes (same overhead)" << endl;
    cout << "string (long)  : " << sizeof(longStr) << " bytes (same overhead)" << endl;
    cout << "Note: String object has fixed size; actual text stored separately" << endl;
    
    // You can also check variable sizes
    cout << "\n=== Checking Variable Sizes ===" << endl;
    int myAge = 25;
    double myHeight = 5.9;
    char myGrade = 'A';
    
    cout << "int myAge      : " << sizeof(myAge) << " bytes" << endl;
    cout << "double myHeight: " << sizeof(myHeight) << " bytes" << endl;
    cout << "char myGrade   : " << sizeof(myGrade) << " bytes" << endl;
    
    return 0;
}
```

### Sample Output (may vary on your system):

```
=== Data Type Sizes on This System ===
Note: Size is shown in bytes (1 byte = 8 bits)

INTEGER TYPES:
short          : 2 bytes
int            : 4 bytes
long           : 8 bytes
long long      : 8 bytes
unsigned int   : 4 bytes

FLOATING-POINT TYPES:
float          : 4 bytes
double         : 8 bytes
long double    : 16 bytes

CHARACTER & BOOLEAN:
char           : 1 bytes
bool           : 1 bytes

STRING:
string (empty) : 32 bytes (object overhead)
string (short) : 32 bytes (same overhead)
string (long)  : 32 bytes (same overhead)
Note: String object has fixed size; actual text stored separately

=== Checking Variable Sizes ===
int myAge      : 4 bytes
double myHeight: 8 bytes
char myGrade   : 1 bytes
```

**💡 Key Insights:**
- `sizeof()` returns the size in bytes
- Use `sizeof(type)` or `sizeof(variable)`
- Run this program on your computer to see platform-specific sizes
- String object size doesn't change with content length (uses dynamic memory)

---

<a id="input-output"></a>
## Input and Output

### Output with `cout`

`cout` (console output) displays information to the screen.

```cpp
#include <iostream>
using namespace std;

int main() {
    cout << "Hello, World!";              // Display text
    cout << "Hello" << " " << "World";    // Multiple outputs
    cout << "Line 1" << endl;             // endl = new line
    cout << "Line 2\n";                   // \n = new line
    
    int age = 25;
    cout << "Age: " << age << endl;       // Mix text and variables
    
    return 0;
}
```

**Output:**
```
Hello, World!Hello World
Line 1
Line 2
Age: 25
```

### Input with `cin`

`cin` (console input) reads data from the keyboard.

```cpp
#include <iostream>
using namespace std;

int main() {
    int age;
    cout << "Enter your age: ";
    cin >> age;                  // Wait for user input
    cout << "You are " << age << " years old." << endl;
    
    return 0;
}
```

### Multiple Inputs

```cpp
int day, month, year;
cout << "Enter date (DD MM YYYY): ";
cin >> day >> month >> year;
cout << "Date: " << day << "/" << month << "/" << year << endl;
```

### Input for Strings

**Problem with `cin` and strings:**

```cpp
string name;
cout << "Enter your name: ";
cin >> name;              // Only reads until first space!
// Input: "John Doe"
// name = "John" (Doe is ignored!)
```

**Solution - Use `getline()`:**

```cpp
string fullName;
cout << "Enter your full name: ";
getline(cin, fullName);    // Reads entire line including spaces
```

### Complete Input/Output Example

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    // Declare variables
    string name;
    int age;
    double height;
    char grade;
    
    // Input
    cout << "Enter your name: ";
    getline(cin, name);
    
    cout << "Enter your age: ";
    cin >> age;
    
    cout << "Enter your height (in meters): ";
    cin >> height;
    
    cout << "Enter your grade: ";
    cin >> grade;
    
    // Output
    cout << "\n--- Your Information ---" << endl;
    cout << "Name: " << name << endl;
    cout << "Age: " << age << " years" << endl;
    cout << "Height: " << height << " meters" << endl;
    cout << "Grade: " << grade << endl;
    
    return 0;
}
```

---

<a id="choosing-right-type"></a>
## Choosing the Right Data Type

### Decision Guide

**1. Need to store whole numbers (no decimals)?**
- Small numbers (-32,768 to 32,767): `short`
- Regular numbers: `int` (MOST COMMON)
- Very large numbers: `long` or `long long`
- Only positive numbers: Add `unsigned`

**Examples:**
```cpp
int studentID = 12345;        // Student IDs
unsigned int pageViews = 5000; // Website views (never negative)
long long accountNumber = 9876543210123456LL; // Bank accounts
```

**2. Need decimal numbers?**
- Use `double` (99% of cases)
- Use `float` only if memory is critical
- Use `long double` for extreme precision

**Examples:**
```cpp
double price = 29.99;          // Prices, measurements
double temperature = 36.6;     // Body temperature
float gamePosition = 10.5f;    // Game coordinates (memory critical)
```

**3. Need a single character?**
- Use `char`

**Examples:**
```cpp
char menuChoice = 'A';         // Menu selections
char yesNo = 'Y';              // Simple yes/no
```

**4. Need text (words/sentences)?**
- Use `string`

**Examples:**
```cpp
string username = "alice123";
string email = "user@example.com";
string address = "123 Main St, City";
```

**5. Need true/false?**
- Use `bool`

**Examples:**
```cpp
bool isLoggedIn = true;
bool isPremiumUser = false;
bool hasPermission = (userLevel > 5);
```

### Real-World Scenarios

#### Scenario 1: Student Management System
```cpp
int studentID = 1001;              // Unique ID
string studentName = "Alice Johnson";
int age = 20;
double gpa = 3.85;
char letterGrade = 'A';
bool isEnrolled = true;
```

#### Scenario 2: E-commerce Product
```cpp
int productID = 5432;
string productName = "Wireless Mouse";
double price = 24.99;
unsigned int stockQuantity = 150;  // Never negative
bool inStock = (stockQuantity > 0);
float rating = 4.5f;
```

#### Scenario 3: Banking Application
```cpp
long long accountNumber = 1234567890123456LL;
string accountHolder = "John Doe";
double balance = 5432.10;
bool isActive = true;
unsigned int transactionCount = 523;
```

---

<a id="best-practices"></a>
## Common Mistakes and Best Practices

### Common Mistakes

**1. Integer Division:**
```cpp
int a = 5, b = 2;
int result = a / b;        // result = 2 (not 2.5!)
// Integers ignore decimals

// Fix:
double result = 5.0 / 2.0;  // result = 2.5
```

**2. Mixing `cin` and `getline`:**
```cpp
int age;
string name;

cin >> age;           // Leaves newline in buffer
getline(cin, name);   // Reads empty line!

// Fix:
cin >> age;
cin.ignore();         // Clear the newline
getline(cin, name);   // Now works correctly
```

**3. Forgetting Variable Initialization:**
```cpp
int count;            // Uninitialized - contains garbage value
cout << count;        // Unpredictable output!

// Better:
int count = 0;        // Always initialize
```

**4. Using Wrong Data Type:**
```cpp
int price = 19.99;    // price = 19 (decimal lost!)
// Should use: double price = 19.99;
```

### Best Practices

**1. Always Initialize Variables:**
```cpp
int count = 0;
double total = 0.0;
string name = "";
bool isValid = false;
```

**2. Use Meaningful Names:**
```cpp
// Bad
int d = 7;
double x = 19.99;

// Good
int daysInWeek = 7;
double productPrice = 19.99;
```

**3. Use `const` for Constants:**
```cpp
const double PI = 3.14159;
const int MAX_STUDENTS = 50;
const string COMPANY_NAME = "TechCorp";
```

**4. Choose Appropriate Data Types:**
```cpp
// Age is always positive and small
unsigned short age = 25;

// Money needs decimals
double salary = 75000.50;

// IDs are whole numbers
int employeeID = 1234;
```

**5. Comment Your Code:**
```cpp
int maxAttempts = 3;  // Maximum login attempts allowed
double taxRate = 0.15;  // 15% tax rate
```

---

## Quick Reference Card

| Need | Use | Example |
|------|-----|---------|
| Whole numbers | `int` | `int count = 10;` |
| Large whole numbers | `long long` | `long long distance = 1000000000LL;` |
| Positive numbers only | `unsigned int` | `unsigned int age = 25;` |
| Decimal numbers | `double` | `double price = 19.99;` |
| Single character | `char` | `char grade = 'A';` |
| Text | `string` | `string name = "John";` |
| True/False | `bool` | `bool isActive = true;` |

---

## Practice Exercise

Try creating a simple program to practice:

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    // Create a program that asks for:
    // 1. User's full name (string)
    // 2. Age (int)
    // 3. Height in meters (double)
    // 4. Favorite letter (char)
    // 5. Are you a student? (bool - input 1 for true, 0 for false)
    
    // Then display all information in a formatted way
    
    return 0;
}
```

---

## Summary

- **Variables** are containers that store data
- **Data types** define what kind of data a variable can hold
- Use `int` for whole numbers, `double` for decimals, `string` for text
- Use `cout` to display output, `cin` for input
- Always initialize your variables
- Choose data types based on what you're storing
- Use meaningful variable names
