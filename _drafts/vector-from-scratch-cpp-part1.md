---
layout: post
title: Vector from Scratch in C++, Part 1
date: 2019-08-25
tags: [cpp, learning]
---

This series of posts will walk through creating your own vector type from scratch in C++. It's aimed at people learning C++ or perhaps already using the language, but who want to see a relatively complete example which highlights many of the key points of the language. We'll start simple with a "C with Classes" coding style, and progressively add more features to our vector until it supports C++11 and beyond.

If you're coming from C, you'll be very aware of the fact that the language provides no automatically resizing containers out of the box. The humble C array is fixed at allocation time, whether that be on the stack or on the heap. If you need more space, it's completely up to you to handle all the messy details.

If you're coming from a higher level language, you probably take automatically resizing arrays for granted. Most higher level languages like Python provide these containers as a pre-defined building block. C++ is no exception here, providing `std::vector` as the standard resizing array type.

Implementing `std::vector` can be remarkably instructive, though. The implementation touches on several key features of the language, which you'll find yourself using in your own code over and over. A vector is a fundamentally a resource managing class, so there will definitely be interesting logic in the constructor, destructor, copy operations, and move operations. (Don't worry if you aren't familiar with some of those, will cover them in detail as we get there!) Vectors also touch on class templates and generic code, iterators, and operator overloading. It's a veritable buffet of C++ concepts!

As we implement our vector we'll write tests for it like good software engineers. Not just because we should though, but also to think through how we want the interface to work and to prove to ourselves that it's actually working. I'm not going to drag in any testing library dependency, though. We'll use what's built in to the standard library to get the job done.

Let's get started in the simplest possible way. Our `Vector` class is going to start by managing an array of integers up to a maximum size, declared at construction time. This isn't much better than a plain old C array of `int`s yet, but we'll grow the implementation one step at a time.

```c++
#include <stddef.h>

class Vector {
private:
  size_t capacity_;
  size_t size_;
  int* data_;
};
```

Our `Vector` class starts with just three data members, a capacity, a size, and a pointer to the data it's managing. Capacity and size might seem like the same thing, but there's an important difference. Capacity is how much total space is available in the chunk of memory we allocated for the data. The size is how much of that capacity is *currently occupied* by elements in the array. For example, we can have a `Vector` which has allocated space for 10 integers, but only have 3 of those spots currently taken up, with 7 more available for future expansion. In the future we'll worry about what happens when we run out of capacity, but for now we'll just allocate all of our capacity when the class is constructed and that's the most we can ever have.

```c++
#include <stddef.h>
#include <stdlib.h>

class Vector {
public:
  Vector(size_t max_capacity)
    : capacity_(max_capacity),
      size_(0),
      data_(nullptr) {
	  data_ = static_cast<int*>(malloc(capacity_ * sizeof(int)));    
  }
  
  ~Vector() {
	  free(data_);
  }

private:
  size_t capacity_;
  size_t size_;
  int* data_;
};
```

We start by implementing the basic resource management that our `Vector` class needs to perform. Whenever you write a class that manages a resource in C++ (whether that be memory, file descriptors, socket connections, or anything else) you generally want to acquire the resource in the constructor and release the resource in the destructor. This is an incredibly useful pattern with an absolutely terrible name. It's known as [RAII](https://en.m.wikipedia.org/wiki/Resource_acquisition_is_initialization), which stands for **R**esource **A**cquisition **I**s **I**nitialization.

The big benefit to RAII is that putting the clean up logic in the destructor means that the clean up code will automatically get run when this class goes out of scope (if allocated on the stack). For example, we can create a local variable in a function that uses our vector, like so:

```c++
void my_cool_function() {
  Vector my_vector(42); // space for 42 integers
  // use the vector...
}
```

No matter how this function is exited, the destructor for `my_vector` will always run, resulting on our memory being properly freed. It doesn't matter if we hit the end of the function, or return early in the middle somewhere, or even throw an exception. The cleanup will always happen because `my_vector`s destructor will run upon exit and free the memory.

Now, there's one tiny issue to resolve in the code we've written so far. Let's take a closer look at the constructor.

```c++
Vector(size_t max_capacity)
  : capacity_(max_capacity),
    size_(0),
    data_(nullptr) {
  data_ = static_cast<int*>(malloc(capacity_ * sizeof(int)));    
}
```

(If you aren't familiar with the `: capacity_(max_capacity), ...` bit, that's called the *initialization list*. You can use it set initial values of your member variables before the constructor body executes.)

The main problem with our constructor is that C++ is going to let us play a little too fast and loose with it. Take a look at the following code:

```c++
Vector my_vector = 42;
```

This probably looks like an obvious error to you. I can't assign an integer to a `Vector`! That's obviously a type error! But the compiler will happily accept this mistake as valid code, given the way our constructor is currently defined.

The problem here is something called implicit type conversion. The compiler tries to be helpful and convert types automatically from one to another if that would resolve a discrepancy between two types. In this case, the compiler says, "Hmm... I need a `Vector` but I only have an integer. I know! I can convert from `42` from an integer to a `Vector` by using the `Vector(int max_capacity)` constructor!

Now, on rare occasions, that *is* the behavior you want. For example, the `std::string` class uses this implicit type conversion to allow you to use a `const char*` string literal in a place where `std::string` is expected. That's a huge ergonomics advantage over having to write `std::string("my string literal")` all the time.

However, in *most* cases this is **not** the behavior you want. To disable these implicit type conversions, we can add the `explicit` keyword to our constructors which accept a single argument. Note that this is only necessary on constructors which take a single argument, because they are the only ones that potentially participate in implicit type conversions.

Our updated constructor looks like this:

```c++
explicit Vector(size_t max_capacity)
  : capacity_(max_capacity),
    size_(0),
    data_(nullptr) {
  data_ = static_cast<int*>(malloc(capacity_ * sizeof(int)));    
}
```

Let's add two more functions to our `Vector` class before we call it a day on Part 1. Let's add a `push_back()` function which adds integers to the back of the array, and a `print()` function which we'll use for debugging to make sure things are working properly. In the future we'll implement iterators and an overloaded indexing operator so we won't need this `print()` function, but those are coming in a future installment!

```c++
#include <stddef.h>
#include <stdio.h>
#include <stdlib.h>

class Vector {
public:
  Vector(size_t max_capacity)
    : capacity_(max_capacity),
      size_(0),
      data_(nullptr) {
	  data_ = static_cast<int*>(malloc(capacity_ * sizeof(int)));    
  }
  
  ~Vector() {
	  free(data_);
  }
  
  void push_back(int x) {
	  if (size_ == capacity_) {
		  // We're all full for now. Nothing we can do!
		  return;
	  }
	  
	  // Add x to the end of the array.
	  data_[size_] = x;
	  size_++;
  }
  
  void print() {
	  for (size_t i = 0; i < size_; ++i) {
		  printf("%d ", data_[i]);
	  }
	  printf("\n");
  }

private:
  size_t capacity_;
  size_t size_;
  int* data_;
};
```

These functions should both be fairly straightforward. Since our `Vector` doesn't support growing or resizing yet, when our size reaches our capacity there's nothing we can do, so we simply drop `x` on the floor and don't add it. If there is room though, we put `x` at the first unoccupied spot at the end of the array and increment the size to keep track of how full our array is.

The `print()` function just loops over all the elements that are currently in use (not the allocated, but unused ones though!) and prints them out with spaces between each one. We can see it in action by adding a `main()` function:

```c++
int main() {
	Vector my_vector(10);
	my_vector.push_back(1);
	my_vector.push_back(2);
	my_vector.push_back(3);
	my_vector.print();
	return 0;
}
```

```shell
$ g++ -Wall -Werror -o my_vector my_vector.cpp
$ ./my_vector
1 2 3
```

That about wraps it up for part 1! In the next part, we'll look at upgrading our vector to support resizing, as well as making it into a class template to support managing arrays of any type, not just integers!