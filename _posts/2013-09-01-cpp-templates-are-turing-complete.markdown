---
layout: post
title: C++ templates are Turing-complete
tags: programming
---

Here is something interesting: [C++ templates are Turing-complete](https://en.wikipedia.org/wiki/Template_metaprogramming)
and programming in them feels like pure functional programming.

*Disclaimer*: You can use C++ `int` type in template metaprogramming (just like in
linked Wikipedia article). This will be faster than my naive definition of
natural numbers below. But it is more fun to not use cheats and implement
everything from scratch.

Let's start with something simple: single linked lists in the style of LISP and
some basic operations on them. We need to declare our data type:

{% highlight cpp %}
struct Nil;
template <typename x, typename xs> struct Cons;
{% endhighlight %}

Values in template-based programming language will be types in C++. For example,
we can declare a list with three elements:

{% highlight cpp %}
struct Elem;
typedef Cons<Elem, Cons<Elem, Cons<Elem, Nil> > > TheList;
{% endhighlight %}

So, if data structures are generic types how can we encode functions? Basic idea
is to have one `struct` per function and use template specialization
for pattern matching. Results of computation are stored by `typedef` it into `struct`.
Check out `Head` and `Tail` functions:

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

This is the same idea as the code above (if you omit syntax differences).

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

In analogous way we can print lists of natural numbers:

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

Before we will checkout higher order functions (yes, we will!) we need to implement `if`
instruction. After that we will be able to get maximum number from list of natural numbers.

Just like all previous function definitions, `If` is also defined by pattern matching:

{% highlight cpp %}
struct True;
struct False;
template <typename b, typename then, typename els> struct If;

template <typename then, typename els>
struct If<True, then, els> {
    typedef then result;
};

template <typename then, typename els>
struct If<False, then, els> {
    typedef els result;
};
{% endhighlight %}

To demonstrate usage of `If` function let's look at less-than predicate for natural
numbers:

{% highlight cpp %}
template <typename n, typename m> struct Less;
// No natural number is strictly less than zero.
template <typename n> struct Less<n, Zero> {
    typedef False result;
};

// The first number greater than n is it's successor.
template <typename n> struct Less<n, Succ<n> > {
    typedef True result;
};

template <typename n, typename m> struct Less<n, Succ<m> > {
    typedef typename Less<n, m>::result result;
};


template <typename xs> struct Maximum;

template <typename n> struct Maximum<Cons<n, Nil> > {
    typedef n result;
};

template <typename n, typename xs> struct Maximum<Cons<n, xs> > {
    typedef typename Maximum<xs>::result RecCall;
    typedef typename Less<RecCall, n>::result Cmp;
    typedef typename If<Cmp, n, RecCall>::result result;

    // Can be also written as:
    // typedef typename If<typename Less<RecCall, n>::result, n, RecCall>::result result;
};
{% endhighlight %}

The last ingredient of template-based programming language will be higher
order functions "support" and obligatory `map` function on lists.

Arguments to our functions must be included in `template` parameter list
with `typename` keyword (and all free variables in pattern matches). If
we want to make use of the fact that one of our arguments is a generic class
(that is, it takes an type argument) we need special syntax. Below is a
function doing more or less the same thing as this Haskell function:

{% highlight haskell %}
apply :: (a -> b) -> a -> b
apply f x = f x
{% endhighlight %}

In C++ templates:

{% highlight cpp %}
template <template <typename x> class f, typename y> struct Apply {
    typedef typename f<y>::result result;
};
{% endhighlight %}

And as I promised, `Map`:

{% highlight cpp %}
template <template <typename x> class f, typename xs> struct Map;

template <template <typename x> class f> struct Map<f, Nil> {
    typedef Nil result;
};

template <template <typename x> class f, typename x, typename xs>
struct Map<f, Cons<x,xs> > {
    typedef typename Cons<typename f<x>::result, typename Map<f, xs>::result> result;
};
{% endhighlight %}

You can continue this awesome fun: write compile-time prime sieve,
implement simple functional language compiled to C++ templates,
create typelevel red-black trees... I just want to get email with
code included if anybody will do any of things listed above.
