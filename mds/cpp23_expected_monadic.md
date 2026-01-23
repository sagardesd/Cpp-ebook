# C++23 std::expected: Monadic Operations - Complete Guide

## Advanced Patterns: Monadic Operations

C++23 enhanced `std::expected<T, E>` with methods to support **monadic operations**—a functional programming concept that allows you to chain operations elegantly without explicit error checking at each step.

### Monadic Functions Overview

| Function | What it does | Example Usage |
|----------|--------------|---------------|
| `and_then(F)` | Chain operations that return `expected` | `result.and_then(validate).and_then(process)` |
| `or_else(F)` | Handle errors, provide fallback | `result.or_else(retry_operation)` |
| `transform(F)` | Transform the success value | `result.transform([](int x) { return x * 2; })` |
| `transform_error(F)` | Transform the error value | `result.transform_error([](auto e) { return AppError::Failed; })` |

---

## 1. `and_then()` - Chaining Operations That Can Fail

`and_then()` is used when you want to chain multiple operations where each operation can fail. If any operation in the chain fails, the rest are skipped and the error is propagated.

### Understanding `and_then()`

```cpp
template<class F>
constexpr auto and_then(F&& f) &;
```

Here, `f` is a callable (function-like object) that you pass to `and_then()`. It can be:
- A lambda function
- A regular function
- A function pointer
- Any object with `operator()`

**What `f` must do:**
- Take the success value as input
- Return another `std::expected<U, E>` (where the error type `E` must match)

**How it works:**
- If the `std::expected` contains a **value**: `and_then()` calls `f` with that value and returns its result
- If the `std::expected` contains an **error**: `f` is NOT called, and the error is propagated forward

### Simple Example

```cpp
std::expected<int, std::string> e = 42;

auto result = e.and_then([](int x) {
    // This lambda is the callable "f"
    // It only runs if e contains a value
    return std::expected<double, std::string>{ x / 2.0 };
});

// result contains 21.0
```

If `e` had contained an error instead:
```cpp
std::expected<int, std::string> e = std::unexpected("error!");

auto result = e.and_then([](int x) {
    // This lambda is NEVER called
    return std::expected<double, std::string>{ x / 2.0 };
});

// result contains the error "error!" - lambda was skipped
```

### Practical Example

```cpp
#include <expected>
#include <string>
#include <iostream>

enum class Error {
    InvalidInput,
    OutOfRange,
    ParseError
};

std::expected<int, Error> parse_int(const std::string& str) {
    try {
        return std::stoi(str);
    } catch (...) {
        return std::unexpected(Error::ParseError);
    }
}

std::expected<int, Error> validate_range(int value) {
    if (value < 0 || value > 100) {
        return std::unexpected(Error::OutOfRange);
    }
    return value;
}

std::expected<int, Error> double_value(int value) {
    return value * 2;
}

int main() {
    // Without and_then - explicit error checking at each step
    auto result1 = parse_int("50");
    if (!result1) {
        std::cout << "Parse failed\n";
        return 1;
    }
    
    auto result2 = validate_range(*result1);
    if (!result2) {
        std::cout << "Validation failed\n";
        return 1;
    }
    
    auto result3 = double_value(*result2);
    // ... more checking
    
    // With and_then - clean chaining!
    auto result = parse_int("50")
        .and_then(validate_range)    // Called only if parse_int succeeds
        .and_then(double_value);      // Called only if validate_range succeeds
    
    if (result) {
        std::cout << "Result: " << *result << "\n";  // Prints: Result: 100
    } else {
        std::cout << "Error in chain\n";
    }
    
    // If any step fails, the chain short-circuits
    auto failed = parse_int("150")
        .and_then(validate_range)   // This fails! (out of range)
        .and_then(double_value);     // This is NEVER called - skipped!
    
    // failed contains Error::OutOfRange
}
```

**Key takeaway:** `and_then()` allows you to chain operations that can fail, and if any step fails, the rest are automatically skipped. This is known as a **monadic bind** pattern (similar to `flatMap` in other languages).

---

## 2. `transform()` - Transform Success Values

`transform()` is used when you want to transform the success value but the transformation itself **cannot fail**. It's like `map` in functional programming.

### Understanding `transform()`

```cpp
template<class F>
constexpr auto transform(F&& f) &;
```

Here, `f` is a callable that you pass to `transform()`. It can be:
- A lambda function
- A regular function
- A function pointer
- Any object with `operator()`

**What `f` must do:**
- Take the success value as input
- Return a **plain value** of any type `U` (NOT `std::expected`)

**How it works:**
- If the `std::expected` contains a **value**: `transform()` calls `f` with that value, wraps the result in a new `std::expected<U, E>`, and returns it
- If the `std::expected` contains an **error**: `f` is NOT called, and the error is propagated forward

**Key difference from `and_then()`:**
- `and_then()`: The function `f` returns `std::expected<U, E>` (can fail)
- `transform()`: The function `f` returns just `U` (cannot fail)

### Simple Example

```cpp
std::expected<int, std::string> e = 42;

auto result = e.transform([](int x) {
    // This lambda takes int, returns double (NOT std::expected)
    // It only runs if e contains a value
    return x / 2.0;
});

// result is std::expected<double, std::string> containing 21.0
```

If `e` had contained an error instead:
```cpp
std::expected<int, std::string> e = std::unexpected("error!");

auto result = e.transform([](int x) {
    // This lambda is NEVER called
    return x / 2.0;
});

// result is std::expected<double, std::string> containing error "error!"
```

### Practical Example

```cpp
#include <expected>
#include <string>
#include <iostream>

enum class Error {
    InvalidInput,
    OutOfRange
};

std::expected<int, Error> get_age() {
    return 25;
}

int main() {
    // Chain multiple transformations
    auto result = get_age()
        .transform([](int age) { 
            // Convert to double - cannot fail
            return static_cast<double>(age); 
        })
        .transform([](double age) { 
            // Convert to string - cannot fail
            return std::to_string(age); 
        })
        .transform([](const std::string& s) { 
            // Add prefix - cannot fail
            return "Age: " + s; 
        });
    
    if (result) {
        std::cout << *result << "\n";  // Prints: Age: 25.000000
    }
    
    // If get_age() returns an error, all transforms are skipped
    // and the error is propagated
}
```

### When to Use `transform()` vs `and_then()`

```cpp
enum class Error { Failed };

std::expected<int, Error> get_value() {
    return 10;
}

// Use transform() when the operation cannot fail
auto result1 = get_value()
    .transform([](int x) {
        return x * 2;  // Simple multiplication - cannot fail
    });

// Use and_then() when the operation can fail
auto result2 = get_value()
    .and_then([](int x) -> std::expected<int, Error> {
        if (x < 0) {
            return std::unexpected(Error::Failed);  // Can fail!
        }
        return x * 2;
    });
```

**Key takeaway:** Use `transform()` for simple transformations that cannot fail (like type conversions, formatting, arithmetic). Use `and_then()` when the next operation might fail.

---

## 3. `or_else()` - Error Recovery and Fallback

`or_else()` is called **only when there's an error**. It allows you to recover from errors or provide fallback values.

### Understanding `or_else()`

```cpp
template<class F>
constexpr auto or_else(F&& f) &;
```

Here, `f` is a callable that you pass to `or_else()`. It can be:
- A lambda function
- A regular function
- A function pointer
- Any object with `operator()`

**What `f` must do:**
- Take the error value as input
- Return `std::expected<T, G>` where `T` matches the original value type (error type `G` can be different)

**How it works:**
- If the `std::expected` contains a **value**: `f` is NOT called, and the value is propagated forward
- If the `std::expected` contains an **error**: `or_else()` calls `f` with that error and returns its result

**Key insight:** This is the opposite of `and_then()`—it operates on errors, not values!

### Simple Example

```cpp
std::expected<int, std::string> e = std::unexpected("error!");

auto result = e.or_else([](const std::string& error) {
    // This lambda is called because e contains an error
    std::cout << "Recovering from: " << error << "\n";
    return std::expected<int, std::string>{ 42 };  // Provide fallback
});

// result contains value 42 (recovered!)
```

If `e` had contained a value instead:
```cpp
std::expected<int, std::string> e = 100;

auto result = e.or_else([](const std::string& error) {
    // This lambda is NEVER called
    return std::expected<int, std::string>{ 42 };
});

// result contains value 100 (original value preserved)
```

### Practical Example

```cpp
#include <expected>
#include <string>
#include <iostream>

enum class NetworkError { 
    Timeout, 
    ConnectionRefused, 
    ServerError 
};

std::expected<std::string, NetworkError> fetch_from_primary() {
    // Simulate network failure
    return std::unexpected(NetworkError::Timeout);
}

std::expected<std::string, NetworkError> fetch_from_backup(NetworkError err) {
    std::cout << "Primary failed with error " << static_cast<int>(err) 
              << ", trying backup server\n";
    
    // Simulate successful backup fetch
    return std::string("Data from backup");
}

std::expected<std::string, NetworkError> fetch_from_cache(NetworkError err) {
    std::cout << "Backup also failed, using cache\n";
    return std::string("Cached data");
}

int main() {
    // Try primary, then backup, then cache
    auto result = fetch_from_primary()
        .or_else(fetch_from_backup)    // Called if primary fails
        .or_else(fetch_from_cache);     // Called if backup also fails
    
    if (result) {
        std::cout << "Success: " << *result << "\n";
    } else {
        std::cout << "All sources failed\n";
    }
}
```

### Multiple Recovery Strategies

```cpp
enum class Error { 
    NotFound, 
    PermissionDenied, 
    NetworkError 
};

std::expected<int, Error> try_operation() {
    return std::unexpected(Error::NotFound);
}

std::expected<int, Error> handle_not_found(Error err) {
    if (err == Error::NotFound) {
        std::cout << "Item not found, creating default\n";
        return 0;  // Default value
    }
    return std::unexpected(err);  // Re-propagate other errors
}

std::expected<int, Error> handle_permission(Error err) {
    if (err == Error::PermissionDenied) {
        std::cout << "Permission denied, requesting access\n";
        // ... request access logic ...
        return 42;  // After getting permission
    }
    return std::unexpected(err);  // Re-propagate other errors
}

int main() {
    auto result = try_operation()
        .or_else(handle_not_found)
        .or_else(handle_permission);
    
    if (result) {
        std::cout << "Final value: " << *result << "\n";
    }
}
```

**Key takeaway:** Use `or_else()` to implement retry logic, fallback mechanisms, or error-specific recovery strategies. It allows graceful degradation when operations fail.

---

## 4. `transform_error()` - Transform Error Values

`transform_error()` transforms the error type, useful for converting between different error representations without affecting the success value.

### Understanding `transform_error()`

```cpp
template<class F>
constexpr auto transform_error(F&& f) &;
```

Here, `f` is a callable that you pass to `transform_error()`. It can be:
- A lambda function
- A regular function
- A function pointer
- Any object with `operator()`

**What `f` must do:**
- Take the error value as input
- Return a **plain error value** of any type `G` (NOT `std::expected`)

**How it works:**
- If the `std::expected` contains a **value**: `f` is NOT called, and the value is propagated forward
- If the `std::expected` contains an **error**: `transform_error()` calls `f` with that error, wraps the result in a new `std::expected<T, G>`, and returns it

**Key difference from `or_else()`:**
- `or_else()`: The function `f` returns `std::expected<T, G>` (can recover to a value)
- `transform_error()`: The function `f` returns just `G` (only transforms the error, cannot recover)

### Simple Example

```cpp
enum class NetworkError { Timeout, ConnectionRefused };
enum class AppError { NetworkIssue, InvalidData };

std::expected<int, NetworkError> e = std::unexpected(NetworkError::Timeout);

auto result = e.transform_error([](NetworkError net_err) {
    // This lambda takes NetworkError, returns AppError (NOT std::expected)
    // It only runs if e contains an error
    return AppError::NetworkIssue;
});

// result is std::expected<int, AppError> containing AppError::NetworkIssue
```

If `e` had contained a value instead:
```cpp
std::expected<int, NetworkError> e = 42;

auto result = e.transform_error([](NetworkError net_err) {
    // This lambda is NEVER called
    return AppError::NetworkIssue;
});

// result is std::expected<int, AppError> containing value 42
```

### Practical Example

```cpp
#include <expected>
#include <string>
#include <iostream>

// Low-level error types
enum class FileError { 
    NotFound, 
    PermissionDenied, 
    ReadError 
};

enum class ParseError { 
    InvalidFormat, 
    MissingField 
};

// High-level application error type
enum class AppError {
    ConfigurationError,
    DataError,
    SystemError
};

std::expected<std::string, FileError> read_config(const std::string& path) {
    // Simulate file not found
    return std::unexpected(FileError::NotFound);
}

std::expected<int, ParseError> parse_port(const std::string& content) {
    return std::unexpected(ParseError::MissingField);
}

int main() {
    // Convert low-level errors to high-level application errors
    auto result = read_config("config.json")
        .transform_error([](FileError err) {
            // Map file errors to app errors
            switch (err) {
                case FileError::NotFound:
                case FileError::PermissionDenied:
                    return AppError::ConfigurationError;
                case FileError::ReadError:
                    return AppError::SystemError;
            }
            return AppError::SystemError;
        })
        .and_then(parse_port)
        .transform_error([](ParseError err) {
            // Map parse errors to app errors
            return AppError::DataError;
        });
    
    // result is now std::expected<int, AppError>
    if (!result) {
        AppError app_err = result.error();
        if (app_err == AppError::ConfigurationError) {
            std::cout << "Configuration error occurred\n";
        }
    }
}
```

### Logging or Enriching Errors

```cpp
enum class Error { 
    NotFound, 
    Timeout 
};

struct DetailedError {
    Error code;
    std::string message;
    std::string timestamp;
};

std::expected<int, Error> fetch_data() {
    return std::unexpected(Error::Timeout);
}

std::string get_timestamp() {
    return "2024-01-15 10:30:00";
}

int main() {
    auto result = fetch_data()
        .transform_error([](Error err) {
            // Enrich simple error with context
            DetailedError detailed;
            detailed.code = err;
            detailed.timestamp = get_timestamp();
            
            switch (err) {
                case Error::NotFound:
                    detailed.message = "Resource not found";
                    break;
                case Error::Timeout:
                    detailed.message = "Operation timed out";
                    break;
            }
            
            return detailed;
        });
    
    if (!result) {
        const auto& err = result.error();
        std::cout << "[" << err.timestamp << "] " 
                  << err.message << "\n";
    }
}
```

### Converting Error Types in a Pipeline

```cpp
// Different layers use different error types
enum class DbError { ConnectionFailed, QueryFailed };
enum class ServiceError { DatabaseError, ValidationError };
enum class ApiError { InternalError, BadRequest };

std::expected<std::string, DbError> query_database() {
    return std::unexpected(DbError::QueryFailed);
}

int main() {
    // Transform errors through each layer
    auto result = query_database()
        // DB layer → Service layer
        .transform_error([](DbError db_err) -> ServiceError {
            return ServiceError::DatabaseError;
        })
        // Service layer → API layer
        .transform_error([](ServiceError svc_err) -> ApiError {
            switch (svc_err) {
                case ServiceError::DatabaseError:
                    return ApiError::InternalError;
                case ServiceError::ValidationError:
                    return ApiError::BadRequest;
            }
            return ApiError::InternalError;
        });
    
    // result is std::expected<std::string, ApiError>
}
```

**Key takeaway:** Use `transform_error()` when you need to convert between different error type domains (e.g., from low-level system errors to high-level application errors) or to enrich errors with additional context.

---

## Comparison of All Monadic Operations

| Operation | Operates On | Input Function Returns | Result When Value | Result When Error | Use Case |
|-----------|-------------|------------------------|-------------------|-------------------|----------|
| `and_then(f)` | **Value** | `std::expected<U, E>` | Calls `f`, returns its result | Propagates error | Chain operations that can fail |
| `transform(f)` | **Value** | `U` (plain value) | Calls `f`, wraps result in `expected` | Propagates error | Transform value (cannot fail) |
| `or_else(f)` | **Error** | `std::expected<T, G>` | Propagates value | Calls `f`, returns its result | Recover from errors, provide fallbacks |
| `transform_error(f)` | **Error** | `G` (plain error) | Propagates value | Calls `f`, wraps result in `expected` | Convert error types |

---

## Real-World Example: Complete Pipeline

Here's a complete example showing how all monadic operations work together:

```cpp
#include <expected>
#include <string>
#include <iostream>
#include <fstream>
#include <sstream>

enum class FileError { NotFound, ReadError };
enum class ParseError { InvalidFormat, MissingField };
enum class ValidationError { OutOfRange };
enum class AppError { Failed, ConfigError };

// Read file
std::expected<std::string, FileError> read_file(const std::string& path) {
    std::ifstream file(path);
    if (!file.is_open()) {
        return std::unexpected(FileError::NotFound);
    }
    std::stringstream buffer;
    buffer << file.rdbuf();
    return buffer.str();
}

// Parse config (can fail)
std::expected<int, ParseError> parse_config(const std::string& content) {
    try {
        return std::stoi(content);
    } catch (...) {
        return std::unexpected(ParseError::InvalidFormat);
    }
}

// Validate (can fail)
std::expected<int, ValidationError> validate_value(int value) {
    if (value < 1 || value > 1000) {
        return std::unexpected(ValidationError::OutOfRange);
    }
    return value;
}

// Fallback for file errors
std::expected<std::string, FileError> use_default_config(FileError err) {
    std::cout << "File error, using default config\n";
    return std::string("100");  // Default value
}

int main() {
    auto result = read_file("config.txt")
        // Provide fallback if file reading fails
        .or_else(use_default_config)
        
        // Chain parsing (can fail with ParseError)
        .and_then(parse_config)
        
        // Convert ParseError to AppError
        .transform_error([](ParseError err) {
            return AppError::ConfigError;
        })
        
        // Chain validation (can fail with ValidationError)
        .and_then(validate_value)
        
        // Convert ValidationError to AppError
        .transform_error([](ValidationError err) {
            return AppError::Failed;
        })
        
        // Transform the final value (cannot fail)
        .transform([](int val) {
            return "Processed value: " + std::to_string(val * 10);
        });
    
    // result is std::expected<std::string, AppError>
    if (result) {
        std::cout << *result << "\n";
    } else {
        std::cout << "Pipeline failed with app error\n";
    }
}
```

---

## Benefits of Monadic Operations

1. **Less Boilerplate**: No need for explicit error checking at each step
2. **Better Readability**: The flow of operations is clear and linear
3. **Composability**: Easy to build complex pipelines from simple operations
4. **Error Propagation**: Errors automatically propagate through the chain
5. **Short-Circuiting**: Once an error occurs, remaining operations are skipped
6. **Type Safety**: The compiler ensures all error types are handled correctly

### Decision Guide: Which Operation to Use?

```
Does your function operate on the VALUE or the ERROR?
│
├─ VALUE
│  │
│  └─ Can the operation FAIL?
│     │
│     ├─ YES → Use and_then()
│     │         (returns std::expected)
│     │
│     └─ NO  → Use transform()
│                (returns plain value)
│
└─ ERROR
   │
   └─ Can it RECOVER to a value?
      │
      ├─ YES → Use or_else()
      │         (returns std::expected)
      │
      └─ NO  → Use transform_error()
                 (returns plain error)
```

These monadic operations make `std::expected` not just a return type, but a powerful tool for building robust, readable error-handling pipelines!