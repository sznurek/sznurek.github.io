---
layout: post
title: C++ templates are Turing-complete
tags: programming
---

Here is something interesting: [C++ templates are Turing-complete](https://en.wikipedia.org/wiki/Template_metaprogramming)
and programming in them feels like pure functional programming.

## Basics: LISP-style lists

Let's start with something simple: single linked lists in the style of LISP and
some basic operations on them. We need to declare our datatype:

{% highlight cpp %}
struct Nil;
template <typename x, typename xs> struct Cons;
{% endhighlight %}

Values in template-based programming language will be types in C++. For example
we can declare list with three element:

{% highlight cpp %}
struct Elem;
typedef Cons<Elem, Cons<Elem, Cons<Elem, Nil> > > TheList;
{% endhighlight %}

So, if data structures are generic types how can we encode functions? Basic idea
is to have one `struct` per function and use template specialization
for doing pattern matching. To store results of computation we `typedef`
it into `struct`. Check out `Head` and `Tail` functions:

{% highlight cpp %}
template <typename l> struct Head;
template <typename x, typename xs> struct Head<Cons<x, xs> > {
    typedef x result;
};

template <typename l> struct Tail;
template <typename x, typename xs> struct Tail<Cons<x, xs> > {
    typedef xs result;
};

// Example usage:
typedef Head<TheList>::result TheHead;
typedef Tail<TheList>::result TheTail;
{% endhighlight %}

We need one more thing before going on: recursive functions. Good example will be
`append` function. Explanation why `typename` is needed at result declaration
[can be found on StackOverflow](http://stackoverflow.com/questions/642229/why-do-i-need-to-use-typedef-typename-in-g-but-not-vs).

{% highlight cpp %}
template <typename xs, typename ys> struct Append;
template <typename ys> struct Append<Nil, ys> {
    typedef ys result;
};

template <typename x, typename xs, typename ys> struct Append<Cons<x, xs>, ys> {
    typedef typename Append<xs, ys>::result RecCall;
    typedef Cons<x, RecCall> result;
};

typedef Append<TheList, TheTail>::result LongerList;
{% endhighlight %}

Here is append function in Haskell:

{% highlight haskell %}
append [] ys     = ys
append (x:xs) ys = x : append xs ys
{% endhighlight %}

This is the same idea as the code above (if you ommit syntax differences).

## How to get results of template programs?

So far we could only check that our programs compile. To get something on
the screen we can for example create run-time values from compile-time
types ("reify" them, if you like this word). The following example will
reify and print compile-time natural numbers:

{% highlight cpp %}
#include <cstdio>

struct Zero;
template <typename n> struct Succ;

template <typename n> struct ReifyNat;
template <> struct ReifyNat<Zero> {
    enum { value = 0 };
};
template <typename n> struct ReifyNat<Succ<n> > {
    enum { value = 1 + ReifyNat<n>::value };
};

typedef Succ<Succ<Succ<Zero> > > Three;

int main() {
    printf("Three: %d\n", ReifyNat<Three>::value);
    return 0;
}
{% endhighlight %}

In analoguous way we can print lists of natural numbers:

{% highlight cpp %}
#include <vector>
using std::vector;

template <typename l> struct ReifyNatList;
template <> struct ReifyNatList<Nil> {
    vector<int> get() {
        vector<int> result;
        return result;
    }
};
template <typename n, typename xs> struct ReifyNatList<Cons<n, xs> > {
    vector<int> get() {
        vector<int> result = ReifyNatList<xs>().get();
        result.insert(result.begin(), ReifyNat<n>::value);
        return result;
    }
};

typedef Succ<Zero> One;
typedef Succ<One> Two;
typedef Succ<Two> Three;
typedef Cons<One, Cons<Two, Cons<Three, Nil> > > SmallList;
typedef Append<SmallList, SmallList>::result TheList;

int main() {
    vector<int> v = ReifyNatList<TheList>().get();
    for(int i = 0; i < v.size(); i++) {
        printf("%d ", v[i]);
    }
    printf("\n");
};
{% endhighlight %}

## Branching
Before we will checkout higher-order function (yes, we will!) we need to implement `if`
instruction. After that we will be able to get maximum number from list of natural numbers.

Because all function definitions are done through pattern matching.
