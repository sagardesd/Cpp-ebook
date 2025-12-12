# RVO: Return Value Optimization and the Rule of 0/3/5

## Return Value Optimization (RVO)

When returning objects from functions, you might expect that temporary objects would be created and then copied or moved. However, modern C++ compilers can optimize this away entirely!

### What is RVO?

**Return Value Optimization (RVO)** is a compiler optimization that eliminates temporary objects when returning values from functions, constructing the return value directly in the caller's memory location.

Before diving into RVO, we need to understand the value catagory **prvalues** (pure rvalues):
(You can refer the Value catagories chapter for more detail to understad various value catagories since C++11)

**Prvalue (pure rvalue)** = A temporary object or value that doesn't have a persistent memory location
- Examples: `Photo{100, 200}`, `5`, `x + y`, function return values
- These are "pure" rvalues because they're truly temporary - about to be created or just created
- Before C++17: prvalues would trigger move operations
- From C++17 onward: prvalues trigger mandatory copy elision (RVO)

### Example: Without RVO

Let's see what would happen without optimization:

```cpp
Photo createPhoto() {
    Photo temp{100, 200};
    return temp;  // Without RVO: copy or move temp to return location
}

int main() {
    Photo myPhoto = createPhoto();  // Without RVO: another copy/move
}
```

**Expected behavior without RVO:**
1. Create `temp` inside `createPhoto()`
2. Copy/move `temp` to a temporary return object
3. Copy/move the return object to `myPhoto`
4. Destroy temporaries

This could involve multiple copy or move operations!

### With RVO: Direct Construction (C++17)

```cpp
Photo createPhoto() {
    return Photo{100, 200};  // Prvalue: mandatory copy elision since C++17
}

int main() {
    Photo myPhoto = createPhoto();
}
```

**C++17 output:**
```
Photo::Photo(int, int) invoked
Destructor invoked
```

**Only ONE constructor call!** The object is constructed directly in `myPhoto`'s memory location. No copy, no move, not even a move constructor call!

**Before C++17:** The move constructor would be called:
```
Photo::Photo(int, int) invoked
Move constructor: Photo(Photo&&) invoked
Destructor invoked
Destructor invoked
```

### Visual Representation of RVO

```
Without RVO (theoretical):
┌─────────────────────────┐
│  createPhoto() stack    │
│  ┌──────────────┐       │
│  │ temp{100,200}│       │
│  └──────┬───────┘       │
│         │ copy/move     │
│         ▼               │
│  ┌──────────────┐       │
│  │return object │       │
│  └──────┬───────┘       │
└─────────┼───────────────┘
          │ copy/move
          ▼
┌─────────────────────────┐
│  main() stack           │
│  ┌──────────────┐       │
│  │   myPhoto    │       │
│  └──────────────┘       │
└─────────────────────────┘

With RVO (C++17):
┌─────────────────────────┐
│  main() stack           │
│  ┌──────────────┐       │
│  │   myPhoto    │◄──────┼─── Constructed directly here!
│  └──────────────┘       │
└─────────────────────────┘
         ▲
         │
    createPhoto() constructs
    the object directly in
    myPhoto's memory location
```

## When Does RVO Apply and When it cannot/won't ?

RVO works in specific scenarios. Let's explore when it applies and when it doesn't.

### Case 1: Returning a Temporary (Prvalue)

```cpp
Photo createPhoto() {
    return Photo{100, 200};  // Prvalue: RVO applies in C++17!
}
```

**C++17 and later output:**
```
Photo::Photo(int, int) invoked
Destructor invoked
```

**RVO applies (mandatory since C++17)** - Direct construction, no copy, no move!

**Before C++17:** This would have called the move constructor:
```
Photo::Photo(int, int) invoked
Move constructor: Photo(Photo&&) invoked
Destructor invoked
Destructor invoked
```

**Key point:** `Photo{100, 200}` is a **prvalue** (pure rvalue) - a temporary being created. Since C++17, the compiler is **required** to perform copy elision for prvalues, constructing the object directly in the caller's location.

### Case 2: Returning a Single Local Variable (NRVO)

```cpp
Photo createPhoto() {
    Photo temp{100, 200};
    // ... do some work with temp ...
    return temp;  // Named RVO (NRVO) may apply
}
```

**Note:** This is **Named Return Value Optimization (NRVO)**. In C++17, NRVO is **not mandatory** but most compilers still perform it. You might see:

```
Photo::Photo(int, int) invoked
Destructor invoked
```

Or with some compilers/flags:
```
Photo::Photo(int, int) invoked
Move constructor: Photo(Photo&&) invoked
Destructor invoked
Destructor invoked
```

### Case 3: Returning Different Objects Based on Condition

```cpp
Photo createPhoto(bool highRes) {
    if (highRes) {
        Photo temp1{1920, 1080};
        return temp1;  // RVO does NOT apply!
    } else {
        Photo temp2{640, 480};
        return temp2;   // RVO does NOT apply!
    }
}

int main() {
    Photo myPhoto = createPhoto(true);
}
```

**Output:**
```
Photo::Photo(int, int) invoked
Move constructor: Photo(Photo&&) invoked
Destructor invoked
Destructor invoked
```

**RVO does NOT apply** because the compiler can't determine at compile time which object will be returned. The **move constructor is used** instead!

### Case 4: Returning Function Parameters

```cpp
Photo processPhoto(Photo input) {
    // ... process input ...
    return input;  // RVO does NOT apply!
}

int main() {
    Photo original{100, 200};
    Photo processed = processPhoto(original);
}
```

**Output:**
```
Photo::Photo(int, int) invoked
Copy Constructor invoked: Photo(const Photo&)
Move constructor: Photo(Photo&&) invoked
Destructor invoked
Destructor invoked
Destructor invoked
```

**RVO does NOT apply** to function parameters. The **move constructor is used** when returning.

### Case 5: Returning with std::move (Anti-pattern!)

```cpp
Photo createPhoto() {
    Photo temp{100, 200};
    return std::move(temp);  // DON'T DO THIS! Prevents RVO!
}
```

**Output:**
```
Photo::Photo(int, int) invoked
Move constructor: Photo(Photo&&) invoked
Destructor invoked
Destructor invoked
```

**Using `std::move` on return values PREVENTS RVO!** This is an anti-pattern. The compiler would have optimized this, but `std::move` forces a move operation.

**Rule:** Never use `std::move` on return values when returning local variables.

## Why We Still Need Move Semantics

Even with C++17's mandatory RVO for prvalues, we still need move constructors and move assignment operators. **RVO and move semantics solve DIFFERENT problems!**

### Understanding the Difference

```
┌─────────────────────────────────────────────────────────────┐
│ RVO solves: The cost of returning PRVALUES                  │
│ Move constructor solves: The cost of moving EXISTING objects│
│ Move assignment solves: The cost of REASSIGNING objects     │
└─────────────────────────────────────────────────────────────┘
```

### Problem 1: RVO Only Works for Prvalues

**✅ RVO handles this:**
```cpp
Photo make() {
    return Photo{100, 200};  // Prvalue → RVO: constructed directly in caller
}

Photo p = make();  // Only ONE constructor call!
```

**RVO cannot handle this:**
```cpp
Photo a{100, 200};
Photo b = std::move(a);  // NEED move constructor!
```

Here, `a` is a **real existing object in memory**. RVO doesn't apply because:
- We're not returning from a function
- `a` is an lvalue, not a prvalue
- We want to transfer resources from an existing object

**Without move constructor:** This would call the copy constructor (expensive deep copy)!

### Problem 2: Move Assignment - Reassigning Existing Objects

RVO applies **only during construction**. Move assignment handles reassignment when the object already exists.

```cpp
Photo a{100, 200};
Photo b{640, 480};

a = std::move(b);     // NEED move assignment operator!
```

**Why RVO doesn't apply:**
- No construction happening
- `a` already exists in memory
- We're **overwriting** an existing object
- Need to clean up `a`'s old resources first, then steal from `b`

**Without move assignment:** This would call the copy assignment operator (expensive)!

### Problem 3: Containers Rely Heavily on Move Constructors

Standard library containers like `std::vector` **cannot use RVO** for internal operations.

#### Example: Vector Growth

```cpp
std::vector<Photo> photos;
photos.push_back(Photo{100, 200});  // Move constructor needed!

// When vector grows:
photos.reserve(100);
```

**What happens during vector reallocation:**
```
Old storage:                    New storage:
┌─────────┐                    ┌─────────┐
│ Photo 1 │ ─── move ────────> │ Photo 1 │
├─────────┤                    ├─────────┤
│ Photo 2 │ ─── move ────────> │ Photo 2 │
├─────────┤                    ├─────────┤
│ Photo 3 │ ─── move ────────> │ Photo 3 │
└─────────┘                    ├─────────┤
                               │   ...   │
                               └─────────┘
```

**Steps:**
1. Allocate larger block
2. **Move construct** each element into new block (move constructor!)
3. Destroy old elements

**RVO cannot help** because:
- Elements already exist in the old storage
- We're moving existing objects, not returning prvalues
- This is a runtime operation based on vector size

**Without move constructors:** Every reallocation would **copy** all elements (extremely slow for large objects)!

#### More Container Examples

```cpp
std::vector<Photo> photos;

// 1. push_back with temporary
photos.push_back(Photo{100, 200});    
// - Prvalue → RVO might help in some cases
// - But vector still needs move constructor to store it

// 2. push_back with existing object
Photo temp{640, 480};
photos.push_back(std::move(temp));    
// - NEED move constructor (RVO doesn't apply)

// 3. Sorting
std::sort(photos.begin(), photos.end());
// - Uses move operations to shuffle elements
// - NEED move constructor and move assignment

// 4. Vector assignment
std::vector<Photo> vec1, vec2;
vec1 = std::move(vec2);
// - NEED move assignment for vector itself
```

### Problem 4: Generic Code and Templates Need Moves

Templates work with many types and cannot rely on RVO for all scenarios.

```cpp
template<typename T>
T make_twice(T x) {
    return x;    // Named variable, NOT a prvalue!
}

Photo p{100, 200};
Photo result = make_twice(p);  // NEED move or copy constructor
```

**Why RVO doesn't apply:**
- `x` is a **named object** (lvalue)
- NRVO (Named RVO) is **not guaranteed**
- The compiler may or may not optimize this
- Move constructor is the fallback

### Problem 5: NRVO is Not Guaranteed

When returning a named local variable, NRVO **may** apply, but it's **not mandatory**.

```cpp
Photo createPhoto() {
    Photo temp{100, 200};
    // ... do work ...
    return temp;   // NRVO: compiler *may* optimize
}
```

**Possible outcomes:**

**Best case (NRVO applies):**
```
Photo::Photo(int, int) invoked
Destructor invoked
```

**Without NRVO (move constructor used):**
```
Photo::Photo(int, int) invoked
Move constructor: Photo(Photo&&) invoked
Destructor invoked
Destructor invoked
```

**Without move constructor (only copy available):**
```
Photo::Photo(int, int) invoked
Copy Constructor invoked: Photo(const Photo&)
Destructor invoked
Destructor invoked
```

### Problem 6: Conditional Returns Cannot Use RVO

```cpp
Photo createPhoto(bool highRes) {
    Photo small{640, 480};
    Photo large{1920, 1080};
    return highRes ? large : small;  // RVO cannot optimize!
}
```

**Why RVO fails:**
- Compiler can't determine at compile time which object is returned
- Both `small` and `large` are lvalues
- **Move constructor is used** as fallback

### Problem 7: Algorithms and STL Operations

```cpp
// Swapping
Photo a{100, 200}, b{640, 480};
std::swap(a, b);  // Uses move constructor and move assignment!

// Moving into data structures
std::map<int, Photo> photoMap;
Photo temp{100, 200};
photoMap[1] = std::move(temp);  // NEED move assignment!

// Returning from algorithms
auto it = std::find(photos.begin(), photos.end(), target);
Photo found = std::move(*it);  // NEED move constructor!
```

### Summary: Different Problems, Different Solutions

| Scenario | Solution | Why RVO Doesn't Help |
|----------|----------|---------------------|
| `return Photo{};` | ✅ RVO (C++17) | N/A - RVO applies! |
| `Photo b = std::move(a);` | Move constructor | `a` is existing object, not prvalue |
| `a = std::move(b);` | Move assignment | Reassignment, not construction |
| `vector::push_back()` | Move constructor | Storing existing objects |
| `vector` reallocation | Move constructor | Moving existing elements |
| `return namedVar;` | Move constructor | NRVO not guaranteed |
| `return cond ? a : b;` | Move constructor | Runtime decision, lvalues |
| `std::swap(a, b)` | Move ctor + assignment | Operating on existing objects |

### The Complete Picture

```cpp
// 1. RVO handles this perfectly (C++17+)
Photo p1 = Photo{100, 200};           // ✅ RVO

// 2. These ALL need move semantics
Photo a{100, 200};
Photo b = std::move(a);                // ❌ No RVO → move constructor

Photo c{640, 480};
b = std::move(c);                      // ❌ No RVO → move assignment

std::vector<Photo> photos;
photos.push_back(std::move(b));        // ❌ No RVO → move constructor
photos.reserve(100);                   // ❌ No RVO → move constructor (realloc)

std::sort(photos.begin(), photos.end()); // ❌ No RVO → move operations
```

**Key Insight:** RVO eliminates moves during **prvalue return**, but the vast majority of move operations happen in **other contexts** where RVO cannot apply. Move semantics are essential for efficient C++ code!

## Summary: RVO Rules

| Scenario | Value Category | RVO Applies? | Fallback |
|----------|---------------|-------------|----------|
| `return Photo{...};` | Prvalue | ✅ Yes (mandatory C++17) | N/A |
| `Photo x{...}; return x;` | Lvalue | ⚠️ Maybe (NRVO, not mandatory) | Move constructor |
| `return condition ? x : y;` | Lvalue | ❌ No | Move constructor |
| `return parameter;` | Lvalue | ❌ No | Move constructor |
| `return std::move(x);` | Xvalue | ❌ No (prevents RVO!) | Move constructor |

**Key Takeaway:** 
- **C++17 and later:** RVO is **mandatory** for prvalues (pure rvalues) - zero copies, zero moves
- **Before C++17:** Prvalues would use move constructor
- Move semantics are still essential as a fallback when RVO can't be applied (lvalues, conditionals, etc.)

## The Rule of Zero, Three, and Five

Now that we understand copy and move semantics, let's discuss best practices for implementing special member functions.

### Special Member Functions

C++ has six special member functions that the compiler can generate automatically:

1. **Default constructor:** `Photo()`
2. **Destructor:** `~Photo()`
3. **Copy constructor:** `Photo(const Photo&)`
4. **Copy assignment operator:** `Photo& operator=(const Photo&)`
5. **Move constructor:** `Photo(Photo&&)`  *(C++11)*
6. **Move assignment operator:** `Photo& operator=(Photo&&)`  *(C++11)*

### Rule of Zero

**If your class doesn't directly manage resources, don't define any special member functions.**

```cpp
// Good example: Rule of Zero
class Photo {
public:
    Photo(int w, int h) : width(w), height(h), data(w * h) {}
    
    // No destructor, no copy/move operations defined!
    // Compiler generates them correctly.
    
private:
    int width;
    int height;
    std::vector<int> data;  // std::vector manages memory for us
};
```

**Why this works:**
- `std::vector` already handles memory management correctly
- The compiler-generated special members correctly copy/move the `std::vector`
- Less code to write and maintain
- No chance of getting it wrong!

**When to use:** Whenever possible! Use standard library containers (`std::vector`, `std::string`, `std::unique_ptr`, etc.) instead of raw pointers.

### Rule of Three (Pre-C++11)

**If you define any one of these three, you should probably define all three:**

1. Destructor
2. Copy constructor
3. Copy assignment operator

```cpp
// Rule of Three example
class Photo {
public:
    Photo(int w, int h) 
        : width(w), height(h), data(new int[w * h]) {}
    
    // 1. Destructor
    ~Photo() {
        delete[] data;
    }
    
    // 2. Copy constructor
    Photo(const Photo& other)
        : width(other.width), height(other.height),
          data(new int[width * height]) {
        std::copy(other.data, other.data + width * height, data);
    }
    
    // 3. Copy assignment operator
    Photo& operator=(const Photo& other) {
        if (this != &other) {
            delete[] data;
            width = other.width;
            height = other.height;
            data = new int[width * height];
            std::copy(other.data, other.data + width * height, data);
        }
        return *this;
    }
    
private:
    int width;
    int height;
    int* data;  // Raw pointer: we manage the memory!
};
```

**Why all three?**
- If you need a destructor, you're managing a resource
- If you're managing a resource, the default copy operations will be wrong (shallow copy)
- You need to implement deep copy semantics

### Rule of Five (C++11 and later)

**If you define any one of the five operations below, you should probably define all five:**

1. Destructor
2. Copy constructor
3. Copy assignment operator
4. Move constructor  *(new in C++11)*
5. Move assignment operator  *(new in C++11)*

```cpp
// Rule of Five example
class Photo {
public:
    Photo(int w, int h) 
        : width(w), height(h), data(new int[w * h]) {}
    
    // 1. Destructor
    ~Photo() {
        delete[] data;
    }
    
    // 2. Copy constructor
    Photo(const Photo& other)
        : width(other.width), height(other.height),
          data(new int[width * height]) {
        std::copy(other.data, other.data + width * height, data);
    }
    
    // 3. Copy assignment operator
    Photo& operator=(const Photo& other) {
        if (this != &other) {
            delete[] data;
            width = other.width;
            height = other.height;
            data = new int[width * height];
            std::copy(other.data, other.data + width * height, data);
        }
        return *this;
    }
    
    // 4. Move constructor
    Photo(Photo&& other) noexcept
        : width(other.width), height(other.height), data(other.data) {
        other.data = nullptr;
        other.width = 0;
        other.height = 0;
    }
    
    // 5. Move assignment operator
    Photo& operator=(Photo&& other) noexcept {
        if (this != &other) {
            delete[] data;
            width = other.width;
            height = other.height;
            data = other.data;
            other.data = nullptr;
            other.width = 0;
            other.height = 0;
        }
        return *this;
    }
    
private:
    int width;
    int height;
    int* data;
};
```

**Why add move operations?**
- Without them, moving will fall back to copying (inefficient!)
- Move operations provide significant performance improvements
- They're expected by modern C++ code (containers, algorithms)

**Note:** Mark move operations as `noexcept` when possible - this allows standard containers to use them more aggressively for optimization.

## Quick Decision Guide

```
Do you directly manage resources (raw pointers, file handles, etc.)?
│
├─ NO  → Rule of Zero
│        Use std::vector, std::string, std::unique_ptr, etc.
│        Let the compiler generate everything.
│
└─ YES → Rule of Five
         Implement all five special member functions.
         (Or better yet: refactor to use Rule of Zero!)
```

### Common Mistake: Rule of Three/Four

```cpp
// Bad: Defined destructor and copy operations, but no move operations
class Photo {
public:
    ~Photo() { delete[] data; }
    Photo(const Photo& other) { /* ... */ }
    Photo& operator=(const Photo& other) { /* ... */ }
    
    // Missing move constructor and move assignment!
    // Moving will fall back to expensive copying!
private:
    int* data;
};
```

**Problem:** This class can't be moved efficiently. Any attempt to move will result in copying.

**Solution:** Either add move operations (Rule of Five) or use RAII types (Rule of Zero).

## Best Practices Summary

1. **Prefer Rule of Zero** - Use standard library types that manage resources for you
2. **If you must manage resources directly, follow Rule of Five** - Implement all five special member functions
3. **Mark move operations as `noexcept`** - Enables better optimizations in standard containers
4. **Trust RVO** - Don't use `std::move` on return values of local variables
5. **Test your special member functions** - Easy to get wrong, especially self-assignment and move operations

## Complete Example: Comparing All Three Rules

### Rule of Zero (Preferred)
```cpp
class Photo {
public:
    Photo(int w, int h) : width(w), height(h), data(w * h) {}
    // That's it! Compiler handles everything correctly.
private:
    int width, height;
    std::vector<int> data;
};
```

### Rule of Five (When Necessary)
```cpp
class Photo {
public:
    Photo(int w, int h);
    ~Photo();
    Photo(const Photo&);
    Photo& operator=(const Photo&);
    Photo(Photo&&) noexcept;
    Photo& operator=(Photo&&) noexcept;
private:
    int width, height;
    int* data;  // Raw resource
};
```

**Rule of thumb:** If you can use Rule of Zero, do it. It's simpler, safer, and less error-prone!
