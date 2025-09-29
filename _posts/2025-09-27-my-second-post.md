---
layout: single
title: "Memory Ownership"
date: 2024-10-25
author: "Cheng Qu"
categories: [blog]
tags: [C]
---

Welcome to my second post! This is my understanding of the memory ownership model, one of the key building blocks for a multi-threading high performance file sharing server. 

For languages that doesn't have garbage collection mechanism, such as C/C++, following good practices can greatly reduce the possibility of running into memory safety issues. 

The core problems when it comes to managing memory include double free, use-after-free or memory leak if a block of memory is not freed etc. One proposal for safely managing memory is called the [pointer ownership](https://www.sei.cmu.edu/blog/using-the-pointer-ownership-model-to-secure-memory-management-in-c-and-c/) model. 

Memory ownership means deciding who is responsible for freeing a given block of memory. The main idea (well..greatly simplified by me) is that assuming you know there is only one valid pointer to a block of memory, if you have the pointer you can free it. So the question is how do we make sure there is only one 'owner' to a block of memory?

One way to achieve this in C is to pass a pointer to the pointer (T**). <br> But why?

Passing T * gives the callee a copy of the pointer. If the callee frees that memory, the caller still has a dangling pointer (it doesn’t know the memory was freed).
Passing T ** lets the callee:
1. Free the memory.
2. Set the caller’s pointer to NULL.<br>

That way, the caller has reliable information about the ownership state:<br>
If *ptr == NULL, caller knows "I no longer own this memory."<br>
If *ptr != NULL, caller still owns it and must eventually free it.

This makes ownership explicit and enforceable.

Let's take a look at the problem here:
```c
void take_ownership_bad(int *ptr) {
    free(ptr);
    // ptr is a local copy, setting it NULL does NOT affect caller
    ptr = NULL;
}

int main(void) {
    int *p = malloc(sizeof(int));
    *p = 42;

    take_ownership_bad(p);

    // Problem: p is NOT NULL here, but points to freed memory
    if (p == NULL) {
        printf("main: pointer is NULL\n");
    } else {
        printf("main: pointer is NOT NULL, value = %d (dangling!)\n", *p); // crash
    }

    return 0;
}
```

But if I pass a pointer to pointer to the function, when the memory is freed, I can set the caller's pointer to NULL. This provides a way for the caller to know if a piece of memory is freed or not. 

```c
void take_ownership(int **ptr) {
    if (ptr != NULL && *ptr != NULL) {
        free(*ptr);          // free the memory
        *ptr = NULL;         // set caller's pointer to NULL
    }
}

int main(void) {
    int *p = malloc(sizeof(int));   // allocate memory
    if (p == NULL) {
        perror("malloc failed");
        return 1;
    }

    *p = 42;   
    // Pass pointer-to-pointer so callee can free and null it
    take_ownership(&p);

    // After ownership transfer, p is NULL
    if (p == NULL) {
        printf("main: pointer is NULL after take_ownership\n");
    } else {
        //
    }

    return 0;
}
```

You might ask: what does this have to do with writing a multi-threaded file sharing server?

In multi-threaded programming, each thread owns the memory on its own stack. A common error occurs when a programmer passes a pointer to memory from Thread T’s stack over to Thread S. This is only safe as long as the code running in Thread T ensures that the referenced memory isn’t released or overwritten.

Consider a scenario with three functions: X, Y, and Z.
Function X allocates some memory (let’s call the pointer M) and then calls Y. Y does some additional work and eventually calls Z, passing along M. When Z finishes its task, it frees M.

At first glance, you might ask: “Why would anyone design things this way?” If a single developer wrote all three functions, they likely wouldn’t. But in practice, developers rarely build an entire codebase from scratch—they integrate and reuse components written by others.

For example, the developer who wrote X knows that Z is responsible for freeing memory, so X doesn’t bother cleaning up M after calling Y. However, X is also trusting that Y behaves correctly—perhaps Y queues the work for later before eventually invoking Z.

To make this contract clearer, we can use double pointers. Instead of passing just M, X passes the address of its local variable holding M. 

This way:
* Y can free M and then set X’s variable to NULL, making it explicit that the pointer is no longer valid.
* If Y forgets to free the memory, X can detect this because its variable still points to valid memory and can clean it up itself.
* Similarly, Y can forward the address of X’s variable to Z, allowing Z to free the memory and update the pointer. If Y doesn’t need to track the state itself, it can just pass responsibility along.

In this way, the use of double pointers makes ownership and cleanup responsibilities explicit across the call chain, and ensures that all functions have a consistent view of whether the memory is still valid.

If you’ve taken the time to read this far, I’m truly flattered—and I can’t wait to show you how a high-performance, multi-threaded file-sharing server is built. Stay tuned!