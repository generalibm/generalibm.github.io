---
title: How to specify a newer version gcc for cmake
date: 2019-04-10 09:46:49
tags: Linux
categories: cpp
keyword: [gcc, cmake, gdb]
description: A small piece of note for Linux toolkit.
toc: true
---

After I upgraded `gcc` from *version of 4.4.7* to *version of 7.4.0* on **CentOS 6.5**, some ridiculous bugs happened when I linked my project to `gtest`. So I rebuilt `gtest` using the newer `gcc`, however, bugs remained. In the end, I found that `cmake` did not use the newer `gcc` when I was typing `cmake ..` command, I guessed that was the key. 

<!--more-->

### How to upgrade `gcc` to a newer version

#### install development tools

```bash
yum groupinstall "Development Tools"
yum install glibc-static libstdc++-static
```

#### download `gcc` package

`http://mirror.hust.edu.cn/gnu/gcc/` turned out to be the most fastest mirror in China. 

#### build `gcc`

```bash
tar -Jxvf gcc-7.4.0.tar.xz
cd gcc-7.4.0
./contrib/download_prerequisits
mkdir BUILD
cd BUILD
../configure --enable-checking=release --enable-languages=c,c++ --disable-multilib
make -j4
sudo make install
```
Note that `./contrib/download_prerequisits` will download 4 packages:
- mpc-1.0.3.tar.gc

- mpfr-3.1.4.tar.bz2

- gmp-6.1.9.tar.bz2

- isl-0.16.1.tar.bz2

#### new feature support testing

v7.4.0 support c++17, here is a sample from [cppreference](https://en.cppreference.com/w/cpp/utility/any).

```c++
/// cpp17_any.cc
#include <any>
#include <iostream>
 
int main()
{
    std::cout << std::boolalpha;
 
    // any type
    std::any a = 1;
    std::cout << a.type().name() << ": " << std::any_cast<int>(a) << '\n';
    a = 3.14;
    std::cout << a.type().name() << ": " << std::any_cast<double>(a) << '\n';
    a = true;
    std::cout << a.type().name() << ": " << std::any_cast<bool>(a) << '\n';
 
    // bad cast
    try
    {
        a = 1;
        std::cout << std::any_cast<float>(a) << '\n';
    }
    catch (const std::bad_any_cast& e)
    {
        std::cout << e.what() << '\n';
    }
 
    // has value
    a = 1;
    if (a.has_value())
    {
        std::cout << a.type().name() << '\n';
    }
 
    // reset
    a.reset();
    if (!a.has_value())
    {
        std::cout << "no value\n";
    }
 
    // pointer to contained data
    a = 1;
    int* i = std::any_cast<int>(&a);
    std::cout << *i << "\n";
}
```

```bash
g++ cpp17_any.cc -std=c++17
./a.out
```

It would raise linking errors for libraries compatibility. 

#### update the  system newer`gcc` linker

```bash
find . -name *libstdc++* # find the newer lib
strings ./prev-x86_64-pc-linux-gnu/libstdc++-v3/src/.libs/libstdc++.so.6.0.24 | grep 'CXXABI' 
strings ./prev-x86_64-pc-linux-gnu/libstdc++-v3/src/.libs/libstdc++.so.6.0.24 | grep 'GLIBCXX'
sudo cp  ./prev-x86_64-pc-linux-gnu/libstdc++-v3/src/.libs/libstdc++.so.6.0.24 /usr/lib64/
strings /usr/lib64/libstdc++.so.6.0.24 | grep 'GLIBCXX' # check lib
strings /usr/lib64/libstdc++.so.6.0.24 | grep 'CXXABI' 
sudo rm /usr/lib64/libstdc++.so.6
sudo ln -s /usr/lib64/libstdc++.so.6.0.24 /usr/lib64/libstdc++.so.6
strings /usr/lib64/libstdc++.so.6 | grep 'CXXABI' # check lib
strings /usr/lib64/libstdc++.so.6 | grep 'GLIBCXX'
```

compile and run  again the output would be:

```bash
i: 1
d: 3.14
b: true
bad any_cast
i
no value
1
```

Note that `gdb` would not be compatible for the newer `gcc`, you should upgrade it too.


### How to specify the newer version `gcc `for `cmake`

#### bad approach

For the sake of specify the newer version of `gcc`, I tried to modify the **CMakeList.txt** by adding some like the following:

```bash
set(CMAKE_C_COMPILER "/usr/local/bin/gcc")
set(CMAKE_CXX_COMPILER "/usr/local/bin/g++")
```

which reminds me it is a bad idea, even though I had backed up a copy.

#### ideal approach

BUT, `export` is [a ideal approach](https://stackoverflow.com/questions/17275348/how-to-specify-new-gcc-path-for-cmake). that is exporting system variables of `CXX` and `CC` to CMake's cache before `cmake`:

```bash
export CC=/usr/local/bin/gcc 
export CXX=/usr/local/bin/g++
```

The export only needs to be done once, the first time you configure  the project, then those values will be read from the CMake cache.

#### reasonable explaination

I recommend against overriding the `CMAKE_C(XX)_COMPILER`  value for two main reasons: because it won't play well with CMake's  cache and because it breaks compiler checks and tooling detection.

When using the `set` command, you have three options: 

- without cache, to create a normal variable
- with cache, to create a cached variable
- force cache, to always force the cache value when configuring

Let's see what happens for the three possible calls to `set`:

**Without cache**

```bash
set(CMAKE_C_COMPILER /usr/bin/clang)
set(CMAKE_CXX_COMPILER /usr/bin/clang++)
```

When doing this, you create a "normal" variable `CMAKE_C(XX)_COMPILER`  that hides the cache variable of the same name. That means your  compiler is now hard-coded in your build script and you cannot give it a  custom value. This will be a problem if you have multiple build  environments with different compilers. You could just update your script  each time you want to use a different compiler, but that removes the  value of using CMake in the first place.

Ok, then, let's update the cache...

**With cache**

```bash
set(CMAKE_C_COMPILER /usr/bin/clang CACHE PATH "")
set(CMAKE_CXX_COMPILER /usr/bin/clang++ CACHE PATH "")
```

This version will just "not work". The `CMAKE_C(XX)_COMPILER` variable is already in the cache, so it won't get updated unless you force it.

Ah... let's use the force, then...

**Force cache**

```bash
set(CMAKE_C_COMPILER /usr/bin/clang CACHE PATH "" FORCE)
set(CMAKE_CXX_COMPILER /usr/bin/clang++ CACHE PATH "" FORCE)
```

This is almost the same as the "normal" variable version, the only  difference is your value will be set in the cache, so users can see it.  But any change will be overwritten by the `set` command.

**Breaking compiler checks and tooling**

Early in the configuration process, CMake performs checks on the  compiler: Does it work? Is it able to produce executables? etc. It also  uses the compiler to detect related tools, like `ar` and `ranlib`. When you override the compiler value in a script, it's "too late", all checks and detections are already done.

For instance, on my machine with gcc as default compiler, when using the `set` command to `/usr/bin/clang`, `ar` is set to `/usr/bin/gcc-ar-7`. When using an export before running CMake it is set to `/usr/lib/llvm-3.8/bin/llvm-ar`.

### How to upgrade `gdb`

#### download the newest version of `gdb`

`http://mirror.hust.edu.cn/gnu/gdb/`

#### specify the newer `gcc` version to build

```bash
mkdir BUILD & cd BUILD/
export CC=/usr/local/bin/gcc
export CXX=/usr/local/bin/g++
```

#### make it support TUI

```bash
../configure --enable-tui --with-python=yes
make -j8
sudo make install
```

#### make it support STL pretty print message

**copy the following script to local with name like `.gdbinit`** 

```python
#                                                                                                        
#   STL GDB evaluators/views/utilities - 1.03
#
#   The new GDB commands:                                                         
# 	    are entirely non instrumental                                             
# 	    do not depend on any "inline"(s) - e.g. size(), [], etc
#       are extremely tolerant to debugger settings
#                                                                                 
#   This file should be "included" in .gdbinit as following:
#   source stl-views.gdb or just paste it into your .gdbinit file
#
#   The following STL containers are currently supported:
#
#       std::vector<T> -- via pvector command
#       std::list<T> -- via plist or plist_member command
#       std::map<T,T> -- via pmap or pmap_member command
#       std::multimap<T,T> -- via pmap or pmap_member command
#       std::set<T> -- via pset command
#       std::multiset<T> -- via pset command
#       std::deque<T> -- via pdequeue command
#       std::stack<T> -- via pstack command
#       std::queue<T> -- via pqueue command
#       std::priority_queue<T> -- via ppqueue command
#       std::bitset<n> -- via pbitset command
#       std::string -- via pstring command
#       std::widestring -- via pwstring command
#
#   The end of this file contains (optional) C++ beautifiers
#   Make sure your debugger supports $argc
#
#   Simple GDB Macros writen by Dan Marinescu (H-PhD) - License GPL
#   Inspired by intial work of Tom Malnar, 
#     Tony Novac (PhD) / Cornell / Stanford,
#     Gilad Mishne (PhD) and Many Many Others.
#   Contact: dan_c_marinescu@yahoo.com (Subject: STL)
#
#   Modified to work with g++ 4.3 by Anders Elton
#   Also added _member functions, that instead of printing the entire class in map, prints a member.



#
# std::vector<>
#

define pvector
	if $argc == 0
		help pvector
	else
		set $size = $arg0._M_impl._M_finish - $arg0._M_impl._M_start
		set $capacity = $arg0._M_impl._M_end_of_storage - $arg0._M_impl._M_start
		set $size_max = $size - 1
	end
	if $argc == 1
		set $i = 0
		while $i < $size
			printf "elem[%u]: ", $i
			p *($arg0._M_impl._M_start + $i)
			set $i++
		end
	end
	if $argc == 2
		set $idx = $arg1
		if $idx < 0 || $idx > $size_max
			printf "idx1, idx2 are not in acceptable range: [0..%u].\n", $size_max
		else
			printf "elem[%u]: ", $idx
			p *($arg0._M_impl._M_start + $idx)
		end
	end
	if $argc == 3
	  set $start_idx = $arg1
	  set $stop_idx = $arg2
	  if $start_idx > $stop_idx
	    set $tmp_idx = $start_idx
	    set $start_idx = $stop_idx
	    set $stop_idx = $tmp_idx
	  end
	  if $start_idx < 0 || $stop_idx < 0 || $start_idx > $size_max || $stop_idx > $size_max
	    printf "idx1, idx2 are not in acceptable range: [0..%u].\n", $size_max
	  else
	    set $i = $start_idx
		while $i <= $stop_idx
			printf "elem[%u]: ", $i
			p *($arg0._M_impl._M_start + $i)
			set $i++
		end
	  end
	end
	if $argc > 0
		printf "Vector size = %u\n", $size
		printf "Vector capacity = %u\n", $capacity
		printf "Element "
		whatis $arg0._M_impl._M_start
	end
end

document pvector
	Prints std::vector<T> information.
	Syntax: pvector <vector> <idx1> <idx2>
	Note: idx, idx1 and idx2 must be in acceptable range [0..<vector>.size()-1].
	Examples:
	pvector v - Prints vector content, size, capacity and T typedef
	pvector v 0 - Prints element[idx] from vector
	pvector v 1 2 - Prints elements in range [idx1..idx2] from vector
end 

#
# std::list<>
#

define plist
	if $argc == 0
		help plist
	else
		set $head = &$arg0._M_impl._M_node
		set $current = $arg0._M_impl._M_node._M_next
		set $size = 0
		while $current != $head
			if $argc == 2
				printf "elem[%u]: ", $size
				p *($arg1*)($current + 1)
			end
			if $argc == 3
				if $size == $arg2
					printf "elem[%u]: ", $size
					p *($arg1*)($current + 1)
				end
			end
			set $current = $current._M_next
			set $size++
		end
		printf "List size = %u \n", $size
		if $argc == 1
			printf "List "
			whatis $arg0
			printf "Use plist <variable_name> <element_type> to see the elements in the list.\n"
		end
	end
end

document plist
	Prints std::list<T> information.
	Syntax: plist <list> <T> <idx>: Prints list size, if T defined all elements or just element at idx
	Examples:
	plist l - prints list size and definition
	plist l int - prints all elements and list size
	plist l int 2 - prints the third element in the list (if exists) and list size
end

define plist_member
	if $argc == 0
		help plist_member
	else
		set $head = &$arg0._M_impl._M_node
		set $current = $arg0._M_impl._M_node._M_next
		set $size = 0
		while $current != $head
			if $argc == 3
				printf "elem[%u]: ", $size
				p (*($arg1*)($current + 1)).$arg2
			end
			if $argc == 4
				if $size == $arg3
					printf "elem[%u]: ", $size
					p (*($arg1*)($current + 1)).$arg2
				end
			end
			set $current = $current._M_next
			set $size++
		end
		printf "List size = %u \n", $size
		if $argc == 1
			printf "List "
			whatis $arg0
			printf "Use plist_member <variable_name> <element_type> <member> to see the elements in the list.\n"
		end
	end
end

document plist_member
	Prints std::list<T> information.
	Syntax: plist <list> <T> <idx>: Prints list size, if T defined all elements or just element at idx
	Examples:
	plist_member l int member - prints all elements and list size
	plist_member l int member 2 - prints the third element in the list (if exists) and list size
end


#
# std::map and std::multimap
#

define pmap
	if $argc == 0
		help pmap
	else
		set $tree = $arg0
		set $i = 0
		set $node = $tree._M_t._M_impl._M_header._M_left
		set $end = $tree._M_t._M_impl._M_header
		set $tree_size = $tree._M_t._M_impl._M_node_count
		if $argc == 1
			printf "Map "
			whatis $tree
			printf "Use pmap <variable_name> <left_element_type> <right_element_type> to see the elements in the map.\n"
		end
		if $argc == 3
			while $i < $tree_size
				set $value = (void *)($node + 1)
				printf "elem[%u].left: ", $i
				p *($arg1*)$value
				set $value = $value + sizeof($arg1)
				printf "elem[%u].right: ", $i
				p *($arg2*)$value
				if $node._M_right != 0
					set $node = $node._M_right
					while $node._M_left != 0
						set $node = $node._M_left
					end
				else
					set $tmp_node = $node._M_parent
					while $node == $tmp_node._M_right
						set $node = $tmp_node
						set $tmp_node = $tmp_node._M_parent
					end
					if $node._M_right != $tmp_node
						set $node = $tmp_node
					end
				end
				set $i++
			end
		end
		if $argc == 4
			set $idx = $arg3
			set $ElementsFound = 0
			while $i < $tree_size
				set $value = (void *)($node + 1)
				if *($arg1*)$value == $idx
					printf "elem[%u].left: ", $i
					p *($arg1*)$value
					set $value = $value + sizeof($arg1)
					printf "elem[%u].right: ", $i
					p *($arg2*)$value
					set $ElementsFound++
				end
				if $node._M_right != 0
					set $node = $node._M_right
					while $node._M_left != 0
						set $node = $node._M_left
					end
				else
					set $tmp_node = $node._M_parent
					while $node == $tmp_node._M_right
						set $node = $tmp_node
						set $tmp_node = $tmp_node._M_parent
					end
					if $node._M_right != $tmp_node
						set $node = $tmp_node
					end
				end
				set $i++
			end
			printf "Number of elements found = %u\n", $ElementsFound
		end
		if $argc == 5
			set $idx1 = $arg3
			set $idx2 = $arg4
			set $ElementsFound = 0
			while $i < $tree_size
				set $value = (void *)($node + 1)
				set $valueLeft = *($arg1*)$value
				set $valueRight = *($arg2*)($value + sizeof($arg1))
				if $valueLeft == $idx1 && $valueRight == $idx2
					printf "elem[%u].left: ", $i
					p $valueLeft
					printf "elem[%u].right: ", $i
					p $valueRight
					set $ElementsFound++
				end
				if $node._M_right != 0
					set $node = $node._M_right
					while $node._M_left != 0
						set $node = $node._M_left
					end
				else
					set $tmp_node = $node._M_parent
					while $node == $tmp_node._M_right
						set $node = $tmp_node
						set $tmp_node = $tmp_node._M_parent
					end
					if $node._M_right != $tmp_node
						set $node = $tmp_node
					end
				end
				set $i++
			end
			printf "Number of elements found = %u\n", $ElementsFound
		end
		printf "Map size = %u\n", $tree_size
	end
end

document pmap
	Prints std::map<TLeft and TRight> or std::multimap<TLeft and TRight> information. Works for std::multimap as well.
	Syntax: pmap <map> <TtypeLeft> <TypeRight> <valLeft> <valRight>: Prints map size, if T defined all elements or just element(s) with val(s)
	Examples:
	pmap m - prints map size and definition
	pmap m int int - prints all elements and map size
	pmap m int int 20 - prints the element(s) with left-value = 20 (if any) and map size
	pmap m int int 20 200 - prints the element(s) with left-value = 20 and right-value = 200 (if any) and map size
end


define pmap_member
	if $argc == 0
		help pmap_member
	else
		set $tree = $arg0
		set $i = 0
		set $node = $tree._M_t._M_impl._M_header._M_left
		set $end = $tree._M_t._M_impl._M_header
		set $tree_size = $tree._M_t._M_impl._M_node_count
		if $argc == 1
			printf "Map "
			whatis $tree
			printf "Use pmap <variable_name> <left_element_type> <right_element_type> to see the elements in the map.\n"
		end
		if $argc == 5
			while $i < $tree_size
				set $value = (void *)($node + 1)
				printf "elem[%u].left: ", $i
				p (*($arg1*)$value).$arg2
				set $value = $value + sizeof($arg1)
				printf "elem[%u].right: ", $i
				p (*($arg3*)$value).$arg4
				if $node._M_right != 0
					set $node = $node._M_right
					while $node._M_left != 0
						set $node = $node._M_left
					end
				else
					set $tmp_node = $node._M_parent
					while $node == $tmp_node._M_right
						set $node = $tmp_node
						set $tmp_node = $tmp_node._M_parent
					end
					if $node._M_right != $tmp_node
						set $node = $tmp_node
					end
				end
				set $i++
			end
		end
		if $argc == 6
			set $idx = $arg5
			set $ElementsFound = 0
			while $i < $tree_size
				set $value = (void *)($node + 1)
				if *($arg1*)$value == $idx
					printf "elem[%u].left: ", $i
					p (*($arg1*)$value).$arg2
					set $value = $value + sizeof($arg1)
					printf "elem[%u].right: ", $i
					p (*($arg3*)$value).$arg4
					set $ElementsFound++
				end
				if $node._M_right != 0
					set $node = $node._M_right
					while $node._M_left != 0
						set $node = $node._M_left
					end
				else
					set $tmp_node = $node._M_parent
					while $node == $tmp_node._M_right
						set $node = $tmp_node
						set $tmp_node = $tmp_node._M_parent
					end
					if $node._M_right != $tmp_node
						set $node = $tmp_node
					end
				end
				set $i++
			end
			printf "Number of elements found = %u\n", $ElementsFound
		end
		printf "Map size = %u\n", $tree_size
	end
end

document pmap_member
	Prints std::map<TLeft and TRight> or std::multimap<TLeft and TRight> information. Works for std::multimap as well.
	Syntax: pmap <map> <TtypeLeft> <TypeRight> <valLeft> <valRight>: Prints map size, if T defined all elements or just element(s) with val(s)
	Examples:
	pmap_member m class1 member1 class2 member2 - prints class1.member1 : class2.member2
	pmap_member m class1 member1 class2 member2 lvalue - prints class1.member1 : class2.member2 where class1 == lvalue
end


#
# std::set and std::multiset
#

define pset
	if $argc == 0
		help pset
	else
		set $tree = $arg0
		set $i = 0
		set $node = $tree._M_t._M_impl._M_header._M_left
		set $end = $tree._M_t._M_impl._M_header
		set $tree_size = $tree._M_t._M_impl._M_node_count
		if $argc == 1
			printf "Set "
			whatis $tree
			printf "Use pset <variable_name> <element_type> to see the elements in the set.\n"
		end
		if $argc == 2
			while $i < $tree_size
				set $value = (void *)($node + 1)
				printf "elem[%u]: ", $i
				p *($arg1*)$value
				if $node._M_right != 0
					set $node = $node._M_right
					while $node._M_left != 0
						set $node = $node._M_left
					end
				else
					set $tmp_node = $node._M_parent
					while $node == $tmp_node._M_right
						set $node = $tmp_node
						set $tmp_node = $tmp_node._M_parent
					end
					if $node._M_right != $tmp_node
						set $node = $tmp_node
					end
				end
				set $i++
			end
		end
		if $argc == 3
			set $idx = $arg2
			set $ElementsFound = 0
			while $i < $tree_size
				set $value = (void *)($node + 1)
				if *($arg1*)$value == $idx
					printf "elem[%u]: ", $i
					p *($arg1*)$value
					set $ElementsFound++
				end
				if $node._M_right != 0
					set $node = $node._M_right
					while $node._M_left != 0
						set $node = $node._M_left
					end
				else
					set $tmp_node = $node._M_parent
					while $node == $tmp_node._M_right
						set $node = $tmp_node
						set $tmp_node = $tmp_node._M_parent
					end
					if $node._M_right != $tmp_node
						set $node = $tmp_node
					end
				end
				set $i++
			end
			printf "Number of elements found = %u\n", $ElementsFound
		end
		printf "Set size = %u\n", $tree_size
	end
end

document pset
	Prints std::set<T> or std::multiset<T> information. Works for std::multiset as well.
	Syntax: pset <set> <T> <val>: Prints set size, if T defined all elements or just element(s) having val
	Examples:
	pset s - prints set size and definition
	pset s int - prints all elements and the size of s
	pset s int 20 - prints the element(s) with value = 20 (if any) and the size of s
end



#
# std::dequeue
#

define pdequeue
	if $argc == 0
		help pdequeue
	else
		set $size = 0
		set $start_cur = $arg0._M_impl._M_start._M_cur
		set $start_last = $arg0._M_impl._M_start._M_last
		set $start_stop = $start_last
		while $start_cur != $start_stop
			p *$start_cur
			set $start_cur++
			set $size++
		end
		set $finish_first = $arg0._M_impl._M_finish._M_first
		set $finish_cur = $arg0._M_impl._M_finish._M_cur
		set $finish_last = $arg0._M_impl._M_finish._M_last
		if $finish_cur < $finish_last
			set $finish_stop = $finish_cur
		else
			set $finish_stop = $finish_last
		end
		while $finish_first != $finish_stop
			p *$finish_first
			set $finish_first++
			set $size++
		end
		printf "Dequeue size = %u\n", $size
	end
end

document pdequeue
	Prints std::dequeue<T> information.
	Syntax: pdequeue <dequeue>: Prints dequeue size, if T defined all elements
	Deque elements are listed "left to right" (left-most stands for front and right-most stands for back)
	Example:
	pdequeue d - prints all elements and size of d
end



#
# std::stack
#

define pstack
	if $argc == 0
		help pstack
	else
		set $start_cur = $arg0.c._M_impl._M_start._M_cur
		set $finish_cur = $arg0.c._M_impl._M_finish._M_cur
		set $size = $finish_cur - $start_cur
        set $i = $size - 1
        while $i >= 0
            p *($start_cur + $i)
            set $i--
        end
		printf "Stack size = %u\n", $size
	end
end

document pstack
	Prints std::stack<T> information.
	Syntax: pstack <stack>: Prints all elements and size of the stack
	Stack elements are listed "top to buttom" (top-most element is the first to come on pop)
	Example:
	pstack s - prints all elements and the size of s
end



#
# std::queue
#

define pqueue
	if $argc == 0
		help pqueue
	else
		set $start_cur = $arg0.c._M_impl._M_start._M_cur
		set $finish_cur = $arg0.c._M_impl._M_finish._M_cur
		set $size = $finish_cur - $start_cur
        set $i = 0
        while $i < $size
            p *($start_cur + $i)
            set $i++
        end
		printf "Queue size = %u\n", $size
	end
end

document pqueue
	Prints std::queue<T> information.
	Syntax: pqueue <queue>: Prints all elements and the size of the queue
	Queue elements are listed "top to bottom" (top-most element is the first to come on pop)
	Example:
	pqueue q - prints all elements and the size of q
end



#
# std::priority_queue
#

define ppqueue
	if $argc == 0
		help ppqueue
	else
		set $size = $arg0.c._M_impl._M_finish - $arg0.c._M_impl._M_start
		set $capacity = $arg0.c._M_impl._M_end_of_storage - $arg0.c._M_impl._M_start
		set $i = $size - 1
		while $i >= 0
			p *($arg0.c._M_impl._M_start + $i)
			set $i--
		end
		printf "Priority queue size = %u\n", $size
		printf "Priority queue capacity = %u\n", $capacity
	end
end

document ppqueue
	Prints std::priority_queue<T> information.
	Syntax: ppqueue <priority_queue>: Prints all elements, size and capacity of the priority_queue
	Priority_queue elements are listed "top to buttom" (top-most element is the first to come on pop)
	Example:
	ppqueue pq - prints all elements, size and capacity of pq
end



#
# std::bitset
#

define pbitset
	if $argc == 0
		help pbitset
	else
        p /t $arg0._M_w
	end
end

document pbitset
	Prints std::bitset<n> information.
	Syntax: pbitset <bitset>: Prints all bits in bitset
	Example:
	pbitset b - prints all bits in b
end



#
# std::string
#

define pstring
	if $argc == 0
		help pstring
	else
		printf "String \t\t\t= \"%s\"\n", $arg0._M_data()
		printf "String size/length \t= %u\n", $arg0._M_rep()._M_length
		printf "String capacity \t= %u\n", $arg0._M_rep()._M_capacity
		printf "String ref-count \t= %d\n", $arg0._M_rep()._M_refcount
	end
end

document pstring
	Prints std::string information.
	Syntax: pstring <string>
	Example:
	pstring s - Prints content, size/length, capacity and ref-count of string s
end 

#
# std::wstring
#

define pwstring
	if $argc == 0
		help pwstring
	else
		call printf("WString \t\t= \"%ls\"\n", $arg0._M_data())
		printf "WString size/length \t= %u\n", $arg0._M_rep()._M_length
		printf "WString capacity \t= %u\n", $arg0._M_rep()._M_capacity
		printf "WString ref-count \t= %d\n", $arg0._M_rep()._M_refcount
	end
end

document pwstring
	Prints std::wstring information.
	Syntax: pwstring <wstring>
	Example:
	pwstring s - Prints content, size/length, capacity and ref-count of wstring s
end 

#
# C++ related beautifiers (optional)
#

set print pretty on
set print object on
set print static-members on
set print vtbl on
set print demangle on
set demangle-style gnu-v3
set print sevenbit-strings off

```

#### `gdb` testing example

**example code**

   ```c++
///  vector_test.cc

#include <iostream>
#include <vector>
 
int main()
{
    // Create a vector containing integers
    std::vector<int> v = {7, 5, 16, 8};
 
    // Add two more integers to vector
    v.push_back(25);
    v.push_back(13);
 
    // Iterate and print values of vector
    for(int n : v) {
        std::cout << n << '\n';
    }
    return 0;
}
   ```

compile it and debug using 

```bash
g++ vector_test.cc -std=c++11 -g
gdb a.out
```

**`gdb` console**

```bash
(gdb) b 9
(gdb) layout src		# set source code layout(TUI mode)
(gdb) r
(gdb) pvector v			# print all the vector elements
(gdb) n
(gdb) n
(gdb) pvector v 4 5      # check v[4] v[5]
```