# Control Flow in C++

## Table of Contents
1. [Introduction](#introduction)
2. [Decision Making - if-else](#if-else)
3. [Switch Case Statement](#switch-case)
4. [Loops](#loops)
5. [Break and Continue](#break-continue)
6. [Best Practices](#best-practices)
7. [Practice Problems](#practice-problems)

---

## Introduction

Control flow statements allow your program to make decisions and repeat actions. Think of them as traffic signals and road signs that direct the flow of your program's execution.

**Three main categories:**
- **Decision Making**: if-else, switch (choosing a path)
- **Loops**: for, while, do-while (repeating actions)
- **Jump Statements**: break, continue (controlling loop behavior)

---
<a id="if-else"></a>
## Decision Making - if-else

### Basic if Statement

Executes code only if a condition is true.

**Syntax:**
```cpp
if (condition) {
    // code to execute if condition is true
}
```

**Example:**
```cpp
int age = 18;

if (age >= 18) {
    cout << "You are an adult." << endl;
}
```

### if-else Statement

Provides an alternative when the condition is false.

**Syntax:**
```cpp
if (condition) {
    // code if condition is true
} else {
    // code if condition is false
}
```

**Example:**
```cpp
int marks = 45;

if (marks >= 50) {
    cout << "You passed!" << endl;
} else {
    cout << "You failed. Try again!" << endl;
}
```

### if-else if-else Ladder

Tests multiple conditions in sequence.

**Syntax:**
```cpp
if (condition1) {
    // code if condition1 is true
} else if (condition2) {
    // code if condition2 is true
} else if (condition3) {
    // code if condition3 is true
} else {
    // code if all conditions are false
}
```

**Example: Grade Calculator**
```cpp
#include <iostream>
using namespace std;

int main() {
    int marks;
    cout << "Enter your marks (0-100): ";
    cin >> marks;
    
    if (marks >= 90) {
        cout << "Grade: A+ (Excellent!)" << endl;
    } else if (marks >= 80) {
        cout << "Grade: A (Very Good)" << endl;
    } else if (marks >= 70) {
        cout << "Grade: B (Good)" << endl;
    } else if (marks >= 60) {
        cout << "Grade: C (Average)" << endl;
    } else if (marks >= 50) {
        cout << "Grade: D (Pass)" << endl;
    } else {
        cout << "Grade: F (Fail)" << endl;
    }
    
    return 0;
}
```

### Nested if Statements

if statements inside other if statements.

**Example: Login System**
```cpp
string username, password;
cout << "Enter username: ";
cin >> username;

if (username == "admin") {
    cout << "Enter password: ";
    cin >> password;
    
    if (password == "1234") {
        cout << "Login successful! Welcome, Admin." << endl;
    } else {
        cout << "Incorrect password!" << endl;
    }
} else {
    cout << "User not found!" << endl;
}
```

### Ternary Operator (Shorthand if-else)

A compact way to write simple if-else statements.

**Syntax:**
```cpp
condition ? value_if_true : value_if_false;
```

**Example:**
```cpp
int age = 20;
string status = (age >= 18) ? "Adult" : "Minor";
cout << status << endl;  // Output: Adult

// Equivalent to:
string status;
if (age >= 18) {
    status = "Adult";
} else {
    status = "Minor";
}
```

**More Examples:**
```cpp
int a = 10, b = 20;
int max = (a > b) ? a : b;  // max = 20

int marks = 75;
cout << "Result: " << (marks >= 50 ? "Pass" : "Fail") << endl;
```

### Logical Operators in Conditions

Combine multiple conditions:

| Operator | Meaning | Example |
|----------|---------|---------|
| `&&` | AND (both must be true) | `(age >= 18 && hasLicense)` |
| `\|\|` | OR (at least one must be true) | `(day == "Sat" \|\| day == "Sun")` |
| `!` | NOT (reverses the condition) | `!(isRaining)` |

**Basic Examples:**
```cpp
int age = 25;
bool hasLicense = true;

// AND operator
if (age >= 18 && hasLicense) {
    cout << "You can drive!" << endl;
}

// OR operator
string day = "Sunday";
if (day == "Saturday" || day == "Sunday") {
    cout << "It's the weekend!" << endl;
}

// NOT operator
bool isRaining = false;
if (!isRaining) {
    cout << "Let's go outside!" << endl;
}

// Complex condition
int marks = 85;
int attendance = 75;
if (marks >= 50 && attendance >= 75) {
    cout << "Eligible for certificate" << endl;
}
```
---

## Short-Circuit Evaluation (IMPORTANT!)

C++ uses **short-circuit evaluation** for logical operators. This is a crucial concept for writing efficient and safe code.

### How && (AND) Short-Circuits

**Rule:** If the **first condition is FALSE**, the remaining conditions are **NOT evaluated**.

**Why?** If one condition in AND is false, the entire expression is false. No need to check further.

**Example 1: Basic Short-Circuit**
```cpp
int x = 5;
int y = 10;

// Second condition is NOT checked because first is false
if (x > 10 && y > 5) {
    cout << "This won't print" << endl;
}
// x > 10 is false, so y > 5 is never evaluated
```

**Example 2: Demonstrating with Functions**
```cpp
#include <iostream>
using namespace std;

bool checkFirst() {
    cout << "Checking first condition..." << endl;
    return false;
}

bool checkSecond() {
    cout << "Checking second condition..." << endl;
    return true;
}

int main() {
    cout << "Testing AND (&&):" << endl;
    if (checkFirst() && checkSecond()) {
        cout << "Both true" << endl;
    }
    
    // Output:
    // Testing AND (&&):
    // Checking first condition...
    // (checkSecond() is NEVER called!)
    
    return 0;
}
```

**Example 3: Preventing Division by Zero**
```cpp
int a = 10;
int b = 0;

// ✅ SAFE: b != 0 is checked first
if (b != 0 && a / b > 2) {
    cout << "Division result is greater than 2" << endl;
}
// If b is 0, the division never happens!

// ❌ DANGEROUS: Would crash if written the other way
// if (a / b > 2 && b != 0) {  // WRONG! Division happens first!
```

**Example 4: Null Pointer Check**
```cpp
int* ptr = nullptr;

// ✅ SAFE: Check pointer before dereferencing
if (ptr != nullptr && *ptr > 10) {
    cout << "Value is greater than 10" << endl;
}
// If ptr is null, *ptr is never accessed

// ❌ DANGEROUS: Would crash
// if (*ptr > 10 && ptr != nullptr) {  // WRONG! Dereferencing null pointer!
```

### How || (OR) Short-Circuits

**Rule:** If the **first condition is TRUE**, the remaining conditions are **NOT evaluated**.

**Why?** If one condition in OR is true, the entire expression is true. No need to check further.

**Example 1: Basic Short-Circuit**
```cpp
int x = 15;
int y = 10;

// Second condition is NOT checked because first is true
if (x > 10 || y > 15) {
    cout << "At least one condition is true" << endl;
}
// x > 10 is true, so y > 15 is never evaluated
```

**Example 2: Demonstrating with Functions**
```cpp
#include <iostream>
using namespace std;

bool checkFirst() {
    cout << "Checking first condition..." << endl;
    return true;
}

bool checkSecond() {
    cout << "Checking second condition..." << endl;
    return false;
}

int main() {
    cout << "Testing OR (||):" << endl;
    if (checkFirst() || checkSecond()) {
        cout << "At least one is true" << endl;
    }
    
    // Output:
    // Testing OR (||):
    // Checking first condition...
    // At least one is true
    // (checkSecond() is NEVER called!)
    
    return 0;
}
```

**Example 3: Default Value Check**
```cpp
string username;
cout << "Enter username: ";
cin >> username;

// Check if empty first (fast check)
if (username.empty() || username == "guest") {
    username = "Anonymous";
}
// If username is empty, the comparison never happens
```

**Example 4: Permission Check**
```cpp
bool isAdmin = false;
bool isOwner = true;
bool hasPermission = false;

// ✅ Efficient: Checks in order of likelihood
if (isAdmin || isOwner || hasPermission) {
    cout << "Access granted!" << endl;
}
// If isAdmin is true, other checks are skipped
```

### Short-Circuit Evaluation Comparison

```cpp
#include <iostream>
using namespace std;

int callCount = 0;

bool expensive_check() {
    callCount++;
    cout << "Expensive check called (count: " << callCount << ")" << endl;
    return true;
}

int main() {
    callCount = 0;
    
    // Test 1: AND with false first
    cout << "\n=== Test 1: AND with false first ===" << endl;
    if (false && expensive_check()) {
        cout << "This won't execute" << endl;
    }
    cout << "Expensive check was called " << callCount << " times" << endl;
    // Output: 0 times (never called!)
    
    // Test 2: AND with true first
    callCount = 0;
    cout << "\n=== Test 2: AND with true first ===" << endl;
    if (true && expensive_check()) {
        cout << "This will execute" << endl;
    }
    cout << "Expensive check was called " << callCount << " times" << endl;
    // Output: 1 time
    
    // Test 3: OR with true first
    callCount = 0;
    cout << "\n=== Test 3: OR with true first ===" << endl;
    if (true || expensive_check()) {
        cout << "This will execute" << endl;
    }
    cout << "Expensive check was called " << callCount << " times" << endl;
    // Output: 0 times (never called!)
    
    // Test 4: OR with false first
    callCount = 0;
    cout << "\n=== Test 4: OR with false first ===" << endl;
    if (false || expensive_check()) {
        cout << "This will execute" << endl;
    }
    cout << "Expensive check was called " << callCount << " times" << endl;
    // Output: 1 time
    
    return 0;
}
```

---

## Best Practices for Logical Operators

### 1. Order Matters for Safety

**Rule:** Always put safety checks FIRST in AND operations.

```cpp
// ✅ CORRECT: Check for null/zero first
if (ptr != nullptr && *ptr > 10) { }
if (denominator != 0 && numerator / denominator > 5) { }
if (!array.empty() && array[0] == 10) { }

// ❌ WRONG: Dangerous operations first
if (*ptr > 10 && ptr != nullptr) { }  // Crash if ptr is null!
if (numerator / denominator > 5 && denominator != 0) { }  // Division by zero!
if (array[0] == 10 && !array.empty()) { }  // Access invalid memory!
```

**Real-World Example:**
```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    string* namePtr = nullptr;
    
    // ✅ SAFE: Check pointer first
    if (namePtr != nullptr && namePtr->length() > 0) {
        cout << "Name: " << *namePtr << endl;
    } else {
        cout << "No name available" << endl;
    }
    
    // ❌ This would CRASH:
    // if (namePtr->length() > 0 && namePtr != nullptr) { }
    
    return 0;
}
```

### 2. Order Matters for Performance

**Rule:** Put **cheap/fast checks** FIRST, **expensive checks** LAST.

```cpp
int age = 25;
bool hasComplexPermission() {
    // Imagine this function does expensive database lookup
    // Takes 100ms to execute
    return true;
}

// ✅ EFFICIENT: Fast check first
if (age >= 18 && hasComplexPermission()) {
    cout << "Access granted" << endl;
}
// If age < 18, expensive function is never called

// ❌ INEFFICIENT: Expensive check first
if (hasComplexPermission() && age >= 18) {
    cout << "Access granted" << endl;
}
// Expensive function ALWAYS called, even if age < 18
```

**Another Example:**
```cpp
string username = "john";
bool isDatabaseUserValid(string user) {
    // Expensive: queries database
    cout << "Querying database..." << endl;
    return true;
}

// ✅ EFFICIENT: Check local variable first
if (!username.empty() && username.length() > 3 && isDatabaseUserValid(username)) {
    cout << "Valid user" << endl;
}
// Database only queried if basic checks pass

// ❌ INEFFICIENT: Database check first
if (isDatabaseUserValid(username) && username.length() > 3) {
    cout << "Valid user" << endl;
}
// Database queried every time, even for invalid usernames
```

### 3. Order for OR Operations

**Rule:** Put **most likely to be true** conditions FIRST.

```cpp
bool isWeekend(string day) {
    // ✅ EFFICIENT: Most common cases first
    if (day == "Saturday" || day == "Sunday") {
        return true;
    }
    return false;
}

// In a user role check:
bool hasAccess() {
    // Put most common role first
    // ✅ If 80% users are "member", check that first
    if (role == "member" || role == "admin" || role == "moderator") {
        return true;
    }
    return false;
}
```

### 4. Readability vs Performance Trade-off

**Sometimes clarity is more important than micro-optimization:**

```cpp
// Option 1: Optimized but less clear
if (ptr && *ptr > 10 && calculate(ptr)) { }

// Option 2: Clearer with separate checks
if (ptr != nullptr) {
    if (*ptr > 10) {
        if (calculate(ptr)) {
            // do something
        }
    }
}
```

**Best approach: Balance both:**
```cpp
// ✅ GOOD: Clear AND efficient
bool isValid = (ptr != nullptr);
bool hasValue = isValid && (*ptr > 10);
bool passesCalculation = hasValue && calculate(ptr);

if (passesCalculation) {
    // do something
}
```

### 5. Complex Conditions - Use Parentheses

```cpp
// ❌ Confusing
if (a && b || c && d) { }

// ✅ Clear with parentheses
if ((a && b) || (c && d)) { }

// Even better with meaningful variables
bool firstConditionMet = (a && b);
bool secondConditionMet = (c && d);
if (firstConditionMet || secondConditionMet) { }
```

### 6. Avoid Side Effects in Conditions

```cpp
int count = 0;

// ❌ BAD: Side effect (incrementing) in condition
if (count++ > 5 && someFunction()) {
    // count might not increment if first condition is false!
}

// ✅ GOOD: Separate side effects
count++;
if (count > 5 && someFunction()) {
    // Clear and predictable
}
```

---

## Practical Scenarios

### Scenario 1: Form Validation

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    string email, password;
    
    cout << "Enter email: ";
    cin >> email;
    cout << "Enter password: ";
    cin >> password;
    
    // ✅ GOOD: Check simple conditions first
    if (!email.empty() && 
        email.find('@') != string::npos && 
        password.length() >= 8) {
        cout << "Registration successful!" << endl;
    } else {
        cout << "Invalid email or password too short" << endl;
    }
    
    return 0;
}
```

### Scenario 2: Safe Array Access

```cpp
#include <iostream>
using namespace std;

int main() {
    int scores[] = {85, 90, 78, 92, 88};
    int size = 5;
    int index;
    
    cout << "Enter index to view (0-4): ";
    cin >> index;
    
    // ✅ SAFE: Check bounds before accessing
    if (index >= 0 && index < size && scores[index] >= 80) {
        cout << "High score: " << scores[index] << endl;
    } else if (index >= 0 && index < size) {
        cout << "Score: " << scores[index] << endl;
    } else {
        cout << "Invalid index!" << endl;
    }
    
    return 0;
}
```

### Scenario 3: User Permissions

```cpp
#include <iostream>
#include <string>
using namespace std;

int main() {
    string role = "user";
    int accountAge = 30;  // days
    bool emailVerified = true;
    
    // ✅ Efficient: Check from least to most restrictive
    // Most users will fail early checks quickly
    if (emailVerified && 
        accountAge >= 7 && 
        (role == "admin" || role == "moderator" || role == "premium")) {
        cout << "Access to premium features granted!" << endl;
    } else {
        cout << "Upgrade to premium for this feature" << endl;
    }
    
    return 0;
}
```

### Scenario 4: Game Damage Calculation

```cpp
#include <iostream>
using namespace std;

int main() {
    int playerHealth = 50;
    int armor = 30;
    int incomingDamage = 40;
    bool hasShield = true;
    
    // ✅ Process shields first (cheaper check)
    if (hasShield && incomingDamage > 0) {
        cout << "Shield absorbed the damage!" << endl;
        hasShield = false;
    } else if (armor > 0 && incomingDamage > armor) {
        incomingDamage -= armor;
        armor = 0;
        playerHealth -= incomingDamage;
        cout << "Armor damaged! Health: " << playerHealth << endl;
    } else if (armor > 0) {
        armor -= incomingDamage;
        cout << "Armor absorbed damage. Remaining: " << armor << endl;
    } else {
        playerHealth -= incomingDamage;
        cout << "Direct hit! Health: " << playerHealth << endl;
    }
    
    if (playerHealth <= 0) {
        cout << "Game Over!" << endl;
    }
    
    return 0;
}
```

---

## Summary: Logical Operators

| Operator | Short-Circuit | When to Use | Order Strategy |
|----------|---------------|-------------|----------------|
| `&&` | Stops at first FALSE | All conditions must be true | Safety checks first, then expensive checks |
| `\|\|` | Stops at first TRUE | At least one must be true | Most likely true conditions first |
| `!` | No short-circuit | Reverse a condition | Use sparingly for clarity |

**Key Takeaways:**
1. **Safety first**: Always check null/zero/bounds before using
2. **Performance**: Put cheap checks before expensive ones
3. **Readability**: Use parentheses for complex conditions
4. **Predictability**: Avoid side effects in conditions
5. **Short-circuit is your friend**: Use it to write safer, faster code

---
<a id="switch-case"></a>
## Switch Case Statement

Executes different code blocks based on the value of a variable. Better than multiple if-else when checking one variable against many values.

### Basic Syntax

```cpp
switch (expression) {
    case value1:
        // code for value1
        break;
    case value2:
        // code for value2
        break;
    case value3:
        // code for value3
        break;
    default:
        // code if no case matches
}
```

**⚠️ Important:** 
- `break` is crucial - without it, execution "falls through" to next case
- `switch` works with `int`, `char`, and `enum` (NOT with `string` or `float`)
- `default` is optional but recommended

### Example 1: Menu System

```cpp
#include <iostream>
using namespace std;

int main() {
    int choice;
    cout << "=== Menu ===" << endl;
    cout << "1. Coffee" << endl;
    cout << "2. Tea" << endl;
    cout << "3. Juice" << endl;
    cout << "4. Water" << endl;
    cout << "Enter your choice (1-4): ";
    cin >> choice;
    
    switch (choice) {
        case 1:
            cout << "You ordered Coffee. Price: $3" << endl;
            break;
        case 2:
            cout << "You ordered Tea. Price: $2" << endl;
            break;
        case 3:
            cout << "You ordered Juice. Price: $4" << endl;
            break;
        case 4:
            cout << "You ordered Water. Price: Free!" << endl;
            break;
        default:
            cout << "Invalid choice!" << endl;
    }
    
    return 0;
}
```

### Example 2: Day of the Week

```cpp
char day;
cout << "Enter first letter of day (M/T/W/F/S): ";
cin >> day;

switch (day) {
    case 'M':
        cout << "Monday" << endl;
        break;
    case 'T':
        cout << "Tuesday or Thursday" << endl;
        break;
    case 'W':
        cout << "Wednesday" << endl;
        break;
    case 'F':
        cout << "Friday" << endl;
        break;
    case 'S':
        cout << "Saturday or Sunday" << endl;
        break;
    default:
        cout << "Invalid input!" << endl;
}
```

### Fall-Through Cases (Intentional)

Sometimes you want multiple cases to execute the same code:

```cpp
int month;
cout << "Enter month number (1-12): ";
cin >> month;

switch (month) {
    case 12:
    case 1:
    case 2:
        cout << "Winter" << endl;
        break;
    case 3:
    case 4:
    case 5:
        cout << "Spring" << endl;
        break;
    case 6:
    case 7:
    case 8:
        cout << "Summer" << endl;
        break;
    case 9:
    case 10:
    case 11:
        cout << "Fall" << endl;
        break;
    default:
        cout << "Invalid month!" << endl;
}
```

### Calculator Example

```cpp
double num1, num2;
char operation;

cout << "Enter first number: ";
cin >> num1;
cout << "Enter operation (+, -, *, /): ";
cin >> operation;
cout << "Enter second number: ";
cin >> num2;

switch (operation) {
    case '+':
        cout << "Result: " << (num1 + num2) << endl;
        break;
    case '-':
        cout << "Result: " << (num1 - num2) << endl;
        break;
    case '*':
        cout << "Result: " << (num1 * num2) << endl;
        break;
    case '/':
        if (num2 != 0) {
            cout << "Result: " << (num1 / num2) << endl;
        } else {
            cout << "Error: Division by zero!" << endl;
        }
        break;
    default:
        cout << "Invalid operation!" << endl;
}
```
[↑ Back to Table of Contents](#table-of-contents)
---

## Loops

Loops allow you to execute code repeatedly. C++ has three types of loops.

### 1. for Loop

Best when you know how many times to repeat.

**Syntax:**
```cpp
for (initialization; condition; update) {
    // code to repeat
}
```

**Execution Flow:**
1. **Initialization**: Runs once at the start
2. **Condition**: Checked before each iteration
3. **Code Block**: Executes if condition is true
4. **Update**: Runs after each iteration
5. Repeat steps 2-4 until condition is false

**Example 1: Print 1 to 10**
```cpp
for (int i = 1; i <= 10; i++) {
    cout << i << " ";
}
// Output: 1 2 3 4 5 6 7 8 9 10
```

**Example 2: Multiplication Table**
```cpp
int num;
cout << "Enter a number: ";
cin >> num;

cout << "Multiplication table of " << num << ":" << endl;
for (int i = 1; i <= 10; i++) {
    cout << num << " x " << i << " = " << (num * i) << endl;
}
```

**Example 3: Sum of Numbers**
```cpp
int n, sum = 0;
cout << "Enter a number: ";
cin >> n;

for (int i = 1; i <= n; i++) {
    sum += i;  // sum = sum + i
}
cout << "Sum of first " << n << " numbers: " << sum << endl;
```

**Example 4: Counting Down**
```cpp
for (int i = 10; i >= 1; i--) {
    cout << i << " ";
}
cout << "Blast off!" << endl;
// Output: 10 9 8 7 6 5 4 3 2 1 Blast off!
```

**Example 5: Nested Loops (Pattern)**
```cpp
// Print a square pattern
for (int row = 1; row <= 5; row++) {
    for (int col = 1; col <= 5; col++) {
        cout << "* ";
    }
    cout << endl;
}
// Output:
// * * * * *
// * * * * *
// * * * * *
// * * * * *
// * * * * *
```

### 2. while Loop

Best when you don't know how many times to repeat (condition-based).

**Syntax:**
```cpp
while (condition) {
    // code to repeat
}
```

**Example 1: Basic Counter**
```cpp
int i = 1;
while (i <= 5) {
    cout << i << " ";
    i++;
}
// Output: 1 2 3 4 5
```

**Example 2: User Input Validation**
```cpp
int password;
cout << "Enter password (1234): ";
cin >> password;

while (password != 1234) {
    cout << "Wrong password! Try again: ";
    cin >> password;
}
cout << "Access granted!" << endl;
```

**Example 3: Menu System**
```cpp
int choice = 0;

while (choice != 4) {
    cout << "\n=== Menu ===" << endl;
    cout << "1. Start Game" << endl;
    cout << "2. Load Game" << endl;
    cout << "3. Settings" << endl;
    cout << "4. Exit" << endl;
    cout << "Choice: ";
    cin >> choice;
    
    switch (choice) {
        case 1:
            cout << "Starting game..." << endl;
            break;
        case 2:
            cout << "Loading game..." << endl;
            break;
        case 3:
            cout << "Opening settings..." << endl;
            break;
        case 4:
            cout << "Goodbye!" << endl;
            break;
        default:
            cout << "Invalid choice!" << endl;
    }
}
```

**Example 4: Sum Until Negative**
```cpp
int num, sum = 0;

cout << "Enter numbers (negative to stop):" << endl;
cin >> num;

while (num >= 0) {
    sum += num;
    cin >> num;
}

cout << "Sum: " << sum << endl;
```

### 3. do-while Loop

Similar to while, but **always executes at least once** (checks condition at the end).

**Syntax:**
```cpp
do {
    // code to repeat (runs at least once)
} while (condition);
```

**Example 1: Basic Usage**
```cpp
int i = 1;
do {
    cout << i << " ";
    i++;
} while (i <= 5);
// Output: 1 2 3 4 5
```

**Example 2: Menu (Guaranteed to Show Once)**
```cpp
char choice;

do {
    cout << "\n=== Options ===" << endl;
    cout << "A. Add" << endl;
    cout << "B. Delete" << endl;
    cout << "C. View" << endl;
    cout << "Q. Quit" << endl;
    cout << "Choice: ";
    cin >> choice;
    
    switch (choice) {
        case 'A':
        case 'a':
            cout << "Adding..." << endl;
            break;
        case 'B':
        case 'b':
            cout << "Deleting..." << endl;
            break;
        case 'C':
        case 'c':
            cout << "Viewing..." << endl;
            break;
        case 'Q':
        case 'q':
            cout << "Exiting..." << endl;
            break;
        default:
            cout << "Invalid choice!" << endl;
    }
} while (choice != 'Q' && choice != 'q');
```

**Example 3: Input Validation**
```cpp
int age;

do {
    cout << "Enter your age (1-120): ";
    cin >> age;
    
    if (age < 1 || age > 120) {
        cout << "Invalid age! Please try again." << endl;
    }
} while (age < 1 || age > 120);

cout << "Age accepted: " << age << endl;
```

### Loop Comparison

| Loop Type | When to Use | Minimum Executions |
|-----------|-------------|-------------------|
| `for` | Know exact iterations | 0 |
| `while` | Unknown iterations, condition first | 0 |
| `do-while` | Unknown iterations, run at least once | 1 |

**Choosing the Right Loop:**
```cpp
// for - when you know the count
for (int i = 0; i < 10; i++) { }

// while - checking condition first
while (userInput != "quit") { }

// do-while - must run at least once (like menus)
do {
    showMenu();
} while (choice != 0);
```

---

## Break and Continue

Special statements that control loop execution.

### break Statement

**Purpose:** Immediately **exits** the loop completely.

**Example 1: Exit on Condition**
```cpp
for (int i = 1; i <= 10; i++) {
    if (i == 6) {
        break;  // Stop loop when i equals 6
    }
    cout << i << " ";
}
// Output: 1 2 3 4 5
```

**Example 2: Search in Loop**
```cpp
int numbers[] = {10, 20, 30, 40, 50};
int target = 30;
bool found = false;

for (int i = 0; i < 5; i++) {
    if (numbers[i] == target) {
        cout << "Found " << target << " at index " << i << endl;
        found = true;
        break;  // No need to continue searching
    }
}

if (!found) {
    cout << target << " not found!" << endl;
}
```

**Example 3: Exit on User Command**
```cpp
while (true) {  // Infinite loop
    string command;
    cout << "Enter command (type 'exit' to quit): ";
    cin >> command;
    
    if (command == "exit") {
        cout << "Goodbye!" << endl;
        break;  // Exit the infinite loop
    }
    
    cout << "You entered: " << command << endl;
}
```

**Example 4: break in switch (already seen)**
```cpp
switch (choice) {
    case 1:
        cout << "Option 1" << endl;
        break;  // Prevents fall-through
    case 2:
        cout << "Option 2" << endl;
        break;
}
```

### continue Statement

**Purpose:** **Skips** the rest of current iteration and moves to the next iteration.

**Example 1: Skip Specific Values**
```cpp
for (int i = 1; i <= 10; i++) {
    if (i == 5) {
        continue;  // Skip when i is 5
    }
    cout << i << " ";
}
// Output: 1 2 3 4 6 7 8 9 10 (5 is skipped)
```

**Example 2: Print Only Odd Numbers**
```cpp
for (int i = 1; i <= 10; i++) {
    if (i % 2 == 0) {
        continue;  // Skip even numbers
    }
    cout << i << " ";
}
// Output: 1 3 5 7 9
```

**Example 3: Skip Negative Numbers**
```cpp
int numbers[] = {5, -2, 8, -1, 10, -3, 7};

cout << "Positive numbers: ";
for (int i = 0; i < 7; i++) {
    if (numbers[i] < 0) {
        continue;  // Skip negative numbers
    }
    cout << numbers[i] << " ";
}
// Output: Positive numbers: 5 8 10 7
```

**Example 4: Input Validation**
```cpp
int sum = 0;
for (int i = 0; i < 5; i++) {
    int num;
    cout << "Enter number " << (i+1) << ": ";
    cin >> num;
    
    if (num < 0) {
        cout << "Negative numbers not allowed. Skipping..." << endl;
        continue;  // Skip this iteration
    }
    
    sum += num;
}
cout << "Sum of valid numbers: " << sum << endl;
```

### break vs continue Comparison

```cpp
// Example demonstrating both

cout << "Using break:" << endl;
for (int i = 1; i <= 10; i++) {
    if (i == 6) {
        break;  // Exit loop completely
    }
    cout << i << " ";
}
// Output: 1 2 3 4 5

cout << "\n\nUsing continue:" << endl;
for (int i = 1; i <= 10; i++) {
    if (i == 6) {
        continue;  // Skip only 6
    }
    cout << i << " ";
}
// Output: 1 2 3 4 5 7 8 9 10
```

**Visual Difference:**

| Statement | Effect | Use When |
|-----------|--------|----------|
| `break` | Exits loop entirely | Found what you need, or need to stop |
| `continue` | Skips to next iteration | Need to skip certain values but keep looping |

### Nested Loop Control

```cpp
// break only exits the innermost loop
for (int i = 1; i <= 3; i++) {
    for (int j = 1; j <= 3; j++) {
        if (j == 2) {
            break;  // Only exits inner loop
        }
        cout << i << "," << j << " ";
    }
    cout << endl;
}
// Output:
// 1,1
// 2,1
// 3,1

// continue only affects current loop
for (int i = 1; i <= 3; i++) {
    for (int j = 1; j <= 3; j++) {
        if (j == 2) {
            continue;  // Skip j=2 in inner loop
        }
        cout << i << "," << j << " ";
    }
    cout << endl;
}
// Output:
// 1,1 1,3
// 2,1 2,3
// 3,1 3,3
```

---

## Best Practices

### 1. Choosing the Right Control Structure

```cpp
// ✅ Use switch for multiple discrete values
switch (menuChoice) {
    case 1: /* ... */ break;
    case 2: /* ... */ break;
}

// ✅ Use if-else for ranges or complex conditions
if (score >= 90) {
    // ...
} else if (score >= 80) {
    // ...
}

// ✅ Use for loop when iteration count is known
for (int i = 0; i < 10; i++) { }

// ✅ Use while when condition-based
while (userInput != "quit") { }

// ✅ Use do-while for at-least-once execution
do {
    showMenu();
} while (choice != 0);
```

### 2. Always Use Braces

```cpp
// ❌ Dangerous (easy to make mistakes)
if (condition)
    doSomething();

// ✅ Safe and clear
if (condition) {
    doSomething();
}
```

### 3. Avoid Deep Nesting

```cpp
// ❌ Hard to read
if (condition1) {
    if (condition2) {
        if (condition3) {
            // deeply nested code
        }
    }
}

// ✅ Better - early returns
if (!condition1) return;
if (!condition2) return;
if (!condition3) return;
// main code here
```

### 4. Initialize Loop Variables

```cpp
// ✅ Always initialize
for (int i = 0; i < 10; i++) { }

// ❌ Uninitialized variable
int i;
for (i; i < 10; i++) { }  // i has garbage value initially
```

### 5. Avoid Infinite Loops (Unless Intentional)

```cpp
// ❌ Accidental infinite loop
for (int i = 0; i < 10; i--) {  // i decreases!
    // never ends
}

// ✅ Intentional infinite loop with break
while (true) {
    if (exitCondition) {
        break;
    }
}
```

### 6. Use Meaningful Variable Names

```cpp
// ❌ Unclear
for (int i = 0; i < n; i++) { }

// ✅ Clear
for (int studentIndex = 0; studentIndex < totalStudents; studentIndex++) { }

// ✅ Or use range-based for loop
for (auto student : students) { }
```

### 7. Avoid Magic Numbers

```cpp
// ❌ What does 7 mean?
for (int i = 0; i < 7; i++) { }

// ✅ Use constants
const int DAYS_IN_WEEK = 7;
for (int day = 0; day < DAYS_IN_WEEK; day++) { }
```

### 8. break and continue Guidelines

```cpp
// ✅ Use break to exit when found
for (int i = 0; i < size; i++) {
    if (array[i] == target) {
        found = true;
        break;  // No need to continue searching
    }
}

// ✅ Use continue to skip invalid data
for (int i = 0; i < size; i++) {
    if (data[i] < 0) {
        continue;  // Skip negative values
    }
    processData(data[i]);
}
```

---

## Practice Problems

Test your understanding with these exercises:

### Problem 1: Even or Odd Checker
Write a program that asks for a number and tells if it's even or odd.

### Problem 2: Simple Calculator
Create a calculator using switch-case that performs +, -, *, / operations.

### Problem 3: Factorial Calculator
Calculate factorial of a number using a loop. (5! = 5 × 4 × 3 × 2 × 1 = 120)

### Problem 4: Prime Number Checker
Check if a number is prime (only divisible by 1 and itself).

### Problem 5: Pattern Printing
Print the following pattern:
```
*
**
***
****
*****
```

### Problem 6: Number Guessing Game
Create a game where the computer picks a random number (1-100) and the user guesses. Use loops and break/continue appropriately.

---

## Summary

**Decision Making:**
- Use `if-else` for conditions and ranges
- Use `switch-case` for multiple discrete values
- Use ternary operator `? :` for simple conditions

**Loops:**
- `for`: When you know iteration count
- `while`: Condition checked first
- `do-while`: Runs at least once

**Control Statements:**
- `break`: Exit loop completely
- `continue`: Skip current iteration

**Key Takeaways:**
- Always use braces `{}` for clarity
- Initialize variables before loops
- Avoid infinite loops (unless intentional)
- Use meaningful variable names
- Comment complex logic
- Choose the right control structure for the task

With these fundamentals, you can now control the flow of any C++ program! 🚀
