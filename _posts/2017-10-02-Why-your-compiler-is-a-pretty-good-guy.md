---
layout: post
title: Why your compiler is a pretty good guy.
---
Note: I'm also learning. If you find any wrong or misleading information, please write to me.

Hello there!
So it's my very first post here (and in my whole life). Please note that English is not m native language, so there might be some mistakes. If you have any concernes just write about it to me!

Let's get it started!

I'm sure you heared about alignment your compiler does for you so you could sleep calmly. No? Then here's little explanation.

Every time you're making a struct or an union in C, compiler implicitly align data to be at least a perfect multiple of the lowest comon multiple of the alignments of all of the member of the struct or union, which is required by ISO C standard[1].
You may think "How is that possible? Can it be that smart?" and the answer is yes.
Let's take a look at the example.

{% gist 58bf7f292d64cca94d5b751fcbc9fb81 %}

As you can see we've got here simple struct with two ints, short and char. As we know in is 4 bytes, so we already have 8 bytes, then 2 bytes of short and 1 byte from char, so lets do some simple math: 4 + 4 + 2 + 1 = 11 bytes. 
Ok, 11 bytes, that's good. It's to fire up our sample program.

![Reslut of alignment.](/images/Result of alignment.png)

Wait, wait. We have got 11 bytes of data, but the size of struct is 12? And that is the alignment! We've got one byte added to our struct! 
But why?

Have you ever heard about atomic operations?
Atomic operation is an operation where access to for example 4 bytes of int is executed simultaneously. This kind of operations are performed on natural addresses. These are addresses that are completely divisible by our variable size (int, schort, char).
In sake of performance we want our addresses to be only natural. It they're not, operation of writing to our variable may took 2 or more writings instead of one.

Is there any way to check it?
In [1] you can find descriptions of specific attributes for types (documentation is for GCC 5.4.0).
There is one attribute which can make it possible and its name is "packed".
In the description we can read:
"This attribute, attached to struct or union type definition, specifies that each member (other than zero-width bit-fields) of the structure or union is placed to minimize the memory required."

It sounds exaclty what we are looking for. Let's modify the example code:

{% gist 62050b0e7318c0a158b4039b01a07151 %}

And the result:

![Reslut of packed alignment.](/images/alignment_packed.png) 

Now we've got 11 bytes! It works! But how does it affect performance?
As we are very curious there is no other way than check it by ourselves!
We will use time.h and again modify example code. We will perform two tests. One for default compiler alignment and one for packed alignment.

{% gist bc20af09e55d5dc729b12154e90f9852 %}

Lets take a look at the results. I have perfomed couple of tests for each of alignment just to take average time, which for deafault alignment was 0,415 ms.

![Time of execution for deafault alignment.](/images/normal_time.png) 

For the packed alignment on the other hand average time of execution was 0,448 ms.

![Time of execution for packed alignment.](/images/packed_time.png) 

Of course these are only tests which should show you that operations without alignment are a little bit slower, but these diffrencies may be omitted in our results. In cases where we've got much, much bigger structs and performing very complex operations and we need it to be as fast as possible, the compiler will do everything to asure that.

And it's end for today! Hope you enjoy it!




