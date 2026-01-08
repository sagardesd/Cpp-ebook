# C++11 Scoped Enum 

**Enumerations (enums)** are a user-defined data type in C++ that consists of a set of named integral constants. They allow programmers to define a type with a restricted set of possible values, making code more readable, self-documenting, and type-safe.

An enum defines a new type and a set of named constants (enumerators) that belong to that type:

```cpp
enum DayOfWeek {
    MONDAY,    // 0
    TUESDAY,   // 1
    WEDNESDAY, // 2
    THURSDAY,  // 3
    FRIDAY,    // 4
    SATURDAY,  // 5
    SUNDAY     // 6
};

DayOfWeek today = WEDNESDAY;  // today has value 2
```

By default, enumerators start at 0 and increment by 1, but you can assign custom values:

```cpp
enum HttpStatus {
    OK = 200,
    NOT_FOUND = 404,
    INTERNAL_ERROR = 500
};
```

Instead of using "magic numbers" or strings scattered throughout your code, enums provide meaningful names for values:

```cpp
// Without enums - unclear and error-prone
int status = 2;  // What does 2 mean?
if (status == 1) {
    // Do something
}

// With enums - clear and maintainable
enum Status { IDLE, RUNNING, STOPPED };
Status status = RUNNING;
if (status == RUNNING) {
    // Do something
}
```

---

Before C++11, C++ used **C-style enums** (also called "unscoped enums" or "plain enums"). While useful, C-style enums used in C++ programming before C++11 have several issues and drawbacks that can lead to bugs, maintenance problems, and poor code quality.

Let's examine each drawback one by one before exploring how C++11 scoped enums solve these problems.

### Problem 1: Scope Issues and Name Conflicts

C-style enums have their enumerators placed in the same scope as the enum itself. This means the enumerator names (like `OFF`, `ON`, `AUTO`) are visible throughout the entire scope where the enum is declared.

Lets look at the below example:

```cpp
#include <iostream>

enum DisplayMode {
    OFF,
    ON,
    AUTO
};

/*
enum PowerState {
    SLEEP,
    OFF,    // ERROR: 'OFF' already declared in DisplayMode
    RUN
};
*/

int main() {
    DisplayMode mode_1 = OFF;                    // Works
    DisplayMode mode_2 = DisplayMode::OFF;       // Also works
    
    // If PowerState was uncommented:
    // PowerState state_1 = OFF;                 // Ambiguous!
    // PowerState state_2 = PowerState::OFF;     // Still ambiguous!
    
    return 0;
}
```

The enumerators `OFF`, `ON`, and `AUTO` are visible throughout the file. If we try to create another enum with a duplicate enumerator name (like `OFF` in `PowerState`), the compiler throws an error because `OFF` is already defined in the same scope.

**Workaround (Ugly):**

You could wrap enums in namespaces, but this is verbose and cumbersome:

```cpp
namespace Display {
    enum Mode { OFF, ON, AUTO };
}

namespace Power {
    enum State { SLEEP, OFF, RUN };
}

int main() {
    Display::Mode mode = Display::OFF;
    Power::State state = Power::OFF;
}
```

---

### Problem 2: Non-Fixed Underlying Type

The underlying type of a C-style enum is implementation-defined. The compiler optimizes the storage type based on the enum's content, which can lead to portability and interoperability issues.

**Example:**

```cpp
#include <iostream>
#include <type_traits>
#include <cstdint>

// 1. Standard C-style enum (Usually defaults to int)
enum Standard { A, B };

// 2. Large C-style enum (Forces a 64-bit underlying type)
enum Huge { 
    BigValue = 0xFFFFFFFFFFFFFFFFULL 
};

// 3. Explicitly fixed type (Using C++11 fixed underlying type syntax)
enum Small : std::uint8_t { 
    Low, 
    High 
};

// Helper template to print the name and size of the underlying type
template <typename T>
void printUnderlyingTypeInfo(const char* enumName) {
    using Underlying = std::underlying_type_t<T>;
    
    std::cout << "Enum [" << enumName << "]:\n";
    std::cout << "  - Size: " << sizeof(Underlying) << " byte(s)\n";
    
    if (std::is_signed_v<Underlying>)
        std::cout << "  - Signed: Yes\n";
    else
        std::cout << "  - Signed: No\n";
    std::cout << "--------------------------\n";
}

int main() {
    printUnderlyingTypeInfo<Standard>("Standard");
    printUnderlyingTypeInfo<Huge>("Huge");
    printUnderlyingTypeInfo<Small>("Small");

    return 0;
}
```

**Output:**

```
Enum [Standard]:
  - Size: 4 byte(s)
  - Signed: No
--------------------------
Enum [Huge]:
  - Size: 8 byte(s)
  - Signed: No
--------------------------
Enum [Small]:
  - Size: 1 byte(s)
  - Signed: No
--------------------------
```

**Note:** This example uses C++17 type traits (`std::underlying_type_t` and `std::is_signed_v`) to identify the underlying storage size.

The size varies based on the enum values. This causes issues in:
- Network communication protocols (where fixed sizes are expected)
- Binary file formats
- Interfacing with hardware or external libraries
- Cross-platform compatibility

While C++11 allows specifying a fixed underlying type (as shown with `Small`), it's not enforced by default for C-style enums.

---

### Problem 3: Implicit Conversion to int (Type Safety Issues)

C-style enums can be implicitly converted to integers, breaking type safety and potentially causing undefined behavior.

**Example:**

```cpp
#include <iostream>

enum Color { Red, Green, Blue };

void draw(Color c) {
    std::cout << "draw(Color) called\n";
}

void draw(int x) {
    std::cout << "draw(int) called with " << x << "\n";
}

int main() {
    Color c = Red;
    
    draw(c);   // OK: calls draw(Color)
    
    draw(42);  // OK: calls draw(int), but 42 is not a valid Color
    
    int n = Green;  // Implicit conversion from Color to int
    draw(n);   // OK: calls draw(int), even though n came from Color
    
    // Even worse:
    Color invalid = static_cast<Color>(999);  // Compiles! Undefined behavior!
    
    return 0;
}
```

**Output:**

```
draw(Color) called
draw(int) called with 42
draw(int) called with 1
```

Enums are not type-safe. You can:
- Assign arbitrary integers to enum variables
- Implicitly convert enums to integers
- Lose the semantic meaning of the enum type
- Accidentally pass wrong values without compiler warnings

---

### Problem 4: No Forward Declaration

C-style enums cannot be forward declared (in C++03 and earlier) because the compiler needs to know the underlying type to determine the enum's size.

**Example:**

```cpp
#include <iostream>

// This WILL NOT compile in C++03:
// enum Color;  // Error: cannot forward declare

// You must provide the full definition:
enum Color { Red, Green, Blue };

class Widget {
    Color favoriteColor;  // Must have full enum definition above
public:
    void setColor(Color c);
};

void Widget::setColor(Color c) {
    favoriteColor = c;
}

int main() {
    Widget w;
    w.setColor(Red);
    return 0;
}
```

**Why is this a problem?**

1. **Compilation dependencies**: Every file that includes a header with an enum must see the complete definition, even if it only needs to know the enum exists. This increases compilation time and creates tight coupling.

2. **Circular dependencies**: If two classes need to reference each other's enums, you can't forward declare, leading to difficult header organization.

3. **Reduced encapsulation**: You can't hide the enum values in the header; everything is exposed.

**Example showing the circular dependency problem:**

```cpp
// device.h
#ifndef DEVICE_H
#define DEVICE_H

// Cannot forward declare!
// enum PowerState;  // Error!

// Must include full definition
enum PowerState { SLEEP, OFF, RUN };

class Device {
    PowerState state;
public:
    void setState(PowerState s);
};

#endif
```

Compare this to classes/structs where forward declaration works fine:

```cpp
// device.h
#ifndef DEVICE_H
#define DEVICE_H

class PowerManager;  // Forward declaration works!

class Device {
    PowerManager* manager;  // Only need pointer/reference
public:
    void setManager(PowerManager* pm);
};

#endif
```

**Why forward declaration fails for C-style enums:**

The compiler must know the size of the enum to allocate memory for enum variables. Since the underlying type is implementation-defined and depends on the enum's values (as shown in Problem 2), the compiler needs to see all the enumerators to determine the size.

---

## C++11 Scoped Enums (`enum class`) - The Solution

C++11 introduced **scoped enums** (also called **strongly-typed enums**) using the `enum class` or `enum struct` syntax. 

### Basic Syntax

```cpp
// Basic scoped enum syntax
enum class EnumName {
    Enumerator1,
    Enumerator2,
    Enumerator3
};

// With explicit underlying type
enum class EnumName : UnderlyingType {
    Enumerator1,
    Enumerator2,
    Enumerator3
};

// Both 'enum class' and 'enum struct' are equivalent
enum class Mode { A, B, C };   // More commonly used
enum struct Mode { A, B, C };  // Exactly the same behavior
```

**Examples:**

```cpp
enum class DisplayMode {
    OFF,
    ON,
    AUTO
};

enum class PowerState {
    SLEEP,
    OFF,    // No conflict! Different scope
    RUN
};

// With explicit underlying type
enum class Priority : std::uint8_t {
    LOW = 0,
    MEDIUM = 1,
    HIGH = 2
};

// With custom values
enum class ErrorCode : int {
    SUCCESS = 0,
    FILE_NOT_FOUND = 404,
    INTERNAL_ERROR = 500
};
```

The new C++11 scoped enums solve all the problems we have discussed above when we use c-style enums.

Lets now look at how its solving these problems and why you should start using the C++11 scoped enums.


---

### Solution 1: Proper Scoping - No More Name Conflicts

Scoped enums keep their enumerators within the enum's scope, preventing naming conflicts.

```cpp
#include <iostream>

enum class DisplayMode {
    OFF,
    ON,
    AUTO
};

enum class PowerState {
    SLEEP,
    OFF,    // No conflict with DisplayMode::OFF
    RUN
};

int main() {
    // Must use scope resolution operator
    DisplayMode mode = DisplayMode::OFF;
    PowerState state = PowerState::OFF;
    
    // This won't compile:
    // DisplayMode bad = OFF;  // Error: 'OFF' not found in this scope
    
    std::cout << "Code compiles successfully!\n";
    
    return 0;
}
```

**Benefits:**
- No naming conflicts between different enums
- More explicit and readable code
- Clearer intent and namespace pollution prevention

---

### Solution 2: Fixed Underlying Type

Scoped enums have a default underlying type of `int`, and you can explicitly specify any integral type you want. This ensures consistency across platforms.


```cpp
#include <iostream>
#include <type_traits>
#include <cstdint>

// Default underlying type is int
enum class Status {
    OK,
    ERROR,
    PENDING
};

// Explicitly specify underlying type
enum class Priority : std::uint8_t {
    LOW,
    MEDIUM,
    HIGH
};

enum class LargeValue : std::uint64_t {
    HUGE = 0xFFFFFFFFFFFFFFFFULL
};

template <typename T>
void printEnumInfo(const char* enumName) {
    using Underlying = std::underlying_type_t<T>;
    
    std::cout << "Enum [" << enumName << "]:\n";
    std::cout << "  - Size: " << sizeof(Underlying) << " byte(s)\n";
    std::cout << "  - Signed: " << (std::is_signed_v<Underlying> ? "Yes" : "No") << "\n";
    std::cout << "--------------------------\n";
}

int main() {
    printEnumInfo<Status>("Status");
    printEnumInfo<Priority>("Priority");
    printEnumInfo<LargeValue>("LargeValue");
    
    return 0;
}
```

**Output:**

```
Enum [Status]:
  - Size: 4 byte(s)
  - Signed: Yes
--------------------------
Enum [Priority]:
  - Size: 1 byte(s)
  - Signed: No
--------------------------
Enum [LargeValue]:
  - Size: 8 byte(s)
  - Signed: No
--------------------------
```

**Benefits:**
- Predictable size across platforms
- Safe for serialization and network protocols
- Memory-efficient when using smaller types like `uint8_t`

---

### Solution 3: No Implicit Conversion - Type Safety

**The key feature of C++11 scoped enums:** They do NOT allow implicit conversion to integers or other types. This provides strong type safety and prevents many common programming errors.

**The Rule:** 
- **No implicit conversion** from scoped enum to int or any other type
- **Must use `static_cast`** for explicit conversion when needed
- This forces programmers to be explicit about their intentions

```cpp
#include <iostream>

enum class Color {
    Red,
    Green,
    Blue
};

enum class Size {
    Small,
    Medium,
    Large
};

void draw(Color c) {
    std::cout << "draw(Color) called\n";
}

void draw(int x) {
    std::cout << "draw(int) called with " << x << "\n";
}

int main() {
    Color c = Color::Red;
    
    draw(c);   // OK: calls draw(Color)
    draw(42);  // OK: calls draw(int)
    
    // ===== These WON'T compile (No implicit conversion) =====
    // int n = Color::Green;           // Error: cannot convert Color to int
    // int m = c;                      // Error: cannot convert Color to int
    // Color c2 = 1;                   // Error: cannot convert int to Color
    // Size s = Color::Red;            // Error: cannot convert Color to Size
    // if (c == 0) { }                 // Error: cannot compare Color with int
    // bool b = c;                     // Error: cannot convert Color to bool
    
    // ===== Must use static_cast for explicit conversion =====
    
    // Enum to int
    int value = static_cast<int>(Color::Green);
    std::cout << "Green value: " << value << "\n";
    
    // Enum to underlying type
    auto underlying_value = static_cast<std::underlying_type_t<Color>>(c);
    std::cout << "Red underlying value: " << underlying_value << "\n";
    
    // Int to enum (use with caution - no validation!)
    Color c3 = static_cast<Color>(2);  // Becomes Color::Blue
    
    // Enum to another enum type (requires double cast)
    Size s = static_cast<Size>(static_cast<int>(Color::Medium));
    
    // Comparison between enums (same type only)
    Color c4 = Color::Red;
    if (c == c4) {  // OK: same enum type
        std::cout << "Colors match!\n";
    }
    
    // if (c == Size::Small) { }  // Error: cannot compare different enum types
    
    return 0;
}
```

**Output:**

```
draw(Color) called
draw(int) called with 42
Green value: 1
Red underlying value: 0
Colors match!
```

**Why This Matters - Comparison with C-Style Enums:**

```cpp
#include <iostream>

// C-style enum (OLD - implicit conversion allowed)
enum OldColor { OLD_RED, OLD_GREEN, OLD_BLUE };

// Scoped enum (NEW - no implicit conversion)
enum class NewColor { RED, GREEN, BLUE };

void processColor(int value) {
    std::cout << "Processing value: " << value << "\n";
}

int main() {
    OldColor oldColor = OLD_RED;
    NewColor newColor = NewColor::RED;
    
    // C-style enum problems:
    processColor(oldColor);           // Compiles! Implicit conversion
    int x = oldColor;                 // Compiles! Implicit conversion
    if (oldColor == 0) { }            // Compiles! Can compare with int
    bool b = oldColor;                // Compiles! Converts to bool
    OldColor bad = 999;               // Compiles! Invalid value allowed
    
    // Scoped enum - all these are errors:
    // processColor(newColor);        // Error: no implicit conversion
    // int y = newColor;               // Error: no implicit conversion
    // if (newColor == 0) { }          // Error: cannot compare with int
    // bool c = newColor;              // Error: no implicit conversion
    // NewColor bad2 = 999;            // Error: cannot convert int to NewColor
    
    // Must be explicit with scoped enums:
    processColor(static_cast<int>(newColor));  // OK: explicit intent
    int y = static_cast<int>(newColor);        // OK: explicit conversion
    
    return 0;
}
```

**Benefits of No Implicit Conversion:**
- **Type safety**: Prevents accidental mixing of unrelated enum types
- **Compiler protection**: Catches errors at compile time instead of runtime
- **Explicit intent**: Forces you to be clear about conversions
- **Prevents invalid values**: Can't accidentally assign random integers
- **More maintainable**: Clear what the code is doing
- **Prevents logic errors**: Can't accidentally compare enums with integers

---

### Solution 4: Forward Declaration Support

Scoped enums can be forward declared because they have a known underlying type (default `int` or explicitly specified).

**Example:**

```cpp
// device.h
#ifndef DEVICE_H
#define DEVICE_H

// Forward declaration works!
enum class PowerState;
enum class DisplayMode : unsigned char;  // With explicit type

class Device {
    PowerState* state;        // Pointer to forward-declared enum
    DisplayMode* display;      // Pointer to forward-declared enum
public:
    void setState(PowerState s);
    void setDisplay(DisplayMode d);
};

#endif
```

```cpp
// device.cpp
#include "device.h"

// Full definitions in implementation file
enum class PowerState {
    SLEEP,
    OFF,
    RUN
};

enum class DisplayMode : unsigned char {
    OFF,
    ON,
    AUTO
};

void Device::setState(PowerState s) {
    // Implementation
}

void Device::setDisplay(DisplayMode d) {
    // Implementation
}
```

**Benefits:**
- Reduces compilation dependencies
- Enables better header organization
- Solves circular dependency issues
- Faster compilation times
- Better encapsulation

---

Here's a side-by-side comparison showing all the differences:

```cpp
#include <iostream>
#include <type_traits>

// ========== C-STYLE ENUM ==========
enum OldColor {
    OLD_RED,
    OLD_GREEN,
    OLD_BLUE
};

// ========== SCOPED ENUM ==========
enum class NewColor {
    RED,
    GREEN,
    BLUE
};

void processOldColor(OldColor c) {
    std::cout << "Old color value: " << c << "\n";
}

void processNewColor(NewColor c) {
    // Must explicitly cast to print value
    std::cout << "New color value: " << static_cast<int>(c) << "\n";
}

int main() {
    // ===== C-style enum usage =====
    OldColor old1 = OLD_RED;          // Works
    OldColor old2 = OldColor::OLD_RED; // Also works
    
    int oldVal = OLD_GREEN;            // Implicit conversion - BAD!
    processOldColor(old1);
    
    // ===== Scoped enum usage =====
    NewColor new1 = NewColor::RED;     // Must use scope
    // NewColor new2 = RED;            // ERROR: RED not in scope
    
    // int newVal = NewColor::GREEN;   // ERROR: no implicit conversion
    int newVal = static_cast<int>(NewColor::GREEN);  // Must be explicit
    processNewColor(new1);
    
    // ===== Size comparison =====
    std::cout << "\nSize comparison:\n";
    std::cout << "sizeof(OldColor): " << sizeof(OldColor) << " bytes\n";
    std::cout << "sizeof(NewColor): " << sizeof(NewColor) << " bytes\n";
    
    // ===== Underlying type =====
    std::cout << "\nUnderlying types:\n";
    std::cout << "OldColor is signed: " 
              << std::is_signed_v<std::underlying_type_t<OldColor>> << "\n";
    std::cout << "NewColor is signed: " 
              << std::is_signed_v<std::underlying_type_t<NewColor>> << "\n";
    
    return 0;
}
```

---

## Best Practices and Recommendations

### When to Use Scoped Enums

**Always prefer `enum class` over plain `enum` in modern C++ code** unless you have a specific reason not to.

Use scoped enums when:
- You want type safety and explicit scoping
- Working with APIs, serialization, or network protocols
- You need forward declarations
- Multiple enums might have similar enumerator names
- Writing new code (C++11 and later)

### When C-Style Enums Might Be Acceptable

- Legacy code that you cannot modify
- When you explicitly want implicit conversion (rare cases)
- When working with C APIs that expect C-style enums

### Syntax Variations

Both `enum class` and `enum struct` are equivalent:

```cpp
enum class Mode { A, B, C };    // More common
enum struct Mode { A, B, C };   // Exactly the same
```

### Specifying Underlying Type

```cpp
// Default (int)
enum class Status { OK, ERROR };

// Custom type
enum class TinyEnum : std::uint8_t { A, B, C };
enum class BigEnum : std::uint64_t { HUGE = 0xFFFFFFFF };
```

### Working with Underlying Values

When you need the integer value:

```cpp
enum class Level : int { LOW = 1, MEDIUM = 5, HIGH = 10 };

Level lv = Level::MEDIUM;

// Get underlying value
int value = static_cast<int>(lv);
std::cout << "Level value: " << value << "\n";  // Prints: 5

// Convert integer to enum (be careful!)
Level lv2 = static_cast<Level>(10);
```

---

## Summary

| Feature | C-Style Enum | Scoped Enum (`enum class`) |
|---------|--------------|----------------------------|
| **Scoping** | Enumerators in surrounding scope | Enumerators in enum scope |
| **Name conflicts** | Common problem | No conflicts |
| **Type safety** | Weak (implicit int conversion) | Strong (no implicit conversion) |
| **Underlying type** | Implementation-defined | `int` by default, explicitly specifiable |
| **Forward declaration** | Not possible (C++03) | Supported |
| **Syntax** | `enum Name { ... }` | `enum class Name { ... }` |
| **Access** | `Name` or `EnumName::Name` | `EnumName::Name` only |
| **Usage recommendation** | Legacy code only | Modern C++ (C++11+) |

**Key Takeaway**: Scoped enums (`enum class`) solve all major problems with C-style enums and should be your default choice in modern C++ programming.

---

## Type Traits for Enums

C++11 and later versions provide several type traits in the `<type_traits>` header for working with enums.

These are useful for template metaprogramming and generic code.

### Available Enum Type Traits

```cpp
#include <iostream>
#include <type_traits>
#include <cstdint>

enum OldStyle { A, B, C };

enum class NewStyle : std::uint16_t {
    X = 100,
    Y = 200,
    Z = 300
};

enum class DefaultStyle {
    P, Q, R
};

int main() {
    // ===== 1. std::is_enum =====
    // Checks if a type is an enumeration type
    std::cout << std::boolalpha;
    std::cout << "std::is_enum:\n";
    std::cout << "  OldStyle: " << std::is_enum<OldStyle>::value << "\n";
    std::cout << "  NewStyle: " << std::is_enum<NewStyle>::value << "\n";
    std::cout << "  int: " << std::is_enum<int>::value << "\n";
    std::cout << "  DefaultStyle: " << std::is_enum<DefaultStyle>::value << "\n\n";
    
    // C++17 shorthand
    std::cout << "  OldStyle (v): " << std::is_enum_v<OldStyle> << "\n\n";
    
    // ===== 2. std::underlying_type =====
    // Gets the underlying integer type of an enum
    std::cout << "std::underlying_type:\n";
    
    using OldUnderlying = std::underlying_type<OldStyle>::type;
    using NewUnderlying = std::underlying_type<NewStyle>::type;
    using DefaultUnderlying = std::underlying_type<DefaultStyle>::type;
    
    std::cout << "  OldStyle underlying type size: " 
              << sizeof(OldUnderlying) << " bytes\n";
    std::cout << "  NewStyle underlying type size: " 
              << sizeof(NewUnderlying) << " bytes\n";
    std::cout << "  DefaultStyle underlying type size: " 
              << sizeof(DefaultUnderlying) << " bytes\n\n";
    
    // C++14 shorthand: std::underlying_type_t
    using NewUnderlyingT = std::underlying_type_t<NewStyle>;
    std::cout << "  NewStyle underlying (using _t): " 
              << sizeof(NewUnderlyingT) << " bytes\n\n";
    
    // ===== 3. Checking if underlying type is signed =====
    std::cout << "Is underlying type signed:\n";
    std::cout << "  OldStyle: " 
              << std::is_signed<std::underlying_type_t<OldStyle>>::value << "\n";
    std::cout << "  NewStyle: " 
              << std::is_signed<std::underlying_type_t<NewStyle>>::value << "\n";
    std::cout << "  DefaultStyle: " 
              << std::is_signed<std::underlying_type_t<DefaultStyle>>::value << "\n\n";
    
    // C++17 shorthand
    std::cout << "  NewStyle (v): " 
              << std::is_signed_v<std::underlying_type_t<NewStyle>> << "\n\n";
    
    // ===== 4. Checking if underlying type is unsigned =====
    std::cout << "Is underlying type unsigned:\n";
    std::cout << "  OldStyle: " 
              << std::is_unsigned_v<std::underlying_type_t<OldStyle>> << "\n";
    std::cout << "  NewStyle: " 
              << std::is_unsigned_v<std::underlying_type_t<NewStyle>> << "\n";
    std::cout << "  DefaultStyle: " 
              << std::is_unsigned_v<std::underlying_type_t<DefaultStyle>> << "\n\n";
    
    // ===== 5. std::is_scoped_enum (C++23) =====
    // Note: This requires C++23 support
    #if __cplusplus >= 202302L
    std::cout << "std::is_scoped_enum (C++23):\n";
    std::cout << "  OldStyle: " << std::is_scoped_enum_v<OldStyle> << "\n";
    std::cout << "  NewStyle: " << std::is_scoped_enum_v<NewStyle> << "\n\n";
    #endif
    
    return 0;
}
```

**Output:**

```
std::is_enum:
  OldStyle: true
  NewStyle: true
  int: false
  DefaultStyle: true

  OldStyle (v): true

std::underlying_type:
  OldStyle underlying type size: 4 bytes
  NewStyle underlying type size: 2 bytes
  DefaultStyle underlying type size: 4 bytes

  NewStyle underlying (using _t): 2 bytes

Is underlying type signed:
  OldStyle: false
  NewStyle: false
  DefaultStyle: true

  NewStyle (v): false

Is underlying type unsigned:
  OldStyle: true
  NewStyle: true
  DefaultStyle: false
```

### Practical Example: Generic Enum to String Conversion

```cpp
#include <iostream>
#include <type_traits>
#include <string>

// Generic function to convert any enum to its underlying value
template<typename E>
constexpr auto toUnderlying(E e) noexcept {
    static_assert(std::is_enum_v<E>, "toUnderlying requires an enum type");
    return static_cast<std::underlying_type_t<E>>(e);
}

enum class Status : std::uint8_t {
    IDLE = 0,
    RUNNING = 1,
    PAUSED = 2,
    STOPPED = 3
};

enum class Priority : int {
    LOW = -1,
    NORMAL = 0,
    HIGH = 1
};

int main() {
    Status s = Status::RUNNING;
    Priority p = Priority::HIGH;
    
    std::cout << "Status value: " << toUnderlying(s) << "\n";
    std::cout << "Priority value: " << toUnderlying(p) << "\n";
    
    // Type information
    std::cout << "\nStatus underlying type size: " 
              << sizeof(std::underlying_type_t<Status>) << " byte(s)\n";
    std::cout << "Priority underlying type size: " 
              << sizeof(std::underlying_type_t<Priority>) << " byte(s)\n";
    
    std::cout << "\nStatus is signed: " 
              << std::is_signed_v<std::underlying_type_t<Status>> << "\n";
    std::cout << "Priority is signed: " 
              << std::is_signed_v<std::underlying_type_t<Priority>> << "\n";
    
    return 0;
}
```

**Output:**

```
Status value: 1
Priority value: 1

Status underlying type size: 1 byte(s)
Priority underlying type size: 4 byte(s)

Status is signed: 0
Priority is signed: 1
```

### Summary of Enum Type Traits

| Type Trait | C++ Version | Purpose | Example |
|------------|-------------|---------|---------|
| `std::is_enum<T>` | C++11 | Check if T is an enum | `std::is_enum<Color>::value` |
| `std::is_enum_v<T>` | C++17 | Shorthand for `is_enum` | `std::is_enum_v<Color>` |
| `std::underlying_type<T>` | C++11 | Get underlying type | `std::underlying_type<Color>::type` |
| `std::underlying_type_t<T>` | C++14 | Shorthand for `underlying_type` | `std::underlying_type_t<Color>` |
| `std::is_scoped_enum<T>` | C++23 | Check if enum is scoped | `std::is_scoped_enum_v<Color>` |
| `std::is_signed<T>` | C++11 | Check if type is signed | Works on underlying type |
| `std::is_unsigned<T>` | C++11 | Check if type is unsigned | Works on underlying type |
| `std::is_signed_v<T>` | C++17 | Shorthand for `is_signed` | `std::is_signed_v<int>` |
| `std::is_unsigned_v<T>` | C++17 | Shorthand for `is_unsigned` | `std::is_unsigned_v<uint8_t>` |

### Common Use Cases for Enum Type Traits

1. **Template constraints**: Ensure template parameters are enums
2. **Generic conversions**: Write functions that work with any enum type
3. **Serialization**: Determine the size needed to serialize an enum
4. **Reflection**: Build runtime type information systems
5. **Static assertions**: Enforce enum properties at compile time

```cpp
#include <type_traits>
#include <cstdint>

// Example: Ensure an enum uses a specific underlying type
enum class ErrorCode : std::uint32_t {
    SUCCESS = 0,
    FAILURE = 1
};

static_assert(std::is_enum_v<ErrorCode>, "ErrorCode must be an enum");
static_assert(sizeof(std::underlying_type_t<ErrorCode>) == 4, 
              "ErrorCode must be 4 bytes");
static_assert(std::is_unsigned_v<std::underlying_type_t<ErrorCode>>, 
              "ErrorCode must be unsigned");
```