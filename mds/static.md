# C++ Static Members

## Table of Contents

1. [Static Data Members in a Class](#1-static-data-members-in-a-class)
2. [Static Functions in a Class](#2-static-functions-in-a-class)
3. [Why Static Functions Cannot Access Non-Static Members (The `this` Pointer Problem)](#3-why-static-functions-cannot-access-non-static-members-the-this-pointer-problem)
4. [When to Use Static Data Members: Real-World Examples](#4-when-to-use-static-data-members-real-world-examples)
5. [Singleton Design Pattern: Using Static Members](#5-singleton-design-pattern-using-static-members)
6. [Static vs Non-Static: Key Differences](#6-static-vs-non-static-key-differences)

---

## 1. Static Data Members in a Class

### What are Static Data Members?

A **static data member** is a class member that is **shared by all objects** of that class. Instead of each object having its own copy, there's only **one copy** that belongs to the class itself.

### Basic Syntax

```cpp
class MyClass {
public:
    static int count;  // Declaration inside class
    int regularVar;    // Non-static (each object has its own copy)
};

// Definition outside class (REQUIRED!)
int MyClass::count = 0;
```

**Important:** Static data members must be defined outside the class (except for `const static` integral types).

### Simple Example

```cpp
class Student {
public:
    string name;
    static int totalStudents;  // Shared by ALL students
    
    Student(string n) {
        name = n;
        totalStudents++;  // Increment shared counter
    }
};

// Must define static member outside class
int Student::totalStudents = 0;

int main() {
    cout << "Total students: " << Student::totalStudents << endl;  // 0
    
    Student s1("Alice");
    cout << "Total students: " << Student::totalStudents << endl;  // 1
    
    Student s2("Bob");
    cout << "Total students: " << Student::totalStudents << endl;  // 2
    
    Student s3("Charlie");
    cout << "Total students: " << Student::totalStudents << endl;  // 3
    
    return 0;
}
```

### Memory Layout Diagram

```
Regular (Non-Static) Members:
Each object has its own copy

    s1 object:              s2 object:              s3 object:
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│ name: "Alice"   │     │ name: "Bob"     │     │ name: "Charlie" │
└─────────────────┘     └─────────────────┘     └─────────────────┘


Static Members:
Only ONE copy shared by all objects

                    ┌─────────────────────────┐
                    │ totalStudents: 3        │ ◄─── Shared by all!
                    └─────────────────────────┘
                              ▲
                              │
              ┌───────────────┼───────────────┐
              │               │               │
          s1 uses         s2 uses         s3 uses
```

### Key Characteristics of Static Data Members

1. **Shared Across All Objects**: Only one copy exists, regardless of how many objects are created
2. **Belongs to Class, Not Objects**: Can be accessed even without creating any object
3. **Must Be Defined Outside Class**: Declaration inside, definition outside (with initialization)
4. **Lifetime**: Exists for the entire program duration
5. **Access**: Can be accessed using class name (`ClassName::staticVar`) or object (`obj.staticVar`)

### Accessing Static Data Members

```cpp
class Counter {
public:
    static int count;
};

int Counter::count = 100;

int main() {
    // Method 1: Using class name (Preferred)
    cout << Counter::count << endl;  // 100
    
    // Method 2: Using object
    Counter c1;
    cout << c1.count << endl;  // 100
    
    Counter c2;
    c2.count = 200;
    
    // All ways show the same value (shared!)
    cout << Counter::count << endl;  // 200
    cout << c1.count << endl;        // 200
    cout << c2.count << endl;        // 200
    
    return 0;
}
```

[↑ Back to Table of Contents](#table-of-contents)

---

## 2. Static Functions in a Class

### What are Static Member Functions?

A **static member function** is a function that belongs to the class itself, not to any specific object. It can be called without creating an object.

### Basic Syntax

```cpp
class MyClass {
public:
    static int count;
    
    static void displayCount() {  // Static function
        cout << "Count: " << count << endl;
    }
};

int MyClass::count = 5;

int main() {
    // Call without creating object
    MyClass::displayCount();  // Count: 5
    
    // Can also call with object (but not recommended)
    MyClass obj;
    obj.displayCount();  // Count: 5
    
    return 0;
}
```

### Real-World Example: Bank Account

```cpp
class BankAccount {
private:
    string accountHolder;
    double balance;
    static double interestRate;  // Same for all accounts
    static int totalAccounts;
    
public:
    BankAccount(string name, double bal) {
        accountHolder = name;
        balance = bal;
        totalAccounts++;
    }
    
    // Static function to set interest rate for ALL accounts
    static void setInterestRate(double rate) {
        interestRate = rate;
    }
    
    // Static function to get total accounts
    static int getTotalAccounts() {
        return totalAccounts;
    }
    
    void applyInterest() {
        balance += balance * interestRate;
    }
    
    void display() {
        cout << accountHolder << ": $" << balance << endl;
    }
};

// Define static members
double BankAccount::interestRate = 0.05;
int BankAccount::totalAccounts = 0;

int main() {
    BankAccount::setInterestRate(0.07);  // Set for ALL accounts
    
    BankAccount acc1("Alice", 1000);
    BankAccount acc2("Bob", 2000);
    
    cout << "Total accounts: " << BankAccount::getTotalAccounts() << endl;  // 2
    
    acc1.applyInterest();
    acc2.applyInterest();
    
    acc1.display();  // Alice: $1070
    acc2.display();  // Bob: $2140
    
    return 0;
}
```

### Characteristics of Static Functions

1. **No `this` Pointer**: Cannot access non-static members directly
2. **Called Using Class Name**: `ClassName::functionName()`
3. **Can Access Only Static Members**: Can use static data members and other static functions
4. **Cannot Be `const` or `virtual`**: These keywords require a `this` pointer
5. **Cannot Be Overridden**: No polymorphism with static functions

### What Static Functions CAN and CANNOT Do

```cpp
class Example {
private:
    int nonStaticVar;
    static int staticVar;
    
public:
    static void staticFunc() {
        // ✓ CAN access static members
        staticVar = 100;
        
        // ✗ CANNOT access non-static members
        // nonStaticVar = 50;  // ERROR!
        
        // ✗ CANNOT call non-static functions
        // nonStaticFunc();  // ERROR!
        
        // ✓ CAN call other static functions
        anotherStaticFunc();
    }
    
    static void anotherStaticFunc() {
        cout << "Another static function" << endl;
    }
    
    void nonStaticFunc() {
        // ✓ Non-static can access everything
        nonStaticVar = 10;
        staticVar = 20;
        staticFunc();
    }
};

int Example::staticVar = 0;
```

[↑ Back to Table of Contents](#table-of-contents)

---

## 3. Why Static Functions Cannot Access Non-Static Members (The `this` Pointer Problem)

### Understanding the `this` Pointer

Every **non-static member function** has a hidden parameter called `this` - a pointer to the object that called the function.

```cpp
class MyClass {
public:
    int x;
    
    void setX(int val) {
        x = val;  // Actually: this->x = val;
    }
};

MyClass obj;
obj.setX(10);  // Compiler passes &obj as 'this' pointer
```

**Behind the scenes:**
```cpp
// What you write:
void setX(int val) {
    x = val;
}

// What compiler sees:
void setX(MyClass* this, int val) {  // Hidden 'this' pointer!
    this->x = val;
}

// How it's called:
obj.setX(10);      // You write this
setX(&obj, 10);    // Compiler generates this
```

### The Problem with Static Functions

**Static functions have NO `this` pointer** because they don't belong to any specific object!

```cpp
class MyClass {
public:
    int x;                    // Non-static member
    static int y;             // Static member
    
    // Non-static function: Has 'this' pointer
    void nonStaticFunc() {
        x = 10;               // OK: Uses this->x
        y = 20;               // OK: Static member
    }
    
    // Static function: NO 'this' pointer
    static void staticFunc() {
        // x = 10;            // ERROR! Which object's x?
                              // No 'this' pointer to refer to!
        
        y = 20;               // OK: Static member doesn't need 'this'
    }
};

int MyClass::y = 0;
```

### Visual Explanation

```
Scenario: Three objects exist

    obj1:               obj2:               obj3:
┌──────────┐        ┌──────────┐        ┌──────────┐
│ x = 5    │        │ x = 10   │        │ x = 15   │
└──────────┘        └──────────┘        └──────────┘


When you call: obj1.nonStaticFunc()
                     ▼
            ┌────────────────────┐
            │ nonStaticFunc()    │
            │ this = &obj1   ◄───┼─── 'this' points to obj1
            │ x = this->x    ◄───┼─── Accesses obj1's x
            └────────────────────┘


When you call: MyClass::staticFunc()
                     ▼
            ┌────────────────────┐
            │ staticFunc()       │
            │ NO 'this' pointer! │ ◄─── Which object's x?
            │ x = ???            │      There's no way to know!
            └────────────────────┘
                     ▲
                     │
              Doesn't belong to
              any specific object
```

### Why This Design Makes Sense

```cpp
class Counter {
public:
    static int count;
    int id;
    
    Counter() {
        id = ++count;
    }
    
    static void resetCounter() {
        count = 0;  // ✓ Makes sense: Reset shared counter
        
        // id = 0;  // ✗ Doesn't make sense: Which object's id?
                    //   There might be 100 Counter objects!
    }
};

int Counter::count = 0;

int main() {
    Counter c1, c2, c3;  // count = 3, ids are 1, 2, 3
    
    Counter::resetCounter();  // Resets shared counter
    
    // But which id should be reset? c1's? c2's? c3's? All?
    // This is why static functions can't access non-static members!
    
    return 0;
}
```

### Workaround: Pass Object as Parameter

If a static function needs to work with non-static members, pass the object as a parameter:

```cpp
class MyClass {
public:
    int x;
    static int y;
    
    static void staticFunc(MyClass& obj) {
        obj.x = 10;   // ✓ Now we know which object!
        y = 20;       // ✓ Static member
    }
};

int MyClass::y = 0;

int main() {
    MyClass obj;
    MyClass::staticFunc(obj);  // Pass the object explicitly
    return 0;
}
```

### Summary: `this` Pointer Table

| Function Type | Has `this` Pointer? | Can Access Non-Static Members? | Can Access Static Members? |
|---------------|---------------------|-------------------------------|---------------------------|
| Non-Static Member Function | ✓ Yes | ✓ Yes | ✓ Yes |
| Static Member Function | ✗ No | ✗ No | ✓ Yes |
| Global Function | ✗ No | ✗ N/A | ✗ N/A |

[↑ Back to Table of Contents](#table-of-contents)

---

## 4. When to Use Static Data Members: Real-World Examples

### Use Case 1: Counting Objects

**Problem:** You need to know how many objects of a class exist at any time.

```cpp
class Employee {
private:
    string name;
    static int employeeCount;  // Shared counter
    
public:
    Employee(string n) : name(n) {
        employeeCount++;
        cout << "Employee created. Total: " << employeeCount << endl;
    }
    
    ~Employee() {
        employeeCount--;
        cout << "Employee destroyed. Total: " << employeeCount << endl;
    }
    
    static int getEmployeeCount() {
        return employeeCount;
    }
};

int Employee::employeeCount = 0;

int main() {
    cout << "Employees: " << Employee::getEmployeeCount() << endl;  // 0
    
    {
        Employee e1("Alice");    // Total: 1
        Employee e2("Bob");      // Total: 2
        
        cout << "Current employees: " << Employee::getEmployeeCount() << endl;  // 2
    }  // e1 and e2 destroyed here
    
    cout << "Employees: " << Employee::getEmployeeCount() << endl;  // 0
    
    return 0;
}
```

**Why Static?** Every employee needs to update the **same** counter. If it were non-static, each employee would have their own count (useless!).

### Use Case 2: Shared Configuration

**Problem:** All objects need to share the same configuration settings.

```cpp
class Logger {
private:
    string moduleName;
    static string logLevel;      // Shared by all loggers
    static bool timestampEnabled; // Shared by all loggers
    
public:
    Logger(string module) : moduleName(module) {}
    
    static void setLogLevel(string level) {
        logLevel = level;  // Changes for ALL loggers
    }
    
    static void enableTimestamp(bool enable) {
        timestampEnabled = enable;  // Changes for ALL loggers
    }
    
    void log(string message) {
        if (timestampEnabled) {
            cout << "[" << __TIME__ << "] ";
        }
        cout << "[" << logLevel << "] ";
        cout << "[" << moduleName << "] ";
        cout << message << endl;
    }
};

string Logger::logLevel = "INFO";
bool Logger::timestampEnabled = true;

int main() {
    Logger networkLogger("Network");
    Logger databaseLogger("Database");
    
    networkLogger.log("Connection established");
    databaseLogger.log("Query executed");
    
    // Change log level for ALL loggers at once
    Logger::setLogLevel("DEBUG");
    
    networkLogger.log("Detailed network info");
    databaseLogger.log("Detailed database info");
    
    return 0;
}

/* Output:
   [TIME] [INFO] [Network] Connection established
   [TIME] [INFO] [Database] Query executed
   [TIME] [DEBUG] [Network] Detailed network info
   [TIME] [DEBUG] [Database] Detailed database info
*/
```

**Why Static?** You want one central configuration that affects all loggers. Changing it once updates all instances.

### Use Case 3: Shared Resource Pool

**Problem:** All objects need to access the same limited resource (e.g., database connections).

```cpp
class DatabaseConnection {
private:
    int connectionID;
    static int maxConnections;        // Limit for ALL connections
    static int activeConnections;     // Current count
    
public:
    DatabaseConnection() {
        if (activeConnections >= maxConnections) {
            throw runtime_error("Connection pool exhausted!");
        }
        connectionID = ++activeConnections;
        cout << "Connection #" << connectionID << " established" << endl;
    }
    
    ~DatabaseConnection() {
        cout << "Connection #" << connectionID << " closed" << endl;
        activeConnections--;
    }
    
    static void setMaxConnections(int max) {
        maxConnections = max;
    }
    
    static int getActiveConnections() {
        return activeConnections;
    }
};

int DatabaseConnection::maxConnections = 3;  // Pool size: 3
int DatabaseConnection::activeConnections = 0;

int main() {
    try {
        DatabaseConnection::setMaxConnections(2);  // Limit to 2
        
        DatabaseConnection db1;  // OK: Connection #1
        DatabaseConnection db2;  // OK: Connection #2
        DatabaseConnection db3;  // ERROR: Pool exhausted!
        
    } catch (const exception& e) {
        cout << "Error: " << e.what() << endl;
    }
    
    return 0;
}

/* Output:
   Connection #1 established
   Connection #2 established
   Error: Connection pool exhausted!
   Connection #2 closed
   Connection #1 closed
*/
```

**Why Static?** The limit and current count must be shared across all connections to enforce the pool size.

### Use Case 4: Unique ID Generation

**Problem:** Each object needs a unique ID, and no two objects should have the same ID.

```cpp
class Task {
private:
    int taskID;
    string description;
    static int nextID;  // Shared ID generator
    
public:
    Task(string desc) : description(desc) {
        taskID = nextID++;  // Get unique ID and increment for next object
        cout << "Task #" << taskID << " created: " << description << endl;
    }
    
    static void resetIDCounter() {
        nextID = 1;
    }
    
    int getID() const {
        return taskID;
    }
};

int Task::nextID = 1;

int main() {
    Task t1("Write code");       // Task #1
    Task t2("Test code");        // Task #2
    Task t3("Deploy code");      // Task #3
    
    cout << "Task IDs: " << t1.getID() << ", " 
         << t2.getID() << ", " << t3.getID() << endl;
    
    return 0;
}

/* Output:
   Task #1 created: Write code
   Task #2 created: Test code
   Task #3 created: Deploy code
   Task IDs: 1, 2, 3
*/
```

**Why Static?** The `nextID` must be shared to ensure every task gets a unique, sequential ID.

### Visual Summary: When to Use Static Members

```
Use Static Data Members When:

1. Counting Objects
   ┌─────────┐ ┌─────────┐ ┌─────────┐
   │ Object1 │ │ Object2 │ │ Object3 │
   └────┬────┘ └────┬────┘ └────┬────┘
        │           │           │
        └───────────┼───────────┘
                    ▼
            ┌───────────────┐
            │ count = 3     │ ◄─── Shared counter
            └───────────────┘

2. Shared Configuration
   All objects read from the same settings
            ┌───────────────────┐
            │ config: "value"   │ ◄─── Single source of truth
            └───────────────────┘
                    ▲
        ┌───────────┼───────────┐
        │           │           │
   ┌────┴────┐ ┌───┴─────┐ ┌───┴─────┐
   │ Object1 │ │ Object2 │ │ Object3 │
   └─────────┘ └─────────┘ └─────────┘

3. Resource Pool
   Enforcing global limits across all objects
            ┌───────────────────────┐
            │ maxConnections = 5    │ ◄─── Global limit
            │ activeCount = 3       │
            └───────────────────────┘

4. Unique ID Generation
   Sequential IDs without duplicates
            ┌───────────────┐
            │ nextID = 4    │ ◄─── Increments for each object
            └───────────────┘
```

[↑ Back to Table of Contents](#table-of-contents)

---

## 5. Singleton Design Pattern: Using Static Members

### What is the Singleton Design Pattern?

The **Singleton Pattern** is a design pattern that ensures a class has **only one instance** throughout the entire program and provides a global point of access to that instance.

**Real-World Analogy:** Think of a country's president - there can only be **one** president at a time, and everyone in the country refers to the same person when they say "the president."

### Why Use Singleton?

Some resources should have only one instance:
- **Database Connection Manager** - One pool managing all connections
- **Logger** - Single logging system for the entire application
- **Configuration Manager** - One central configuration
- **Device Drivers** - Only one driver managing hardware
- **Cache** - Single shared cache for the application

### The Problem Without Singleton

```cpp
class Database {
public:
    Database() {
        cout << "Database connection created" << endl;
    }
    
    void query(string sql) {
        cout << "Executing: " << sql << endl;
    }
};

int main() {
    Database db1;  // Creates connection 1
    Database db2;  // Creates connection 2 - Wasteful!
    Database db3;  // Creates connection 3 - More waste!
    
    // We wanted ONE connection, but got THREE!
    return 0;
}
```

### How Static Members Achieve Singleton

The Singleton pattern uses:
1. **Private constructor** - Prevents external instantiation
2. **Static instance** - Holds the single instance
3. **Static function** - Provides global access to the instance

### Basic Singleton Implementation

```cpp
class Singleton {
private:
    // Private constructor - cannot create from outside
    Singleton() {
        cout << "Singleton instance created" << endl;
    }
    
    // Static pointer to hold the single instance
    static Singleton* instance;
    
public:
    // Static function to get the instance
    static Singleton* getInstance() {
        if (instance == nullptr) {
            instance = new Singleton();  // Create only once
        }
        return instance;
    }
    
    void doSomething() {
        cout << "Doing something..." << endl;
    }
};

// Define the static member
Singleton* Singleton::instance = nullptr;

int main() {
    // Singleton s;  // ERROR! Constructor is private
    
    Singleton* s1 = Singleton::getInstance();  // Creates instance
    Singleton* s2 = Singleton::getInstance();  // Returns same instance
    Singleton* s3 = Singleton::getInstance();  // Returns same instance
    
    cout << "s1 address: " << s1 << endl;
    cout << "s2 address: " << s2 << endl;
    cout << "s3 address: " << s3 << endl;
    // All three have the SAME address!
    
    s1->doSomething();
    
    return 0;
}

/* Output:
   Singleton instance created       (only once!)
   s1 address: 0x1234abcd
   s2 address: 0x1234abcd           (same address)
   s3 address: 0x1234abcd           (same address)
   Doing something...
*/
```

### Visual Diagram: Singleton Pattern

```
Without Singleton:
    main()
      │
      ├─→ new Object()  ──→  Instance 1  ┐
      │                                    │
      ├─→ new Object()  ──→  Instance 2   ├─ Multiple instances (wasteful)
      │                                    │
      └─→ new Object()  ──→  Instance 3  ┘


With Singleton:
    main()
      │
      ├─→ getInstance()  ─┐
      │                   │
      ├─→ getInstance()  ─┼─→  Single Instance  ← Static member
      │                   │
      └─→ getInstance()  ─┘
      
    All calls return the SAME instance!
```

### Real-World Example: Logger Singleton

```cpp
class Logger {
private:
    static Logger* instance;
    string logFile;
    
    // Private constructor
    Logger() {
        logFile = "application.log";
        cout << "Logger initialized with file: " << logFile << endl;
    }
    
public:
    // Prevent copying
    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;
    
    static Logger* getInstance() {
        if (instance == nullptr) {
            instance = new Logger();
        }
        return instance;
    }
    
    void log(string level, string message) {
        cout << "[" << level << "] " << message << endl;
        // In real code, would write to logFile
    }
    
    void setLogFile(string filename) {
        logFile = filename;
    }
};

Logger* Logger::instance = nullptr;

int main() {
    // Multiple parts of the program can access the same logger
    Logger::getInstance()->log("INFO", "Application started");
    Logger::getInstance()->log("DEBUG", "Processing data...");
    Logger::getInstance()->log("ERROR", "Something went wrong!");
    
    // Only ONE Logger instance was created for all these calls
    
    return 0;
}

/* Output:
   Logger initialized with file: application.log    (only once!)
   [INFO] Application started
   [DEBUG] Processing data...
   [ERROR] Something went wrong!
*/
```

### Thread-Safe Singleton (Modern C++)

The basic singleton above isn't thread-safe. Here's a better approach using **Meyer's Singleton** (C++11):

```cpp
class ThreadSafeLogger {
private:
    ThreadSafeLogger() {
        cout << "ThreadSafeLogger created" << endl;
    }
    
public:
    // Prevent copying
    ThreadSafeLogger(const ThreadSafeLogger&) = delete;
    ThreadSafeLogger& operator=(const ThreadSafeLogger&) = delete;
    
    static ThreadSafeLogger& getInstance() {
        static ThreadSafeLogger instance;  // Created only once, thread-safe!
        return instance;
    }
    
    void log(string message) {
        cout << "LOG: " << message << endl;
    }
};

int main() {
    ThreadSafeLogger::getInstance().log("Message 1");
    ThreadSafeLogger::getInstance().log("Message 2");
    
    // Same instance, guaranteed thread-safe by C++11 standard
    
    return 0;
}
```

**Why this is better:**
- No need for manual pointer management
- Thread-safe by language guarantee (C++11+)
- Automatic cleanup when program ends
- Simpler code

### Destroying the Singleton Instance

Unlike regular objects, Singleton instances need careful cleanup management. Here are different approaches:

#### Approach 1: Manual Cleanup with destroy() Method

```cpp
class Database {
private:
    static Database* instance;
    
    Database() {
        cout << "Database connection opened" << endl;
    }
    
    ~Database() {
        cout << "Database connection closed" << endl;
    }
    
public:
    Database(const Database&) = delete;
    Database& operator=(const Database&) = delete;
    
    static Database* getInstance() {
        if (instance == nullptr) {
            instance = new Database();
        }
        return instance;
    }
    
    // Method to explicitly destroy the instance
    static void destroyInstance() {
        if (instance != nullptr) {
            delete instance;
            instance = nullptr;
            cout << "Singleton instance destroyed" << endl;
        }
    }
    
    void query(string sql) {
        cout << "Executing: " << sql << endl;
    }
};

Database* Database::instance = nullptr;

int main() {
    Database::getInstance()->query("SELECT * FROM users");
    Database::getInstance()->query("INSERT INTO logs...");
    
    // Manually destroy when done
    Database::destroyInstance();
    
    // Can recreate if needed
    Database::getInstance()->query("SELECT * FROM products");
    
    // Clean up again
    Database::destroyInstance();
    
    return 0;
}

/* Output:
   Database connection opened
   Executing: SELECT * FROM users
   Executing: INSERT INTO logs...
   Database connection closed
   Singleton instance destroyed
   Database connection opened           (recreated!)
   Executing: SELECT * FROM products
   Database connection closed
   Singleton instance destroyed
*/
```

#### Approach 2: Automatic Cleanup (Meyer's Singleton - Recommended)

```cpp
class Logger {
private:
    Logger() {
        cout << "Logger created" << endl;
    }
    
    ~Logger() {
        cout << "Logger destroyed (automatic cleanup)" << endl;
    }
    
public:
    Logger(const Logger&) = delete;
    Logger& operator=(const Logger&) = delete;
    
    static Logger& getInstance() {
        static Logger instance;  // Automatically destroyed at program end!
        return instance;
    }
    
    void log(string message) {
        cout << "LOG: " << message << endl;
    }
};

int main() {
    Logger::getInstance().log("Application started");
    Logger::getInstance().log("Processing data");
    
    // No need to manually destroy!
    // Destructor automatically called when program ends
    
    return 0;
}

/* Output:
   Logger created
   LOG: Application started
   LOG: Processing data
   Logger destroyed (automatic cleanup)    ← Automatic!
*/
```

#### Approach 3: Smart Pointers (Modern C++ Style)

```cpp
class Cache {
private:
    static unique_ptr<Cache> instance;
    
    Cache() {
        cout << "Cache initialized" << endl;
    }
    
    ~Cache() {
        cout << "Cache destroyed" << endl;
    }
    
public:
    Cache(const Cache&) = delete;
    Cache& operator=(const Cache&) = delete;
    
    static Cache* getInstance() {
        if (instance == nullptr) {
            instance = unique_ptr<Cache>(new Cache());
        }
        return instance.get();
    }
    
    // Optional: Manual reset
    static void reset() {
        instance.reset();  // Automatically deletes and sets to nullptr
        cout << "Cache reset" << endl;
    }
    
    void store(string key, string value) {
        cout << "Stored: " << key << " = " << value << endl;
    }
};

unique_ptr<Cache> Cache::instance = nullptr;

int main() {
    Cache::getInstance()->store("user", "Alice");
    Cache::getInstance()->store("session", "xyz123");
    
    // Manual cleanup if needed
    Cache::reset();
    
    // Can recreate
    Cache::getInstance()->store("user", "Bob");
    
    // Automatic cleanup at program end even without reset()
    return 0;
}

/* Output:
   Cache initialized
   Stored: user = Alice
   Stored: session = xyz123
   Cache destroyed
   Cache reset
   Cache initialized
   Stored: user = Bob
   Cache destroyed              ← Automatic cleanup at program end
*/
```

#### Approach 4: atexit() for Guaranteed Cleanup

```cpp
class ResourceManager {
private:
    static ResourceManager* instance;
    
    ResourceManager() {
        cout << "Resources allocated" << endl;
    }
    
    ~ResourceManager() {
        cout << "Resources released" << endl;
    }
    
    static void cleanup() {
        if (instance != nullptr) {
            delete instance;
            instance = nullptr;
        }
    }
    
public:
    ResourceManager(const ResourceManager&) = delete;
    ResourceManager& operator=(const ResourceManager&) = delete;
    
    static ResourceManager* getInstance() {
        if (instance == nullptr) {
            instance = new ResourceManager();
            atexit(cleanup);  // Register cleanup function
        }
        return instance;
    }
    
    void manage() {
        cout << "Managing resources..." << endl;
    }
};

ResourceManager* ResourceManager::instance = nullptr;

int main() {
    ResourceManager::getInstance()->manage();
    ResourceManager::getInstance()->manage();
    
    // No manual cleanup needed!
    // atexit() ensures cleanup() is called when program exits
    
    return 0;
}

/* Output:
   Resources allocated
   Managing resources...
   Managing resources...
   Resources released        ← Called by atexit() automatically
*/
```

### Comparison: Cleanup Approaches

| Approach | Pros | Cons | Best For |
|----------|------|------|----------|
| **Manual destroy()** | Full control, can reset/recreate | Must remember to call, easy to forget | When you need explicit control |
| **Meyer's Singleton** | Automatic, thread-safe, simple | Can't reset during program execution | Most use cases (RECOMMENDED) |
| **Smart Pointers** | Automatic memory management, can reset | Slightly more complex syntax | When you need reset capability |
| **atexit()** | Guaranteed cleanup, automatic | Less common pattern, global function | Legacy code or special requirements |

### Important Notes About Destruction

1. **Meyer's Singleton is usually best** - Automatic, safe, simple
2. **Order of destruction matters** - If Singleton A depends on Singleton B, destruction order can cause issues
3. **Don't access after destruction** - If manually destroyed, ensure no further access
4. **Memory leaks in basic pointer version** - If you never call delete, memory is leaked (but OS cleans up at program end)

### Destruction Order Example (Potential Issue)

```cpp
class Logger {
private:
    Logger() { cout << "Logger created" << endl; }
    ~Logger() { cout << "Logger destroyed" << endl; }
    
public:
    static Logger& getInstance() {
        static Logger instance;
        return instance;
    }
    
    void log(string msg) { cout << "LOG: " << msg << endl; }
};

class Database {
private:
    Database() {
        Logger::getInstance().log("Database created");
    }
    
    ~Database() {
        // DANGER: Logger might be destroyed already!
        Logger::getInstance().log("Database destroyed");
    }
    
public:
    static Database& getInstance() {
        static Database instance;
        return instance;
    }
};

int main() {
    Database::getInstance();
    // At program end, destruction order of static objects is undefined!
    // If Logger is destroyed before Database, the log() call in ~Database() fails!
    return 0;
}
```

**Solution:** Avoid dependencies between Singletons' destructors, or use dependency injection instead of Singleton pattern.

### Key Points About Singleton Pattern

| Aspect | Details |
|--------|---------|
| **Purpose** | Ensure only one instance of a class exists |
| **Private Constructor** | Prevents direct instantiation |
| **Static Instance** | Holds the single instance (shared by all) |
| **Static Access Method** | Provides global access point |
| **Thread Safety** | Use Meyer's Singleton (static local) for thread safety |
| **Use Cases** | Logger, Config, DB Connection Pool, Cache |

### Pros and Cons of Singleton

**Pros:**
- ✓ Controlled access to single instance
- ✓ Reduced memory footprint
- ✓ Global access point
- ✓ Lazy initialization (created when first needed)

**Cons:**
- ✗ Can make unit testing difficult
- ✗ Violates Single Responsibility Principle
- ✗ Can introduce global state issues
- ✗ Requires careful handling in multi-threaded environments

### When to Use Singleton

✓ **Use when:**
- Only one instance should exist (e.g., hardware device manager)
- Global access point is needed
- Lazy initialization is beneficial

✗ **Don't use when:**
- You might need multiple instances in the future
- It complicates testing
- Dependency injection would be cleaner

[↑ Back to Table of Contents](#table-of-contents)

---

## 6. Static vs Non-Static: Key Differences

### Comparison Table: Static vs Non-Static

| Feature | Static Members | Non-Static Members |
|---------|---------------|-------------------|
| **Belongs To** | Class | Object |
| **Memory** | One copy per class | One copy per object |
| **Access** | ClassName::member or object.member | object.member only |
| **Lifetime** | Entire program | Object's lifetime |
| **this Pointer** | Not available | Available |
| **Can Access** | Only static members | Both static and non-static |
| **Use Case** | Shared data/utilities | Object-specific data |

### Real-World Analogy

Think of a **company** (class) and **employees** (objects):

**Static Members** = Company-wide policies/resources
- Total employee count (shared data)
- Company-wide holiday list (shared configuration)
- HR policies (static functions)
- These affect ALL employees equally

**Non-Static Members** = Individual employee properties
- Employee name (unique to each)
- Employee salary (unique to each)
- Individual performance review (non-static function)
- These are specific to each employee

```cpp
class Company {
public:
    // Static: Shared by all employees
    static string companyName;
    static int totalEmployees;
    static double companyRevenue;
    
    // Non-static: Unique to each employee
    string employeeName;
    double employeeSalary;
    string department;
    
    // Static function: Company-level operation
    static void announceCompanyMeeting() {
        cout << companyName << " meeting at 3 PM!" << endl;
    }
    
    // Non-static function: Employee-specific operation
    void giveRaise(double amount) {
        employeeSalary += amount;
    }
};
```

[↑ Back to Table of Contents](#table-of-contents)

---

## Summary: Static Members Key Concepts

### Quick Reference

```
Static Data Members:
✓ Shared by all objects of the class
✓ One copy per class, not per object
✓ Must be defined outside class
✓ Accessed using ClassName::member or object.member
✓ Lifetime: Entire program duration

Static Member Functions:
✓ Belong to the class, not objects
✓ Called using ClassName::function()
✓ No 'this' pointer
✓ Can only access static members
✓ Cannot be virtual, const, or override
✓ Used for class-level operations

When to Use Static: