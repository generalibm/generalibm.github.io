---
title: unsigned integers in cpp
date: 2019-05-07 10:23:38
tags: cpp
categories: cpp
keyword: [unsigned integers, wraps around, modulo wrapping, overflow]
description: A brief illustration of c++ unsigned integers.
toc: true
---

In [computer programming](https://en.wikipedia.org/wiki/Computer_programming), an **integer overflow** occurs when an [arithmetic](https://en.wikipedia.org/wiki/Arithmetic)  operation attempts to create a numeric value that is outside of the  range that can be represented with a given number of digits – either  larger than the maximum or lower than the minimum representable value. 

The most common result of an overflow is that the least  significant representable digits of the result are stored; the result is  said to *wrap* around the maximum (i.e. [modulo](https://en.wikipedia.org/wiki/Modular_arithmetic) a power of the [radix](https://en.wikipedia.org/wiki/Radix), usually two in modern computers, but sometimes ten or another radix). 

An overflow condition may give results leading to unintended  behavior. In particular, if the possibility has not been anticipated,  overflow can compromise a program's reliability and [security](https://en.wikipedia.org/wiki/Software_security). 

<!--more-->

### Unsigned integer range

We all know that `c++` supports both **signed integer** and **unsigned integer**, the most regular example would be:

```c++
/// the usage of signed integer with 4-bytes.
for (int i = 0; i < container.size(); i++)
{
	// do something    
}

/// the usage of unsigned integer with 4-bytes.
/// why do developer do in this way for that `size()` returns a unsigned type
/// and size_t is actually a typedef of unsigned type in cpp
for (size_t i = 0; i < container.size(); i++)
{
    // do something
}
```

Both of them would be behaving well. If we need a bigger number, we would use more bytes to present it 

| size/type       | range                          |
| --------------- | ------------------------------ |
| 1 byte unsigned | 0 to 255                       |
| 2 byte unsigned | 0 to 65,535                    |
| 4 byte unsigned | 0 to 4,294,967,295             |
| 8 byte unsigned | 0 to18,446,744,073,709,551,615 |

An n-bit unsigned variable has a range of 0 to (2^n)-1.

### Unsigned integer overflow

But when we traverse a container in reversed direction, would it properly behave as we expected?

```c++
/// signed integer example
for (int i = container.size()-1; i >= 0; i--)
{
    // do something
}

// unsigned integer example
// dead loop here
for (size_t i = container.size()-1; i >= 0; i--)
{
    // do somthing
}
```

No, when we traverse a container in reversed direction with **unsigned integer**, it would cause **dead loop**.

#### Overflow reasons

Why would it cause dead loop, but **signed integer** would not? Actually dead loop usually can not be caught easily, because the program does not raise any intuitive marks. But, we could check the log or debug it step by step or set break points just in your new codes([TDD](https://en.wikipedia.org/wiki/Test-driven_development) suggests you do like this, and it usually works). 

Here, we simplify it like:

```c++
for (int i = 5; i >=0; i--)
{
    std::cout << "i = " << i << std::endl;
}
for (size_t j = 5; j >=0; j--)
{
     std::cout << "j = " << j << std::endl;
}
```
we might get(run on x64):

```bash
i = 5
i = 4
i = 3
i = 2
i = 1
i = 0 
j = 5
j = 4
j = 3
j = 2
j = 1
j = 0
j = 18446744073709551615
j = 18446744073709551614
j = 18446744073709551613
j = 18446744073709551612
j = 18446744073709551611
j = 18446744073709551610
j = 18446744073709551609
j = 18446744073709551608
j = 18446744073709551607
j = 18446744073709551606
...
j = 0
j = 18446744073709551615
j = 18446744073709551614
...
j = 0
j = 18446744073709551615
j = 18446744073709551614
...
```

the first loop stopped because `i` reached `-1`, while`j` is always positive number as a **unsigned type**. It will not be less than `0`, so the program did not stopped. 

`size_t` is a typedef of `long unsigned int`and we found that it would reach to the biggest 8-bytes unsigned integer(run on x64) and then decrease to 0 then repeat it again and again. The reason is that a unsigned integer will be represented by the type simply "wraps around"(sometimes called "modulo wrapping") when it is "overflow"(not real overflow, since the ISO standard states that for unsigned integers modulo wrapping is the defined behavior and the term overflow never applies:"a computation involving unsigned operands can never overflow."). 

That is when we try to store the number `-1`(which requires 9 bytes to represent) in a 8-byte unsigned integer, it would not overflow. Instead, if a value is out of range, it is divided by one greater than the largest number of the type, and only the remainder kept. The number `-1` it too less to fit in our 8-byte range of `0` to `18,446,744,073,709,551,615`. `1 `greater than the largest number of type is `18,446,744,073,709,551,616`, therefore, we divide `-1` by `18,446,744,073,709,551,616`, getting `1` remainder `18,446,744,073,709,551,615`, the remainder of `18,446,744,073,709,551,615` is what is stored.

When we store the number `280`(which requires 9 bits to represent) in a 8-bits unsigned integer, it would also not overflow. `280` is greater than `255`(the largest number of 8-bits unsigned type). `1` greater than `255` is `256`, so we divide `280` by `256`, getting `1` remainder `24` which would be stored. 

When we figure out the quiz, the solution may be simple, DO NOT use unsigned type when traverse a container with reverse direction.

```c++
/// always use signed type when traverse with reverse direction
for (int i = container.size()-1; i >= 0; i++)
{
    // ...
}

/// or use iterator
for (auto it = container.end()-1; it != container.begin(); it--)
{
    // ...
}
```

In common language, **unsigned integer wrap around** is sometimes incorrectly called "overflow" since the cause is identical to signed integer overflow which is undefine behavior in ISO standard. 

### Other overflow examples

Many notable bugs in video game history happened due to wrap around behavior with unsigned integers. 
In the arcade game Donkey Kong, it’s not possible to go past level 22 due to an bug that leaves the user with not enough bonus time to complete the level. 
In the PC game Civilization, Gandhi was known for being the first one to use nuclear weapons, which seems contrary to his normally passive nature. Gandhi’s aggression setting was normally set at 1, but if he went democratic, he’d get a -2 modifier. This wrapped around his aggression setting to 255, making him maximally aggressive!

### The controversy over unsigned numbers

Many developers (and some large development houses, such as Google)  believe that developers should generally avoid unsigned integers.

This is largely because of two behaviors that can cause problems.

First, consider the subtraction of two unsigned numbers, such as 3  and 5.  3 minus 5 is -2, but -2 isn’t representable as an unsigned  number.

```c++
unsigned int x = 3;
unsigned int y = 5;

std::cout << x-y << std::endl; // 4294967294
```

The occurs due to `-2` wrapping around to a number close to the top of the range of a 4-byte integer.

Second, unexpected behavior can result when you mix signed and  unsigned integers.  In the above example, even if one of the operands (x  or y) is signed, the same behavior will result!

Consider the following snippet:

```c++
void fun(unsigned int x)
{
    // ...
}

int bar()
{
    fun(-1);
}
```


The author of fun() was expecting someone to call this  function with only positive numbers.  But the caller is passing in *-1*.  What happens in this case?

The signed argument of *-1* gets implicitly converted to an  unsigned parameter.  *-1* isn’t in the range of an unsigned number, so it  wraps around to some large number (probably `4294967295`).  Then the program goes ballistic.  Worse, there’s no good way to guard against  this condition from happening.  C++ will freely convert between signed  and unsigned numbers, but it won’t do any range checking to make sure  you don’t overflow your type.

Many modern programming languages (such as Java and C#) either don’t include unsigned types, or limit their use. Python would raise a exception.

**Bjarne Stroustrup**,  the designer of C++, said, “Using an `unsigned` instead of an `int` to gain one more bit to represent positive integers is almost never a good idea”.

Unfortunately, due to some poor design choices in the C++ standard  library, completely avoiding unsigned numbers in C++ isn’t possible at  this point in time.

Warning

>   Avoid unsigned numbers whenever possible.  Don’t avoid negative  numbers by using unsigned types.  If you need a larger range, use a  larger signed type.If you do use unsigned numbers, take care not to mix signed and unsigned numbers.