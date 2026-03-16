# Reconstructable Singleton

Conventional singletons live for the entire application lifetime and typically require heap allocation.</br>This article describes a singleton design that uses pre-allocated static storage instead, hides implementation details from headers,</br> and supports explicit reconstruction.

## Use Cases

A reconstructable singleton is useful whenever the instance lifetime does not match the process lifetime:

- Unit testing — destroy and reconstruct the singleton between tests for a clean slate, without restarting the process.
- Plugin / dynamic library unloading — explicitly destroy the singleton before unloading a `.dll`/`.so` to avoid crashes from resources freed too late.
- Configuration reload — reinitialize a subsystem in-place (e.g. reconnect to a different database, restart a logger with new settings) without a full process restart.
- Phased startup / shutdown — control initialization and teardown order explicitly when static initialization order is insufficient.

## Framework

Instead of heap allocation as in PImpl, the singleton is stored in pre-allocated static storage managed by `SingletonProxy`.</br>Unlike a conventional singleton that lives for the entire application lifetime, the framework supports explicit construction and destruction, enabling reconstruction when needed.

The framework is split into two reusable headers — `SingletonProxy.h` and `Singleton.h` — so the developer only needs to write the interface and the implementation.

### SingletonProxy.h

Manages singleton lifetime: static storage, mutex protection, and ref counting. The implementation is constructed on the first `Construct()` call and destroyed only when the last matching `Destruct()` call brings the ref count to zero — allowing multiple RAII handles to safely share one underlying instance.

```cpp
#pragma once

#include <mutex>

template <typename IT, typename T>
class SingletonProxy {
public:
    static IT& Construct() {
        std::lock_guard<std::mutex> lock(s_mutex);
        if (++s_refCount == 1) {
            s_pImpl = new (&s_storage) T();
        }
        return *s_pImpl;
    }

    static void Destruct() {
        std::lock_guard<std::mutex> lock(s_mutex);
        if (--s_refCount == 0) {
            dynamic_cast<T*>(s_pImpl)->T::~T();
            s_pImpl = nullptr;
        }
    }

    static std::mutex& Mutex() {
        return s_mutex;
    }

private:
    static unsigned char s_storage[sizeof(T)];
    static T* s_pImpl;
    static std::mutex s_mutex;
    static int s_refCount;
};

template<typename IT, typename T>
T* SingletonProxy<IT, T>::s_pImpl = nullptr;

template<typename IT, typename T>
std::mutex SingletonProxy<IT, T>::s_mutex;

template<typename IT, typename T>
alignas(T) unsigned char SingletonProxy<IT, T>::s_storage[sizeof(T)];

template<typename IT, typename T>
int SingletonProxy<IT, T>::s_refCount = 0;
```

### Singleton.h

An RAII wrapper — constructs the implementation on creation, destructs it on destruction, and dispatches calls to any virtual function through the interface.

```cpp
#pragma once

#include <type_traits>
#include <functional>

template <auto Construct, auto Destruct>
class Singleton {
    using IT = std::remove_reference_t<decltype(Construct())>;
public:
    Singleton() : impl_(Construct()) {}
    ~Singleton() { Destruct(); }

    template<typename F, typename... Args>
    auto Call(F f, Args... args) {
        return std::invoke(f, impl_, args...);
    }

private:
    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;

protected:
    IT& impl_;
};
```

## Example: Calculator

To illustrate, a singleton class `Calculator` implements the interface `ICalculator`.

### Calculator.h

Declares the interface, the construct/destruct functions (defined in `Calculator.cpp`), and the `Calculator` class itself. Implementation details are fully hidden from this header.

```cpp
#pragma once

#include "Singleton.h"

class ICalculator {
public:
    virtual void Add(int n) = 0;
    virtual int Sum() = 0;
};

// Defined in Calculator.cpp, which is the only translation unit that knows about CalculatorImpl
ICalculator& ConstructCalculator();
void DestructCalculator();

class Calculator : public Singleton<ConstructCalculator, DestructCalculator> {};
```

### Calculator.cpp

The only file that knows about `CalculatorImpl`. It will never appear in any header.

```cpp
#include "Calculator.h"
#include "SingletonProxy.h"

class CalculatorImpl : public ICalculator {
public:
    void Add(int n) override {
        sum_ += n;
    }

    int Sum() override {
        return sum_;
    }

private:
    int sum_ = 0;
};

ICalculator& ConstructCalculator() {
    return SingletonProxy<ICalculator, CalculatorImpl>::Construct();
}

void DestructCalculator() {
    SingletonProxy<ICalculator, CalculatorImpl>::Destruct();
}
```

### main.cpp

Both `calculator` and the temporary `Calculator()` refer to the same underlying `CalculatorImpl` instance — the ref count keeps it alive until both handles go out of scope.

```cpp
#include "Calculator.h"
#include <iostream>

int main() {
    Calculator calculator;
    calculator.Call(&ICalculator::Add, 7);

    Calculator().Call(&ICalculator::Add, 10);  // temporary, shares the same instance

    std::cout << "sum: " << calculator.Call(&ICalculator::Sum) << std::endl;
    // prints: sum: 17

    return 0;
}
```

The code can be tested at: [https://wandbox.org/permlink/2Yq6uPhxIF5TJrui](https://wandbox.org/permlink/2Yq6uPhxIF5TJrui)
