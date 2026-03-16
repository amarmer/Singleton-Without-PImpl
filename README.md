## Destructible Singleton without PImpl

The classic PImpl (Pointer to Implementation) is a well-known C++ pattern to hide implementation details from header files.</br>
However, it requires heap allocation via `std::unique_ptr<Impl>`, which may not always be desirable.</br>
This article describes an alternative approach for singletons that hides implementation details without heap allocation</br>
and allows singleton reconstruction.

## Framework

Instead of std::unique_ptr<Impl>, the singleton is stored in pre-allocated stack storage managed by SingletonProxy, 
with no heap allocation.</br> Unlike a simple singleton that lives for the entire application lifetime, 
the framework allows it to be recreated when needed.

The framework is split into two reusable headers — `SingletonProxy.h` and `Singleton.h` — so the developer only 
needs to write the interface and the implementation.

### SingletonProxy.h

Manages singleton lifetime — storage, mutex, ref counting.

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
            s_pImpl->~T();
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

Wrapper class — constructs on creation, destructs on destruction, calls any virtual function.

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

## Example: Calculator.h, Calculator.cpp, main.cpp

For instance, a singleton class `Calculator` implements interface `ICalculator`:

### Calculator.h

```cpp
#pragma once

#include "Singleton.h"

class ICalculator {
public:
    virtual ~ICalculator() = default;
    virtual void Add(int n) = 0;
    virtual int Sum() = 0;
};

// References to SingletonProxy<ICalculator, CalculatorImpl>::Construct/Destruct in Calculator.cpp
ICalculator& ConstructCalculator();
void DestructCalculator();

class Calculator: public Singleton<ConstructCalculator, DestructCalculator> {};
```

### Calculator.cpp

```cpp
#include "Calculator.h"
#include "SingletonProxy.h"

class CalculatorImpl: public ICalculator {
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

// These functions are referenced in Calculator.h
ICalculator& ConstructCalculator() {
    return SingletonProxy<ICalculator, CalculatorImpl>::Construct();
}

void DestructCalculator() {
    SingletonProxy<ICalculator, CalculatorImpl>::Destruct();
}
```

### main.cpp

```cpp
#include "Calculator.h"
#include <iostream>

int main() {
    Calculator calculator;
    calculator.Call(&ICalculator::Add, 7);

    Calculator().Call(&ICalculator::Add, 10);

    std::cout << "sum: " << calculator.Call(&ICalculator::Sum) << std::endl;
    
    return 0;
}
```

The code can be tested at: [https://wandbox.org/permlink/WdKS9GIGJiGAxpYa](https://wandbox.org/permlink/WdKS9GIGJiGAxpYa)
