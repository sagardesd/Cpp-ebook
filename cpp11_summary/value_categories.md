# Value Categories in C++

## Table of Contents

1. [Overview](#overview)
2. [Value Category Hierarchy Diagram](#value-category-hierarchy-diagram)
3. [prvalue (pure rvalue)](#prvalue-pure-rvalue)
4. [lvalue](#lvalue)
5. [xvalue (expiring value)](#xvalue-expiring-value)
6. [Summary Table](#summary-table)
7. [Quick Examples](#quick-examples)
8. [Intuition](#intuition)
9. [Visual Relationships](#visual-relationships)

---

## Overview

In C++, every expression has two properties: a **type** and a **value category**.  
Value categories describe what kind of value an expression yields.

C++11 introduced 3 primary categories:

- **lvalue**
- **xvalue**
- **prvalue**

And two broader categories:

- **glvalue** = lvalue or xvalue  
- **rvalue** = xvalue or prvalue

[↑ Back to Table of Contents](#table-of-contents)

---

## Value Category Hierarchy Diagram

```
                    expression
                        |
           +------------+------------+
           |                         |
       glvalue                    rvalue
           |                         |
      +----+----+              +-----+-----+
      |         |              |           |
   lvalue    xvalue        xvalue      prvalue
```

**Explanation:**
- **glvalue** (generalized lvalue): Has identity
- **rvalue**: Can be moved from
- **lvalue**: Has identity, cannot be moved from (unless explicitly cast)
- **xvalue**: Has identity AND can be moved from
- **prvalue**: No identity, temporary value

[↑ Back to Table of Contents](#table-of-contents)

---

## prvalue (pure rvalue)

A *prvalue* is a temporary value that does not refer to an existing object.

**Examples:**

```cpp
int x = 5;          // 5 is a prvalue  
int y = x + 10;     // (x + 10) is a prvalue  
std::string s("hi"); // temporary std::string → prvalue
```

**Characteristics:**
- ❌ Not addressable (no identity)
- Creates a new temporary object
- Result of most operators and literals

[↑ Back to Table of Contents](#table-of-contents)

---

## lvalue

An *lvalue* refers to an identifiable, persistent object in memory.

**Examples:**

```cpp
int a = 10;  // a is an lvalue  
int &r = a;  // r is also an lvalue  
struct S { int m; };
S s;
s.m = 5;     // s.m is an lvalue
```

**Characteristics:**
- ✅ Addressable (has identity)
- Persists beyond a single expression
- Can appear on the left side of assignment

[↑ Back to Table of Contents](#table-of-contents)

---

## xvalue (expiring value)

An *xvalue* is a special glvalue that refers to an object whose resources can be reused (i.e., movable).

**Examples:**

```cpp
std::string s = "hello";
std::string s2 = std::move(s);  // std::move(s) → xvalue  

struct S { std::string name; };
S getS();
getS().name = "Alice"; // getS() is prvalue, getS().name is xvalue
```

**Characteristics:**
- ✅ Addressable (has identity)
- About to expire (resources can be moved)
- Result of `std::move()` or member access on rvalue

[↑ Back to Table of Contents](#table-of-contents)

---

## Summary Table

| Category    | Meaning                                    | Example                | Addressable? |
|-------------|--------------------------------------------|------------------------|--------------|
| **prvalue** | Temporary / non-object expression          | `5`, `"hi"`, `x+1`     | ❌           |
| **lvalue**  | Persistent object with identity            | variables, members     | ✅           |
| **xvalue**  | Expiring object suitable for move          | `std::move(obj)`       | ✅           |
| **glvalue** | lvalue or xvalue                           | ---                    | ✅           |
| **rvalue**  | prvalue or xvalue                          | ---                    | depends      |

[↑ Back to Table of Contents](#table-of-contents)

---

## Quick Examples

```cpp
int a = 10;        // a → lvalue  
int b = a + 5;     // a + 5 → prvalue  

int &r = a;        // r → lvalue  

int &&rr = 20;     // 20 → prvalue; rr → lvalue (named ref)  

std::string s = "hello";          
std::string s2 = std::move(s);    // xvalue  

struct S { int m; };
S get();
get().m = 1;        // get() → prvalue, .m → xvalue
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Intuition

- **prvalue** → makes a new temporary object  
- **lvalue** → refers to an existing object  
- **xvalue** → refers to a disposable/expiring object  
- **glvalue** → has identity  
- **rvalue** → temporary or movable

[↑ Back to Table of Contents](#table-of-contents)

---

## Visual Relationships

```
Properties Matrix:

                    Has Identity    No Identity
                    ────────────    ───────────
Can Move From       │  xvalue   │   prvalue  │
                    ────────────────────────────
Cannot Move From    │  lvalue   │     N/A    │
                    ────────────────────────────

Groupings:

glvalue = { lvalue, xvalue }  ← Things with identity
rvalue  = { xvalue, prvalue } ← Things you can move from
```

**Key Insight:**
- **xvalue** is the intersection: has identity AND can be moved from
- Think of value categories as answering two questions:
  1. Does it have an identity (address)?
  2. Can we steal its resources (move)?

[↑ Back to Table of Contents](#table-of-contents)