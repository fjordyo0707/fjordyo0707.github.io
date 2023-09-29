---
title: "Modern C++: Move Semantics"
date: 2023-05-05T15:04:25-07:00
draft: false
tag: C++
author: Cheng-Yu Fan
tags: ["C++"]
categories: ["Programming"]
---
## Why we want to add Move Semantics?

### Copy Semantics (with C++98/C++03)
Mostly the compiler will know that it is a temporary object even though you use it inline.
```cpp
std::vector<std::string> vec;
vec.reserve(3);

std::string s(getData());

vec.push_back(s);

// temporary object will be created and destory
vec.push_back(getData());
```

### Object with Name
We keep read the string to read_str and put it in read_vec.
``` cpp
std::string read_str;
std::vector<std::string> read_vec;
while(std::getline(my_stream, read_str)) {      // read a line to read_str
    read_vec.push_back(read_str);               // copy it to the element of read_vec
}
```

## Move Semantics
The `std::move` is all about ownership, which means that I no longer need this value here. With the **move semantics**, you are telling the compiler to avoid declaring temporary object. Besides, it will just move the pointer of the object to the object who are taking this ownership. 
``` cpp
std::string read_str;
std::vector<std::string> read_vec;
while(std::getline(my_stream, read_str)) {      // read a line to read_str
    read_vec.push_back(std::move(read_str));    // move it to the element of read_vec
}
```

### What is the state of object after moved?
The object will become a valid but unspecified state. You could still `std::cout` the value, but you don't know what is it inside.

### Re-use the object after moved
```cpp
void swap(std::string& a, string& b) {
    std::string temp(std::move(a));
    a = std::move(b);
    b = std::move(temp);
}
```

## RValue References
### Unnecessary copies in C++98/C++03
```cpp
template <typename T>
class vector {
    public:
        void push_back(const T& elem);
}

std::vector<std::string> vec;
std::string str = getData();

vec.push_back(s);           // copy s into vec
vec.push_back(getData());   // unnecessary copy into vec
vec.push_back(s+s);         // unnecessary copy into vec
vec.push_back(s);           // no longer need s, unnecessary copy into vec
```
### Vector with rvalue references
rvalues reference bind to rvalues
```cpp
template <typename T>
class vector {
    public:
        void push_back(const T& elem);
        void push_back(const T&& elem);
}
```

### String with rvalue references
```cpp
class string {
    private:
        int len;
        char* data;
    public:
        // create a full copy of s
        string(const string& s)
        : len(s.len) {
            if(len > 0) {
                data = new char[len + 1]
                memcpy(data, s.data, len+1)
            }
        }

        //create a copy of s with content moved
        string (string&& s)
        : len(s.len),       // copy length and
          data(s.data) {    // copy pointer to memory
            // erase the memory at source to move ownership
            s.data = nullptr;
            s.len = 0;
        } 
}
```
* `std::move(s)` is just the same as `static_cast<string&&>(s)`
* Don't declare `const` to the object which you are going to move later

## Summary for std::move
1. Optimizing copying
2. Stealing ownership form the source
3. Dealing with rvalue references(type&&)
4. Ideally support temporary object

## Basic Move Support

### Generated Move Semantics
```cpp
class Foo {
    private:
        std::string first;
        std::string last;
        int val;
  public:
        Foo(const std::string& f, const std::string l, int v): first{f}, last{l}, val{v} {   
        }
    // no copy constructor/assignment 
    // no move constructor/assignment
    // no destructor

    
    // If declared copy constructor/assignment, move fall back to copy constuctor/assignment
    Foo(const Foo& f): first{f.first}, last{f.last}, val{f.val} {}
    
    
    // If declared move constructor/ assignment, copy constuctor/assignment deleted because of speical user-declared speoical move member function
    Foo(Foo&& f) noexcept
        : first{std::move(f.first)}, last{std::move(f.last)}, val{f.val} {
        f.val *= -1
    }
    
    // Then move semantics is enabled 
}

std::vector<Foo> vec;
vec.push_back(Foo{"Dwayne", "Johnson", 44}); // create object and copy/move into vec;
```


|  | default constructor | copy constructor | copy assignment | move consturctor | move assignment | destructor | 
| -------- | -------- | -------- | -------- | -------- | -------- | -------- | 
| nothing | defaulted | defaulted | defaulted | defaulted | defaulted | defaulted | 
| any constructor | <span style="color:pink">undeclared</span> | defaulted | defaulted | defaulted | defaulted | defaulted | 
| default constructor | <span style="color:purple">user declared</span> | defaulted | defaulted | defaulted | defaulted | defaulted | 
| copy constructor | <span style="color:pink">undeclared</span> | <span style="color:purple">user declared</span> | defaulted | <span style="color:pink">undeclared(fallback enabled)</span> | <span style="color:pink">undeclared(fallback enabled)</span> | defaulted | 
| copy assignment | defaulted | defaulted | <span style="color:purple">user declared</span> | <span style="color:pink">undeclared(fallback enabled) | <span style="color:pink">undeclared(fallback enabled)</span> | defaulted | 
| move constructor | <span style="color:pink">undeclared</span> | <span style="color:red">deleted</span> | <span style="color:red">deleted</span> | <span style="color:purple">user declared</span> | <span style="color:pink">undeclared(fallback disabled)</span> | defaulted | 
| move assignment | defaulted | <span style="color:red">deleted</span> | <span style="color:red">deleted</span> | <span style="color:pink">undeclared(fallback disabled)</span> | <span style="color:purple">user declared</span> | defaulted | 
| destructor | defaulted | defaulted | defaulted | <span style="color:pink">undeclared(fallback enabled)</span> | <span style="color:pink">undeclared(fallback enabled)</span> | <span style="color:purple">user declared</span> | 

## How-to for Move Semantics 
* As an application programmer"
    * Avoid object with names
    * Used std::move()
    * if you pass a named object/value to copy you no longer use (and where copying is expensive)
* As a programmer of non-trivial classes:
    * If copying is implemented and expensive, implement move
    *  If moving members creates inconsistencies, implement move
    *  Use **noexcept** when implementing move operations   
    *  If you have to declare **one** of the 5 special member functions, **think carefully about all** of them
        *  copy constructor, move constructor, copy assignment, move assignment, destructor

