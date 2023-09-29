---
title: "Modern C++: Smart Pointers"
date: 2023-05-05T15:03:58-07:00
draft: false
author: Cheng-Yu Fan
tags: ["C++"]
categories: ["Programming"]
---
## Polymorphism and heap memory

### Polymorphism with Inheritance
Before learning smart pointer, we need some review for polymorphism and heap memory.
The most important feature of Polymorphism is <span style="color:red"> heterogenous collection</span>
```c++
class GeoObj {
public:
    GeoObj() = default;
    virtual void draw() const = 0;
    virtual ~GeoObj() = default;
    ...
};

class Circle : public GeoObj {
private:
    Coord center;
    int rad;
public:
    Circle(Coord c, int r);
    virtual void draw() const override;
    ...
};

class Line : public GeoObj {
private:
    Coord from;
    Coord to;
public:
    Line(Coord f, Coord t);
    virtual void draw() const override;
    ...
}
```
```c++
std::vector<GeoObj*> p;    // heterogenous collection
Line l{Coord{1,2}, Coord{3, 4}};
Circle c{Coord{5,5}, 7};
p.push(&l);
p.push(&c);

for(GeoObj* gp : p) {
    p->draw();    // polymorphic call
}
```
![](https://i.imgur.com/8Di7cUp.png)

``` c++
std::vector<GeoObj*> createPicture() {
    std::vector<GeoObj*> p;    // heterogenous collection
    Line l{Coord{1,2}, Coord{3, 4}};
    Circle c{Coord{5,5}, 7};
    p.push(&l);
    p.push(&c);
    return p;    // Fatal runtime error: Returns vector with destroyed pointers
}

std::vector<GeoObj*> pict = creaturePicture();

for(GeoObj* gp: pict) {
    gp->draw();    //ERROR(undefined behavior)
}
```
![](https://i.imgur.com/E8oZI1u.png)


### Polymorphism with Heap Memory
To keep the value without destorying in stack memory, we create the memory in heap
```c++
std::vector<GeoObj*> createPicture() {
    std::vector<GeoObj*> p;    // heterogenous collection
    Line* lp = new Line{Coord{1,2}, Coord{3, 4}};
    Circle* cp = new Circle{Coord{5,5}, 7};
    p.push(lp);
    p.push(cp);
    return p;
}

std::vector<GeoObj*> pict = creaturePicture();

for(GeoObj* gp: pict) {
    gp->draw();    //ERROR(undefined behavior)
}

// remove all element without memory leak:
for(GeoObj*& geoPtr: pict) { // reference to pointer as we assign a new value to each element
    delete geoPtr;
    geoPtr = nullptr; // in case we don't remove the element
}

pict.clear();
```

**Can we have something clean up for us?**

## Smart Pointers
* Smart pointers
    * Object can be used like pointers, but are smarter
    * Act as <span style="color:blue">owners</span> of the objects
        * Call <span style="color:red"> delete</span> for the objects they point to **when they are is no longer used (owned)**
* <span style="color:red">Shared pointers</span>
    * Shared ownership
    * Some overhead
* <span style="color:red">Unique pointers</span>
    * Exlusive ownership
    * No overhead

### Shared Pointers
```c++
// define type alias:
using GeoPtr = std::shared_ptr<GeoObj>;

std::vector<GeoPtr> creaturePicture() {
    std::vector<GeoPtr> p;
    auto lp = std::make_shared<Line>(Coord{1, 2}, Coord{3, 4});
    // std::shared_ptr<Line> lp{new Line{Coord{1,2}, Coord{3, 4}}};
    auto cp = std::make_shared<Circle>(Coord{5, 5}, 7);
    // std::shared_ptr<Circle> cp{new Circle{Coord{1,2}, 7}};
    p.push(lp);
    p.push(cp);
    return p;
}

std::vector<GeoPtr> pict = creaturePicture();
```
**During the function call `createPicture()`**

![](https://i.imgur.com/KVeUawb.png)
**After the end of call `createPicture()`**
![](https://i.imgur.com/dV3A5pK.png)
**After the call `pict.clear()`**
![](https://i.imgur.com/URyFSrX.png)


### Weak Pointers
*  **Class** <span style="color:blue">weak_ptr</span>
    *  <span style="color:red">Observer but not owner</span>
        *  Pointer that knows when it dangles
            *  Allows code to share but not own an object or resource
    *  Useful
        *  Cyclic references
        *  Pinters to a resource with decouple lifecycle

![](https://i.imgur.com/VLAcg9d.png)


If we delete the `shared_ptr` points to `resource A`, we could still ask the `weak_ptr` "Is this still an object", which you cannot do in raw pointer. Raw pointers point the memory which may become meantime some other object, but weak pointers know when it is dangled.
![](https://i.imgur.com/IkfVcsC.png)

#### Weak pointers might hold memory after deleter is called
```c++
{
    std::weak_ptr<R> wp;
    {
        std::shared_ptr<R> gp;
        {
            auto sp = std::make_shared<R>();
            gp = sp;
            wp = sp;
        } // (1)
    }     // (2)
    ...
}         // (3) 
```
(1)
![](https://i.imgur.com/TWQH1M4.png)
(2)
![](https://i.imgur.com/incyTdi.png)
(3)
![](https://i.imgur.com/WO1OdFb.png)

* The control block is bounded by the resource `R`, which means **even though you thought the object is destoryed, but the entire object memory is still there until the weaks counter counts to 0**.
* <span style="color:red">Simple walk around solution</span>: use `std::shared_ptr<R> sp{new R};`. Two step initialization will seperate the memory and control block, which make control block memory and resource seperately.


## How is the performance of shared pointers?
![](https://i.imgur.com/6QQfsU4.png)

* Does copying shared pointers in different threads cause a data race?
    * <span style="color:red">No, changes in `use_count()` do not reflect modifications that can introduce data races.</span>
``` c++
// initalize vector with 1000 shared pointers:
std::vector<std::shared_ptr<T>> coll;
for(int i = 0; i < 1000; ++i) {
    coll.push_back(std::make_shared<T>());
}

int numIterations = 1'000'000;

void threadLoop(int numThreads) 
{
    // loop 1 milllion times (partitioned over all thread) overall all pointers:
    // & optional => optional copying the shared pointers
    for(auto& sp: coll) {            // By reference can be faster by a factor of 2 to 1000
        sp->incrementLocalInt();
    }
}
```
**<span style="color:blue">In conclusion, pass shared and weak pointers by reference</span>**

## Unique Pointers
```c++
// define type alias:
using GeoPtr = std::unique_ptr<GeoObj>;

std::vector<GeoPtr> creaturePicture() {
    std::vector<GeoPtr> p;
    auto lp = std::make_unique<Line>(Coord{1, 2}, Coord{3, 4});
    // std::unique_ptr<Line> lp{new Line{Coord{1,2}, Coord{3, 4}}};
    auto cp = std::make_unique<Circle>(Coord{5, 5}, 7);
    // std::unique_ptr<Circle> cp{new Circle{Coord{1,2}, 7}};
    
    p.push(lp); // ERROR: copying diabled
    p.push(cp); // ERROR: copying diabled
    
    p.push(std::move(lp)); // moves ownership
    p.push(std::move(cp)); // moves ownership
     
    return p;
}
```
![](https://i.imgur.com/dqnTMe5.png)
![](https://i.imgur.com/Afmh80h.png)

### Unique Pointers And Polymorphism
* **Downcasts do not work for unique pointers**
    * They will create a second owner to the object
    * You have to use temporary raw pointer
```c++
class GeoObj {...};
class Circle : public GeoObj {...};

std::vector<std::unique_ptr<GeoObj>> geoObjs;
geoObjs.push_back(std::make_unique<Circle>(...));  // OK, insert circle into collection

const auto& p = geoObjs[0];                         // p is unique_ptr<GeoObj>
std::unique_ptr<Circle> cp{p};                      // comiple-time Error
auto cp(dynamic_cast<std::unique_ptr<Circle>>(p));  // compile-time Error
auto cp = dynamic_cast<std::unique_ptr<Circle>>(p); // compile-time Error

if(auto cp = dynamic_cast<Circle*>(p.get())) {      // OK (temporary raw pointer)
    ... //use cp as Circle*
}
```

## std::variant
### Polymorphism with std::variant<>
```c++
using GeoObjVar = std::variant<Circle, Line>;

std::vecotr<GeoObjVar> createPicture() {
    std::vector<GeoObjVar> p;
    p.push_back(Line{Coord{1, 2}, Coord{3, 4}});
    p.push_back(Circle{Coord{5, 5}, 7});
    return p;
}

std::vector<GeoObjVar> pict = creatPicture();
for(const GeoObjVar& geoobj: pict) {
    switch(geoobj.index()) {
        case 0:
            std::get<0>(geoobj).draw();
            break;
        case 1:
            std::get<1>(geoobj).draw();
            break;
    }
    
    // or using visit
    std::visit([] (const auto& obj) {
        obj.draw(); //polymorphic call
    },
    geoobj);
}

pict.clear()
```


