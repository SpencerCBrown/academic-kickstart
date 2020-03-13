---
title: Userland Memory Allocation
summary: An overview of the process of creating a simple sub-page memory allocator for C/C++.
authors: ["Spencer Brown"]
tags: ["Operating Systems", "C/C++"]
categories: ["Operating Systems"]
date: "2019-02-05T00:00:00Z"
slides:
  # Choose a theme from https://github.com/hakimel/reveal.js#theming
  theme: league
  # Choose a code highlighting style (if highlighting enabled in `params.toml`)
  #   Light style: github. Dark style: dracula (default).
  highlight_style: dracula
---

# Userspace Memory Allocation in C++

Making dynamic structures possible.

---

## Overview

1. Motivation
2. Requirements
3. Implementation

---

## Getting Memory

Dynamic Allocation (Heap):
```c++
// C++
int *numArray = new int[5];
MyObject obj = new MyObject();

// C
char *buffer = malloc(10);
```

---

## How Memory Works

1. OS loads program from storage disk into memory - code, global variables, and empty space for the stack.

2. When a program needs more memory, it has to request it from the operating system
 
3. The OS grants permission for the program to have read/write access to an additional "block" of RAM. (Page)

---

## Problem

C and C++ expect to be able to create variables of arbitrary sizes.

The OS only gives out fixed-size pages - usually 4KB - or consecutive groups of pages.

Every time a variable/object is dynamically allocated, it will use some multiple of 4KB. Hugely inefficient for a list of 5 `int`s.

---

## Solution

A userspace memory allocator can break up large pages into smaller chunks of memory.

These smaller chunks can be used to store variables, without the huge waste of resources.

---

## Implementation: Naive Approach

The simplest allocator: linked list of free 'chunks'.

{{< figure src="bitmap.png"  >}}

```c++
class node {
  node* next;
  size_t size;
};
```

---

## Problems with Naive Approach

{{< figure src="bitmap2.png" title="Fragmentation"  >}}

* Where to allocate this size chunk?

* Worst-Case Linear Search?

---

## A Better Solution: Buddy System

Buddy system uses a binary tree concept to manage memory.

Blocks are split into powers of 2 sizes. 2, 4, 8, 16, 32, 64, 128, ... 2048, 4096

{{< figure src="bitmap3.png" title="Simplest Allocations"  >}}

---

A separate linked list is kept for each size, for faster access.

{{< figure src="bitmap4.png" title="More complex buddy system-managed page."  >}}

A little memory is wasted for block sizes in between powers of 2, but is still pretty insignificant - and predictable.

---

## Tree Structure of Buddy System

{{< figure src="buddytree.jpg" title="Conceptual tree of buddy system. Borrowed from: http://softpack.develop.trigger.pt/images.php"  >}}

---

## Recursively Allocating a Block
Block Allocate Routine:

1. Chunk of size `size` is requested.

2. `size` is rounded up to nearest power of 2.

3. Free list for that size is checked. 

4. If node in that free list exists, it is popped.

5. If not, call `Block Allocate Routine` recursively.

6. Split the block returned in half and return one.

---

```c++
node* Block_Allocate(size_t size) {
  size_t rounded_size = Round_Up(size);
  int block_size = Log(2, rounded_size);
  List<node*> freelist = FreeListArray[block_size];
  if (freelist.peek() != nullptr) {
    return freelist.pop();
  } else {
    /* Special case: size = 2^12 = 4096B = 1 Page. */
    if (block_size = 12)    return GetNewPageFromOS();
    
    node* parent_block = Block_Allocate(power(2, block_size));
    auto pairOfBlocks = Split(parent_block);
    node* buddy1 = pairOfBlocks[0];
    node* buddy2 = pairOfBlocks[1];
    freelist.append(buddy1);
    return buddy2;}}
```

---

## Freeing Memory Chunks

Freeing just entails re-inserting a chunk back into its corresponding free list.

---

## What is Coalescing?
What happens if we just free blocks, with no additional work?
{{< figure src="bitmap5.png"  >}}

---

## Recursively Coalescing Adjacent Buddies
The buddy system's tree representation shows an elegant way to coalesce.

1. For a freed block, check if its buddy exists as the same size and is also free.
2. If so, merge the two
3. Recursively call the coalescing routine to try the same with the newly merged block
4. Do this until no free buddy is found, or 4096B page is reached.


---

## Using Allocator
To actually use the allocator requires overriding the operator::new() and operator::delete operators in C++, so that when dynamically creating objects and variables the custom allocator will be called instead of the one built into the C++ standard library.

```c++
void * operator new(size_t size)
{
    return MyMalloc(size);
} 
  
void operator delete(void * p) 
{ 
    MyFree(p);
} 
```

---

# Questions?

[Project Source Code](https://github.com/SpencerCBrown/...)

[Slideshow URL](https://spencercbrown.net/slides/allocation)