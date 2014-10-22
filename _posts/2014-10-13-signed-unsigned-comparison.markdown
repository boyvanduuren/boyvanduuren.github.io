---
layout: post
comments: true
title:  "Comparison of unsigned integers to signed integers"
date:   2014-10-13
categories: C puzzle integer signed unsigned
---
Browsing reddit, as I often do, I came across a post about [C puzzles][cpuzzles].
This page contains source code to various (simple) programs that contain subtle errors. The goal is, of course, to find these errors.

Starting out with the first one, we come across the following source:
{% highlight C %}
#include<stdio.h>

#define TOTAL_ELEMENTS (sizeof(array) / sizeof(array[0]))
int array[] = {23,34,12,17,204,99,16};

int main()
{
    int d;

    for(d=-1;d <= (TOTAL_ELEMENTS-2);d++)
        printf("%d\n",array[d+1]);

    return 0;
}
{% endhighlight %}

Looking at above code, you'd think this program would simply loop through the array, printing the integers it contains. That is not the case. This program will print nothing.

So, what is happening?
For this code not to print anything, the `d <= (TOTAL_ELEMENTS-2)` expression should be false on the first iteration, causing the for-loop to end immediately.
Let's check if that's true:
{% highlight C %}
#include <stdio.h>

#define TOTAL_ELEMENTS (sizeof(array) / sizeof(array[0]))
int array[] = {23,34,12,17,204,99,16};

int main()
{
    int d = -1;

    if (d <= (TOTAL_ELEMENTS-2)) {
        printf("%d <= %d\n", d, TOTAL_ELEMENTS-2);
    }
    else {
        printf("%d > %d\n", d, TOTAL_ELEMENTS-2);
    }

    return 0;
}
{% endhighlight %}
{% highlight bash %}
# gcc test.c&&./a.out
-1 > 5
{% endhighlight %}
That expression does evaluate to false, but how?

`TOTAL_ELEMENTS` stores the result of `(sizeof(array) / sizeof(array[0]))`. Because `sizeof()` [returns an unsigned integral type which is usually denoted by size_t][wiki_sizeof], `TOTAL_ELEMENTS` will be unsigned.
The other variable, `d`, is a signed integer.

When comparing a signed integer to an unsigned integer, the signed variable will be converted to unsigned. This is done by adding `UINT_MAX+1` to the mathematical value of the signed integer, in accordance to the C standard. This actually doesn't change anything in memory as `-1+UINT_MAX+1=4294967295`, `-1` in [two's complement][2complement] and `4294967295` are both presented in memory as `0xffffffff`.

Below is a small snippet of the disassembled binary, showing the resulting instructions from the if-statement.
{% highlight asm %}
movl   $0xffffffff,-0x4(%rbp)   ; put value of d on the stack
mov    -0x4(%rbp),%eax          ; moves the value of d into %eax
cmp    $0x5,%eax                ; compare TOTAL_ELEMENTS to d
ja     0x400538                 ; if d > TOTAL_ELEMENTS jump to 0x400538
{% endhighlight %}
The `ja` instruction is the instruction we are looking at. `ja` does an **unsigned** comparison.

Lets change our last C program to explicitly cast `TOTAL_ELEMENTS` to a signed integer.
{% highlight C %}
    if (d <= (signed int)(TOTAL_ELEMENTS-2)) {
{% endhighlight %}

The resulting assembly looks as follows.
{% highlight asm %}
movl   $0xffffffff,-0x4(%rbp)   ; put value of d on the stack
cmpl   $0x5,-0x4(%rbp)          ; compare TOTAL_ELEMENTS to d
jg     400536                   ; if d > TOTAL_ELEMENTS jump to 0x400536
{% endhighlight %}
We'll look at the instruction after the compare again. This time `jg` is used. Guess what? `jg` does a **signed** comparison. Causing the program to function as you'd think it would.

I'm still left with a question though, why do the binaries differ so much depending on whether we cast `TOTAL_ELEMENTS` or not. I'd think that this would just result in a different jump, `ja` or `jg`, but more is happening, for instance `cmp` is used in one case, `cmpl` in the other.

[cpuzzles]:     http://www.gowrikumar.com/c/index.php
[wiki_sizeof]:  https://en.wikipedia.org/wiki/Sizeof
[securecoding]: https://www.securecoding.cert.org/confluence/display/seccode/INT02-C.+Understand+integer+conversion+rules
[2complement]:  https://en.wikipedia.org/wiki/Two's_complement
