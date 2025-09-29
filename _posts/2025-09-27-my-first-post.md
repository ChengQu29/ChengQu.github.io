---
layout: single
title: "Callbacks"
date: 2024-09-15
author: "Cheng Qu"
categories: [blog]
tags: [C]
---

Here, I share my understanding and explanation of callback function, one of the key building blocks of a multi-threading file sharing server I developed in the OMSCS program.. 

A callback is a function that you pass as an argument to another function. The receiving function can then invoke the callback at a particular point, allowing for flexible and resuable code.

In languages that treat function as first class citizen, such as Python and Javascript, this is very easy because you can pass them as arguments. Callbacks are the bread and butter of async programming in JavaScript. The callbacks and async behaviour make JS work so well with front end, because web apps are inherently interactive - responding to clicks, typing, scrolling etc. When such an event happens, JS says: "Here's this function I'll call later". 

Here is a code snippet for doing this:
```javascript
const button = document.getElementById('myButton');
button.addEventListener('click', () => {
    // some random color
    const color = '#3543543';
    button.style.backgroundColor = color;
});
```

Unlike Python or JavaScript, functions are not first-class citizens in C, meaning you can't assign them to variables, pass them as arguemtns, store them in data structures or return them from other functions etc. How does C perform 'callbacks'?

We can leverage the function pointer construct in C.
Here’s a small example:

```c
#include <stdio.h>

void print_number(int num) {
    printf("Number: %d\n", num);
}

void process_number(int num, void (*callback)(int)) {
    callback(num);
}

int main(void) {
    process_number(42, print_number);
    return 0;
}
```
With function pointer, you can 'sort of' pass function (address of function) around. You can also store it in variables, or call it indirectly. Just remember you still can’t create functions at runtime or attach state to them. The classic C Programming Language book by K&R chapter 5.11 provided an classical qsort algorithm that uses the function pointer and callback concepts. 

I use Java a lot at work, so it helps to compare this with Java's way for doing callbacks. In pre-Java 11, the callbacks can be achieved by using Java language's interface construct. This is Java's object oriented way of doing things. 

Here is the same little programe implemented in Java:

```java
// Define an interface with a method that will act as the callback
interface NumberProcessor {
    void process(int num);
}

class PrintNumber implements NumberProcessor {
    @Override
    public void process(int num) {
        System.out.println("Number: " + num);
    }
}

public class CallbackExample {

    public static void processNumber(int num, NumberProcessor callback) {
        callback.process(num);
    }

    public static void main(String[] args) {
        NumberProcessor printNumber = new PrintNumber();

        processNumber(42, printNumber);
    }
}
```
Because interface defines a contract, and if an object implements that interface it must provide the implementation. And object can be passed around in Java. At runtime, `processNumber` does not care which specific implementation it gets. This is exactly the callback pattern: a function is passed as an argument and invoked later. 

Java 11 introduced lambda expression, so implementing callbacks in Java looks similar to that of the first example in Javascript now:
```java
processNumber(42, n -> System.out.println("Print: " + n));
```

Believe it or not, this is one of the two main building blocks for building a multi-threading inter-process communication server!