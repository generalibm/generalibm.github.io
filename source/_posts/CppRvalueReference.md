---
title: C++ rvalue reference
date: 2019-02-14 18:46:35
tags: [cpp2.0]
keyword: [rvalue reference, move sematics, perfect forwarding]
categories: Tech
description: A brief illustration of c++ rvalue reference.
toc: true
---
## c++ rvalue reference

### why we need rvalue references?

rvalue references are a new reference type introduced in c++0x that help solve the problem of **unnecessary copying** and enable **perfect forwarding**. When the right-hand side of an assignment is an rvalue, the the left-hand side object can steal resources from the right-hand side object rather than performing a separate allocation, thus enabling **move semantics**.
<!--more-->

### what is rvalue and lvalue?

- lvalue: those could be appeared in the left of the `operator =`
- rvalue: those could only be appeared in the right of the `operator =`

**note:** `C++` with its user-defined types has introduced some subtleties regarding modifiability and assignability that cause this definition to be incorrect. Let us see the following samples:

```c++
/// sample 1
int a = 9;
int b = 4;

a = b;      //OK
b = a;      //OK
a = a + b;  //OK

a + b = 12; // Error, lvalue required as left operand of assignment

/// sample 2
string s1("hello");
string s2("world");

s1 + s2 = s2; // It is OK, how surprising!!!
cout << "s1 = " << s1 << endl; //s1 = hello
cout << "s2 = " << s2 << endl; //s2 = world

string() = "Hello World"; // It could be assigned to a temp object!!!

/// sample 3
complex<int> c1(3, 8), c2(1, 0);

c1 + c2 = complex<int>(4, 9);  //OK, c1 + c2 could be as lvalue!!!
cout << "c1 = " << c1 << endl; //c1 = (3,8)
cout << "c2 = " << c2 << endl; //c2 = (1,0)

complex<int>() = complex<int>(4, 9); //It could be assigned to a temp object!!!
```

### the forms of rvalue

- those could only appeared in the right of the `operator =`
- temp object
- return value(special form of temp object)

**note:** It would lose some import information of object(especial for rvalue) in call hierarchy. The following is a sample.

```c++
void process(int & i);
void process(int && i);

void forward(int && i)
{
    process(i);// unperfect forwarding
}
```

### What is perfect forwarding?

Perfect forwarding allows you to write a single function template that takes n arbitrary arguments and forwards them transparently to another arbitrary function. The nature of argument(modifiable, const, lvalue, or rvalue) is preserved in this forwarding process. 

```c++
template <typename T1, typename T2>
void functionA(T1 && t1, T2 && t2)
{
    functionB(std::forward<T1>(t1), 
              std::forward<T2>(t2));
}
```

### Implementation of moveable aware class

A  class with awareness have to implement the big 5, which support **move semantics**. And the **move constructor** and the **destructor** should be guaranteed  without throw, while **copy constructor** behaves as deep copy.

```c++
class Mystring
{
public:
    ~Mystring()  
    { 
        if (m_data) delete m_data; 
    }
    
    Mystring() 
    : m_data(nullptr)
    , m_len(0)
    {}
    
    Mystring(const Mystring & rhs)
    {
        m_len = rhs.m_len;
        m_data = new char[m_len + 1];
        memcpy(m_data, rhs.m_data, rhs.m_len);
        m_data[m_len] = '\0';
    }
    
    Mystring(const Mystring && rhs) noexcept // noexcept
    {
        m_data = rhs.m_data;
        m_len = rhs.m_len;
        
        rhs.m_len = 0;
        rhs.m_data = nullptr;// when rvalue is a temp object, the dctor will be called while it leaves the efficetive domain(eg: as a function parameter)
    }
    
    Mystring & operator=(const Mystring & rhs)
    {
        if (this != rhs) 
        {
            m_len = rhs.m_len;
            if (m_data) delete m_data;
            m_data = new char[m_len + 1];
            memcpy(m_data, rhs.m_data, rhs.m_len);
            m_data[m_len] = '\0';
        }
        return *this;
    }
    
    Mystring & operator=(const Mystring && rhs)
    {
        if (this != rhs)
        {
            m_len = rhs.m_len;
            if (m_data) delete m_data;
            m_data = rhs.m_data;
         
            rhs.m_len = 0;
            rhs.m_data = nullptr;
        }
        return *this;
    }
 
private:
    char * m_data; 
    size_t m_len;
};
```

### Testify it in the STL containers

