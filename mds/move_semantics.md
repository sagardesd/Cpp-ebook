# Move Semantics - rvalues and Move Constructors

## Understanding the Problem

### The Inefficiency of Copying

Let's start with a basic `Photo` class that manages dynamic memory:

```cpp
#include <iostream>

class Photo {
    public:
        Photo(int width, int height);
        Photo(const Photo& other);              // Copy constructor
        Photo& operator=(const Photo& other);   // Copy assignment operator
        ~Photo();
    private:
        int width;
        int height;
        int* data;
};

Photo::Photo(int width, int height): 
    width(width), 
    height(height), 
    data(new int[width * height]) {
    std::cout << "Photo::Photo(int, int) invoked\n";
}

Photo::Photo(const Photo& other)
    : width(other.width), 
    height(other.height),
    data(new int[width * height])
{
    std::cout << "Copy Constructor invoked: Photo(const Photo&)\n";
    std::copy(other.data, other.data + width * height, data);
}

Photo& Photo::operator=(const Photo& other) {
    std::cout << "Copy assignment operator invoked: operator=(const Photo&)\n";
    if (this == &other) return *this;
    delete[] data;
    width = other.width;
    height = other.height;
    data = new int[width * height];
    std::copy(other.data, other.data + width * height, data);
    return *this;
}

Photo::~Photo() {
    std::cout << "Destructor invoked\n";
    delete[] data;
}
```

**Tracing Object Creation Flow**

Let's see what happens when we create and assign objects:

```cpp
int main() {
    std::cout << "Check - 1\n";
    Photo selfie = Photo {0, 0}; 
    std::cout << "------------\n";
    Photo retake{4,5};
    std::cout << "------------\n";
    std::cout << "Check - 2\n";
    retake = Photo{1,2};
    std::cout << "------------\n";
}
```

**Output (compiled with `-O0 -fno-elide-constructors` to visalize the in-efficiency without compiler optimization):**

```
Check - 1
Photo::Photo(int, int) invoked
Copy Constructor invoked: Photo(const Photo&)
Destructor invoked
------------
Photo::Photo(int, int) invoked
------------
Check - 2
Photo::Photo(int, int) invoked
Copy assignment operator invoked: operator=(const Photo&)
Destructor invoked
------------
Destructor invoked
Destructor invoked
```


**Notice what happens:**

**Line: `Photo selfie = Photo{0, 0};`**
- Creates a temporary `Photo{0, 0}` object
- Copies it to `selfie` using the copy constructor (allocates new memory and copies all data)
- Destroys the temporary object

**Line: `retake = Photo{1,2};`**
- Creates a temporary `Photo{1,2}` object
- Copies it to `retake` using copy assignment (allocates new memory and copies all data)
- Destroys the temporary object

**The inefficiency:** We're allocating memory and copying data from temporary objects that are about to be destroyed anyway! This is wasteful, especially for large objects.

---

## Understanding rvalues

### What is an rvalue?

In the expression `Photo selfie = Photo{0, 0}`, the `Photo{0, 0}` is an **rvalue**.

An rvalue is a temporary object that:
- Doesn't have a persistent memory address
- Exists only for the duration of the expression
- Cannot have its address taken (cannot use `&` on it)
- Is about to be destroyed, so we can "steal" its resources instead of copying them

### Examples of rvalues and lvalues

**rvalues (temporaries):**
```cpp
Photo{1, 2}        // rvalue - temporary object
5                  // rvalue - literal
x + y              // rvalue - result of expression
takePhoto()        // rvalue - return value of function
```

**lvalues (persistent objects):**
```cpp
Photo selfie{1, 2};  // selfie is an lvalue - it has a persistent address
int x = 5;           // x is an lvalue
```

---

## Passing Objects to Functions

### The Naive Approach

Let's say we want to upload a photo:

```cpp
void upload(Photo p) {
    std::cout << "upload(Photo p) invoked\n";
}

int main() {
    Photo selfie = Photo{1,2};
    upload(selfie);
}
```

**Problem:** This copies the entire `Photo` object (including allocating memory and copying all pixel data) when calling `upload`. Very inefficient!

### Solution 1: Pass by lvalue Reference

To avoid copying lvalues, pass by reference:

```cpp
void upload(Photo& p) {
    std::cout << "upload(Photo& p) invoked\n";
}

int main() {
    Photo selfie = Photo{1,2};
    upload(selfie);  // No copy! Just passes a reference
}
```

**Much better!** No copying occurs.

### The Problem with Temporary Objects

What if we try this?

```cpp
int main() {
    upload(Photo{1,2});  // Passing a temporary
}
```

**Compiler error:**
```
error: candidate function not viable: expects lvalue as 1st argument
```

The problem: `Photo{1,2}` is an rvalue (temporary), but `Photo&` only binds to lvalues!

### Solution 2: rvalue References

To accept temporary objects without copying, we use **rvalue references**:

```cpp
void upload(Photo&& p) {
    std::cout << "upload(Photo&& p) invoked\n";
}

int main() {
    upload(Photo{1,2});  // Works! No copy!
}
```

**Syntax:** `Type&&` is an rvalue reference.

### Key Differences Between Reference Types

| Feature | lvalue reference (`Type&`) | rvalue reference (`Type&&`) |
|---------|---------------------------|----------------------------|
| **Binds to** | Persistent objects (lvalues) | Temporary objects (rvalues) |
| **Expectation** | Object must remain valid | Object is temporary, can be modified |
| **Use case** | Avoid copying persistent objects | Avoid copying temporary objects |

### Function Overloading with References

You can overload functions based on lvalue vs rvalue references:

```cpp
void upload(Photo& p) {
    std::cout << "upload(Photo& p) - lvalue version\n";
}

void upload(Photo&& p) {
    std::cout << "upload(Photo&& p) - rvalue version\n";
}

int main() {
    Photo selfie{1,2};
    upload(selfie);        // Calls lvalue version
    upload(Photo{3,4});    // Calls rvalue version
}
```

The compiler automatically chooses the correct version based on whether the argument is an lvalue or rvalue!

---

## Move Constructor and Move Assignment (C++11)

### The Concept

Since rvalues are temporary and about to be destroyed, we can **steal (move)** their resources instead of copying them. C++11 introduced two new special member functions:

1. **Move Constructor:** `Type(Type&& other)`
2. **Move Assignment Operator:** `Type& operator=(Type&& other)`

### Visual Comparison: Copy vs Move

#### Copy Constructor (Expensive)

```
Before Copy:
  temporary           selfie
  ┌────────┐         ┌────────┐
  │width: 2│         │  ???   │
  │height:3│         │  ???   │
  │data: ──┼──       │  ???   │
  └────────┘ │       └────────┘
             │
             ▼
         [pixel data]
         [in memory ]

After Copy Constructor:
  temporary           selfie
  ┌────────┐         ┌────────┐
  │width: 2│         │width: 2│
  │height:3│         │height:3│
  │data: ──┼──       │data: ──┼──
  └────────┘ │       └────────┘ │
             │                   │
             ▼                   ▼
         [pixel data]        [NEW pixel data]
         [original  ]        [COPIED!       ]
  
  - Allocated new memory and copied all data!
  - Two separate copies of pixel data exist
```

#### Move Constructor (Efficient)

```
Before Move:
  temporary           selfie
  ┌────────┐         ┌────────┐
  │width: 2│         │  ???   │
  │height:3│         │  ???   │
  │data: ──┼──       │  ???   │
  └────────┘ │       └────────┘
             │
             ▼
         [pixel data]
         [in memory ]

After Move Constructor:
  temporary           selfie
  ┌────────┐         ┌────────┐
  │width: 2│         │width: 2│
  │height:3│         │height:3│
  │data:NULL│        │data: ──┼──
  └────────┘         └────────┘ │
                                 │
                                 ▼
                             [pixel data]
                             [STOLEN!   ]
  
  - Just copied the pointer (steal)!
  - Set source pointer to nullptr
  - No memory allocation, no data copying!
```

### Implementation

```cpp
class Photo {
    public:
        // ... (previous members)
        Photo(Photo&& obj);               // Move constructor
        Photo& operator=(Photo&& obj);    // Move assignment operator
};

// Move constructor
Photo::Photo(Photo&& obj) {
    std::cout << "Move constructor: Photo(Photo&&) invoked\n";
    // Steal the resources
    this->width = obj.width;
    this->height = obj.height;
    this->data = obj.data;
    
    // Leave the source object in a valid but empty state
    obj.data = nullptr;
}

// Move assignment operator
Photo& Photo::operator=(Photo&& obj) {
    std::cout << "Move assignment operator: operator=(Photo&&) invoked\n";
    if (this == &obj) return *this;
    
    // Clean up our current resources
    delete[] data;
    
    // Steal the resources from obj
    this->width = obj.width;
    this->height = obj.height;
    this->data = obj.data;
    
    // Leave obj in a valid but empty state
    obj.data = nullptr;
    
    return *this;
}
```

**Key points:**
- Instead of allocating new memory and copying, we just **steal the pointer**
- We set `obj.data = nullptr` so the source object's destructor won't delete the memory we stole
- Much more efficient: just copying a few integers and a pointer!

### The Results

Running the same code with move semantics:

```cpp
int main() {
    std::cout << "Check - 1\n";
    Photo selfie = Photo {0, 0}; 
    std::cout << "------------\n";
    Photo retake{4,5};
    std::cout << "------------\n";
    std::cout << "Check - 2\n";
    retake = Photo{1,2};
    std::cout << "------------\n";
}
```

**Output (with move semantics):**

```
Check - 1
Photo::Photo(int, int) invoked
Move constructor: Photo(Photo&&) invoked
Destructor invoked
------------
Photo::Photo(int, int) invoked
------------
Check - 2
Photo::Photo(int, int) invoked
Move assignment operator: operator=(Photo&&) invoked
Destructor invoked
------------
Destructor invoked
Destructor invoked
```

**Notice:** The copy constructor and copy assignment are replaced with their move counterparts!

---

## std::move - Forcing Move Semantics

### When lvalue References Aren't Enough

Sometimes we have an lvalue that we **know** will never be used again. In these cases, copying is still inefficient.

#### Problem: Unnecessary Copies of lvalues

Consider this code that inserts a photo into a collection:

```cpp
void PhotoCollection::insert(const Photo& pic, int pos) {
    for (int i = size(); i > pos; i--)
        myPhotos[i] = myPhotos[i - 1];  // Line 3: Shuffle elements down
    myPhotos[pos] = pic;
}
```

**The inefficiency on line 3:**
- `myPhotos[i - 1]` is an lvalue (it has a persistent address)
- The copy assignment operator is called
- Each element is **copied** into its new position
- But the original value at `myPhotos[i - 1]` is **never used again** - it will be immediately overwritten!

We're doing expensive deep copies when we could just move the resources!

#### Solution: Using std::move

We can use `std::move` to treat an lvalue as an rvalue:

```cpp
void PhotoCollection::insert(const Photo& pic, int pos) {
    for (int i = size(); i > pos; i--)
        myPhotos[i] = std::move(myPhotos[i - 1]);  // Use move assignment!
    myPhotos[pos] = pic;
}
```

Now the move assignment operator is called instead of copy assignment, making the shuffling much more efficient!

### What is std::move?

**Important:** `std::move` doesn't actually move anything!

`std::move` is just a **type cast** that converts an lvalue to an rvalue reference:

```cpp
Photo selfie{1, 2};
Photo moved = std::move(selfie);  // std::move(selfie) casts selfie to Photo&&
```

After `std::move`:
1. The compiler sees an rvalue reference (`Photo&&`)
2. The move constructor/assignment operator is called
3. Resources are stolen from `selfie`
4. `selfie` is left in a **valid but unspecified state**

---

## The Danger of std::move

### Be Careful with Moved-From Objects!

```cpp
Photo takePhoto() {
    return Photo{100, 100};
}

void foo(Photo whoAmI) {
    Photo selfie = std::move(whoAmI);  // Force move from lvalue
    whoAmI.get_pixel(21, 24);          // ⚠️ DANGER!
}
```

**What happens to `whoAmI` after it's moved?**
- Its resources have been stolen by `selfie`
- It's in a **valid but unspecified state**
- In our `Photo` implementation, `whoAmI.data == nullptr`
- Calling `get_pixel()` will likely crash or cause undefined behavior!

### Moved-From Object Guarantees

After an object is moved from:
- It's in a **valid state** (you can safely destroy it)
- You can assign a new value to it
- You **cannot assume anything else** about its state
- Don't call methods that depend on its resources

**Example:**

```cpp
Photo a{10, 10};
Photo b = std::move(a);

// Safe operations on 'a':
a = Photo{5, 5};     // OK: assign new value
// a is destroyed     // OK: destructor works

// Unsafe operations on 'a':
a.get_pixel(1, 1);   // NOT OK: might crash
int w = a.width;     // NOT OK: undefined value
```

---

## Best Practices

### When to Use std::move

#### Good use cases:

**1. You know for certain the object won't be used again**
```cpp
std::vector<Photo> photos;
Photo temp{100, 100};
photos.push_back(std::move(temp));  // OK: temp not used after this
```

**2. Performance is critical and you control the object lifetime**
```cpp
Photo a{1000, 1000};
Photo b = std::move(a);
// Don't touch 'a' again!
```

**3. Implementing move constructors/assignment operators**
```cpp
Photo(Photo&& other) {
    data = std::move(other.data);  // Moving members
}
```

#### Avoid std::move when:

1. **You're not sure if the object will be used later**
2. **The performance gain is negligible** (e.g., moving small objects)
3. **You're working with function parameters that might be accessed after**

### General Guidelines

**1. Don't overuse std::move**

The compiler automatically uses move semantics for rvalues (temporaries). Only use `std::move` when you need to force move semantics on an lvalue.

**2. After moving, either:**
- Don't touch the object again, or
- Assign it a new value before using it

**3. Document when functions take ownership:**

```cpp
// Takes ownership of photo (moves it)
void PhotoCollection::insert(Photo&& photo) {
    // ...
}
```

**4. In most code, prefer copy semantics for clarity**

Use move semantics only when performance profiling shows it's necessary.

---

## Summary

### Quick Reference Table

| Concept | Syntax | Purpose |
|---------|--------|---------|
| **lvalue reference** | `Type&` | Bind to persistent objects to avoid copying |
| **rvalue reference** | `Type&&` | Bind to temporary objects to enable moving |
| **Move constructor** | `Type(Type&& other)` | Construct by stealing resources from a temporary |
| **Move assignment** | `Type& operator=(Type&& other)` | Assign by stealing resources from a temporary |

### The Big Idea

**Copy semantics (lvalue):** Object will continue to exist, must keep it valid → expensive deep copy

**Move semantics (rvalue):** Object is temporary and will be destroyed → cheap resource transfer

Move semantics provide significant performance improvements for classes that manage resources (dynamic memory, file handles, network connections, etc.) by eliminating unnecessary copies of temporary objects.

### The Complete Picture

```cpp
// 1. Automatic move (compiler does this)
Photo a = Photo{1, 2};           // Temporary → move constructor called

// 2. Copy an lvalue (default behavior)
Photo b{3, 4};
Photo c = b;                     // lvalue → copy constructor called

// 3. Force move an lvalue (use with caution!)
Photo d = std::move(b);          // std::move casts lvalue to rvalue
                                 // move constructor called
                                 // b is now in unspecified state!
```

**Key Takeaway:** Move semantics are a powerful optimization, but with great power comes great responsibility. Use `std::move` sparingly and only when you're certain the moved-from object won't be accessed again.