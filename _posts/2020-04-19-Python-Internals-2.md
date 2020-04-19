---
layout: post
title: Walrus operator - Python Internals 2.
---

Python and walrus - our new mini zoo.

_Note: I'm also learning. If you find any wrong or misleading information, please write to me._  
_Note: All the code you may find at my github_

I decided this series will be about everything connected to Python. I will describe things I find courious and try to explain how those are working in details, often looking at it's impelementation. I wish you good lecture!

In Python 3.8 there was introduced [new assigment operator](https://docs.python.org/3/whatsnew/3.8.html#assignment-expressions) called _walrus_, which names comes from the animal it resembles of. The operator looks like follows `:=`.

Walrus-operator is another name for assignment expressions. According to the official documentation, it is a way to assign to variables within an expression using the notation NAME := expr. The Assignment expressions allow a value to be assigned to a variable, even a variable that doesn’t exist yet, in the context of expression rather than as a stand-alone statement[1].

Ok, so now you know what is it and where you can use it, but what is happening under the hood?
_NOTE: I'll be using two Python versions in this comparission - Python 3.6.9 and Python 3.8.2_

To discover what is happening we will use [dis](https://docs.python.org/3/library/dis.html) module, which disassemble Python bytecode. To do it we need an example code snippet, so here it comes:
```python
#Python 3.6.9
1. def func():
2.   a = [1,2,3,4]
3.   n = len(a)
4.   if n > 3:
5.     print(n)
	 
#Python 3.8.2
1. def func():
2.   a = [1,2,3,4]
3.   if (n := len(a)) > 3:
4.     print(n)
```

As you can see the difference here lies between lines 3., 4. from first case and line 4. from the second. That's basically the whole trick - we may change two lines of code into one. How it looks when tou disassemble it?

![1](/images/walrus/1.jpeg)

Bytecode for list definition and returning value is the same. The main difference here starts at 18th byte of dissassebled bytecode - for future notice left is always Python 3.6.9, and right is Python 3.8.2.

![2](/images/walrus/2.jpeg)

Bytecode of 3.6.9. seems pretty standard - load function and list to the stack, call function and store it in variable. Next, put variable and const on the stack, compare them and jump if false.
On the other hand in 3.8.2. there is this thing called `DUP_TOP`. The name does not telling us much (or does it...? ; ) ), so lets dive deep in Python 3.8.2 source code and the main interpreter loop, to find out what this is doing.
```c
1397         case TARGET(DUP_TOP): {
1398             PyObject *top = TOP();
1399             Py_INCREF(top);
1400             PUSH(top);
1401             FAST_DISPATCH();
1402         }
```
So first things first. `TOP()` is just a macro to get the top value from the stack:
```c
 952 #define TOP()             (stack_pointer[-1])
```

`Py_INCREF` simply incremetns the reference count of an object - `top` in this case.
`PUSH` just pushes things on the top of the stack
```c
963 #define BASIC_PUSH(v)     (*stack_pointer++ = (v))
```
The last thing is `FAST_DISPATCH`, defined as follows:
```c
 876 #define FAST_DISPATCH() goto fast_next_opcode
```
Which basically takes next opcodes from Python bytecode.

To be honest I was hoping for some more research, but it seems, that _walrus_ is optimizing pushing and poping things from stack, by pushing the result of function call again to the stack and incrementing it's reference count, so after the `STORE_FAST` is executed, there is another copy of variable, which is used in comparision, which helps us to understand that `DUP_TOP` is a shortcut from `DUPLICATE_TOP`.

----
****
1. https://www.geeksforgeeks.org/walrus-operator-in-python-3-8/