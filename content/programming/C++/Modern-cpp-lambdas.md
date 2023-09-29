---
title: "Modern C++: Lambdas"
date: 2023-05-05T15:04:11-07:00
draft: false
tag: C++
author: Cheng-Yu Fan
tags: ["C++"]
categories: ["Programming"]
---
## STL
* <span style="color:red">Data stuctures</span> as ranges
* <span style="color:red">Algorithm</span>
* <span style="color:red">Ilterators</span> as glue interface
* <span style="color:red">Callables</span>

### Before Modern C++:
``` cpp
class Person {
    public:
        std::string getName() const;
        std::string getId() const;
}
```

``` cpp
bool lessName(const Person& p1, const Person& p2) {
    return p1.getName() < p2.getName();
}
bool lessId(const Person& p1, const Person& p2) {
    return p1.getId() < p2.getId();
}

std::vector<Person> coll;

std::sort(coll.begin(), coll.end(), lessName);
std::sort(coll.begin(), coll.end(), lessId);
```
The problem is that we have to define functions, and functions have several drawback. One is that we cannot define function inside local scope. It's hard to maintain the code due to naming.

### More General Approach
Creater helper functions which is callable. One of possible solution is **Lambda function**
``` cpp
std::vector<Person> coll;

std::sort(coll.begin(), coll.end(), 
         [](const Person& p1, const Person& p2) {
             return p1.getName() < p2.getName;
         });
std::sort(coll.begin(), coll.end(), 
         [](const Person& p1, const Person& p2) {
             return p1.getId() < p2.getId;
         });
```
**Lambda**: <span style="color:red">function object</span> defined on the fly.
In C++14, we even skip the specific type to more general type
``` cpp
std::vector<Person> coll;

std::sort(coll.begin(), coll.end(), 
         [](const auto& p1, const auto& p2) {
             return p1.getName() < p2.getName;
         });
std::sort(coll.begin(), coll.end(), 
         [](const auto& p1, const auto& p2) {
             return p1.getId() < p2.getId;
         });
```
``` cpp
auto twice = [](const auto& x) {
    return x+x;
};

auto i = twice(3);
auto d = twice(1.7);
auto s = twice(std::string{"hi"});
auto t = twice("hi"); // Error: const char[3] + const char[3]
```


## Lambdas as Better Functions
``` cpp
bool less7(int v) {
    return v < 7;
}

bool less8(int v) {
    return v < 8;
}

count_if(c.begin(), c.end(), less7);
count_if(c.begin(), c.end(), less8);

// what if the value is undetermined ?
void foo(int max) {
    count_if(c.begin(), c.end(), lessMax); // ?????
}
```
Lambdas can <span style="color:red">capture</span> on the behavior parameters.
* Functionality can depend on run-time parameters
``` cpp
void foo(int max) {
    count_if(c.begin(), c.end(),
    [max] (int v) {
        return v < max;
    }); 
}
```

## Lambdas are function object
* Lambdas are <span style="color:red">function objects</span>
    * of a "unique non-union class <span style="color:red">closure</span> type"
    * The object can be used like functions
        * Having <span style="color:red">operator()</span> defined
It has the same effect as:
``` cpp
auto add = [] (int x, int y) {
    return x + y;
};

class lambda??? {
    public:
    auto operator() (int x, int y) const {
        return x + y;
    }
}

auto add = lambda???{};
```

``` cpp
while (...) {
    int min, max;
    ...
    p = std::find_if(coll.begin(), coll.end(),
                    [min, max](int i) {
                        return min <= i && i <= max;
                    });
}

class lambda??? {
    private:
    int _min, _max;
    public:
    lambda???(int min, int max): _min(min), _max(max) {} // only callable before C++20
    auto operator() (int i) const {
        return min <= i && i <= max;
    }
};

// It is equal to
while (...) {
    int min, max;
    ...
    p = std::find_if(coll.begin(), coll.end(),
                    lambda???(min, max));
}

```

## Type of Lambdas
* Each lambda has its own (closure) type
    * The type of the lambda expression is a unqiue, unnamed nonunion class type
``` cpp
auto x = [] () {}; // x has its own type
auto y = [] () {}; // y has its own type

std::is_same<decltype(x), decltype(y)>::value; //yields false
auto z = x; // z has the same type
std::is_same<decltype(x), decltype(z)>::value; //yields true
```

* Lambdas are objects of a generated class
    * The closure type is a class
* You cannot find out whether an object is a lambda
``` cpp
// There is no std::is_lambda<> 
bool b1 = std::is_class<decltype(twice)>::value;  //true
bool b1 = std::is_class<decltype(twice)>::value;  //true (since C++17)
```

## Generic Functions vs. Generic Lambdas
* Generic Lambdas since C++14:
``` c++
auto printLmbd = [] (const auto& coll) {
    for (const auto& elem: coll) {
        std::cout<< elem << '\n';
    }
};
```
* Generic functions: 
```c++
template<template T>
void printFunc(const T& coll) {
    for(const auto& elem: coll) {
        std:cout << elem << '\n';
    }
}
```
``` c++
std::vector<int> v;
...
printFunc(v);
printLmbd(v);
printFunc<std::string>("hi");     //OK
printLmbd<std::string>("hi");     //ERROR
// To specify the template parameter use: printLmbd.operator()<std::string>("hi")
// The object don't have a template

call(printFunc, v);               // ERROR
call(printFunc<dectype(v)>, v);   //OK
call(printLmbd, v);               // OK, Lambda is not a generic object
```

## Lambdas: Return Type
* The minimal lambda:
```c++
[]{} // equivalent to []() -> void { }
```
* The return type can be deduced
    * ``void`` without any return statement
    * ``auto`` if all return statements have same value
```c++
[] (long val) {        // ERROR: return types differ
    if (val < 10) {
        return val;
    }
    return 10;
}

[] (long val) -> long {        // OK
    if (val < 10) {
        return val;
    }
    return 10;
}
```

## Lambdas as Function Pointers
* Lambdas can be used as ordinary <span style="color:red">function pointers</span>
    * No captures allowed
    * Can even passed to C functions
```c++
#include <cstdlib>    // for atexit()

void (*logFP)(const std::string& msg);    // actual logger (function pointer to log msg)

int main() 
{
    std::atexit([]() {                    // register lambda to call on program exit
        std::cout<<"good bye\n";
    });
    
    logFP = [](const auto& string msg) {  // set lambda as logger
        std::cout<< "LOG> "<<msg << '\n';
    };
     
    logFP("we are insdie main()");        //call the actual logger
}
```

## Lambda: Caputre by Reference
``` c++
std::vector<int> coll;

double sum = 0.0;
std::for_each(coll.begin(), coll.end(), [&sum](double d) { //[&] would catch all by reference
    sum+= d;
})
```
### Other usages
* Captures allow access to scope where lambda is defined
    * Global object are visible anyway
    * Object copied by-value are read-only(by default)
* Examples:
    * `[x, y]`
        * access to a copy of `x` and `y`
    * `[&]`
        * access to <span style="color:blue">all</span> <span style="color:red">objects</span> of outside scope <span style="color:red">by reference</span>
    * `[=]`
        * access to a copy of <span style="color:blue"> all used</span> <span style="color:red">objects</span>
        * Avoid capturing with `=` to avoid expensive copies
    * ``[&x, y]``
        *  access to `x` <span style="color:red">by reference</span>, but to a <span style="color:red">copy of</span> `y`
    *  ``[&,x]``
        *  access to a <span style="color:red">copy of</span> `x`, but to <span style="color:blue">all other</span> <span style="color:red">objects by reference</span>