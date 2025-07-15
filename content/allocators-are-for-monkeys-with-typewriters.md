+++
title = "Allocators are Monkeys With Typewriters"
description = "Or How easy it is to implement an allocator"
date = 2025-06-19
[extra]
thumbnail = "/static/images/monkey.png"
+++
{{ resize_image(path="./images/monkey.png", width=600, height=200, op="fit_width") }}
---

# Table of Contents
1. [Introduction](#introduction)
2. [Memory allocators](#memory-allocators)
3. [Buddy system](#buddy-system)
   1. [Allocation](#allocation)
   2. [Deallocation](#deallocation)
   
## Introduction
Recently I was looking at an issue on [mimalloc](https://github.com/microsoft/mimalloc/), a "_state-of-the-art_" memory allocator developed by Microsoft. [The issue](https://github.com/microsoft/mimalloc/issues/53#issuecomment-622976237) was quite simple, developers wanted a way to preallocate a piece of memory and use it as mimalloc's heap. Seeing that _mimalloc_ does not offer this feature, I thought:
>"_how hard can it be to write a memory allocator to manage a preallocated region?_".

The answer to this question is:
>"_given enough time, even a monkey with a typewriter can write a memory allocator_".

The implementation is around 163 LoC and very straightforward.

{{ resize_image(path="./images/mimalloc_issue.png", width=800, height=200, op="fit_width") }}

## Memory allocators
To write a memory allocator, we first need to understand what an allocator does. An allocator is "_a component of a programming language or runtime system that is responsible for managing the allocation and deallocation of memory during program execution_".[^1] Roughly speaking, a custom allocator needs to provide a way for the developer to allocate, deallocate and reallocate memory from the operating system. 

The C standard library provides `malloc/free/resize` (_C11 introduced [aligned_alloc](https://en.cppreference.com/w/c/memory/aligned_alloc) and C23 introduced [free_sized](https://en.cppreference.com/w/c/memory/free_sized) and [free_aligned_sized](https://en.cppreference.com/w/c/memory/free_aligned_sized)_), then a custom allocator will need to provide the same (_or nearly_) the same interface for easy integration with already existing code.

## Memory Fragmentation

One of the problems an allocator tries to solve is memory fragmentation. Memory fragmentation happens when the memory is split into small unused blocks throughout a memory chunk, making larger allocations harder because there is fewer contiguous memory available to allocate.

<img src="/images/memory_fragmentation.svg" />

Since memory is allocated in chunks, sometimes the real size allocated by the allocator is bigger than the requested size, and the unused memory left is wasted. This is called **internal fragmentation**[^2]. **External fragmentation** "arises when free memory is separated into small blocks and is interspersed by allocated memory"[^3].

There are a lot of methods an allocator can use to minimize memory fragmentation. General purpose ones do it by creating different allocation buckets based on the allocation size. So if an allocation of 512 bytes is requested, the allocator will use a bucket for small allocations, preventing that 512 bytes are wasted in the memory when big allocations happen.

## Buddy system

There are various allocation techniques, and one of the simplest and most reliable is the buddy allocation, which is the one I implemented. It is widely used in the industry, and it is the one used by the Linux kernel to handle page allocations (_albeit a modified version_).

The way it works is by dividing a contiguous memory chunk into two smaller chunks of memory, each a power of two (_hence the "buddy"_), until a small enough size is reached, and then returning this memory to the user.

### Allocation
As an example, suppose the programmer requested a 16KB memory chunk, and the size of the whole memory block is 256KB. It will do:

1. Requested size is smaller than the chunk size, it will split the 256KB chunk into two 128KB chunk.
2. Requested size is smaller than the current chunk size, first 128KB chunk is split in two 64KB chunk.
3. Requested size is smaller than the current chunk size, first 64KB chunk is split in two 32KB chunks.
4. Requested size is smaller than the current chunk size, first 32KB chunk is split in two 16KB chunks.
5. Requested size is equals the chunk size. Return it to the user.

Here is the representation of this allocation:
<img src="/images/buddy_allocator_internal.svg" />

And that is it, just split the memory until the requested size is `<=` than the current chunk size. The disadvantage of this technique is that it has internal fragmentation since the returned chunk can be smaller than the requested size and so the unused memory is wasted.

### Deallocation
To deallocate the memory is straightforward, just mark the chunk received by the `free` function as unused and check if its buddy is also unused, if it is, just coalesce it into a bigger chunk.

Suppose the previous 16KB memory chunk allocated is being freed, it would be like this[^4]:
<img src="/images/buddy_allocator_free.svg" />

### Resize
The resize part is also simple to implement, and it stays as a exercise for the reader.

### Conclusion
After spending some time reading and implementing an allocator, I was surprised by how simple an allocator is. Even general purpose allocators like *mimalloc* is only a few thousand lines of code and most of that code is (*probaly*) just dealing with multithread.

The code for my implementation can be found here: [tgmatos/imalloc](https://github.com/tgmatos/imalloc)

I would have loved to delve more into the internals of an allocator, the techniques general purpose allocators use and everything, but I have neither the time nor the proficiency (*yet*) to write an article about it. I will probably do it in the future, but for now my plan is to just implement a way for mimalloc to use preallocated memory.

[^1]: Definition taken from [What’s a Memory Allocator anyway ?](https://sumofbytes.com/blog/whats-a-memory-allocator-anyway)
[^2]: [Internal Memory Fragmentation (wikipedia)](https://en.wikipedia.org/w/index.php?title=Fragmentation_(computing)&useskin=vector#Internal_fragmentation)
[^3]: [External Memory Fragmentation (wikipedia)](https://en.wikipedia.org/w/index.php?title=Fragmentation_(computing)&useskin=vector#External_fragmentation)
[^4]: Since the only used chunk was a 16KB chunk, the `free` function would coalesce all the blocks until only a contigous 256KB chunk was available.
