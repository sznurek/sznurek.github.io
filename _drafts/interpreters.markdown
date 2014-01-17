---
layout: post
title: You should use defunctionalization more
tags: programming
---

Do you know what React.js and free monads have in common? Delayed interpretation.

## React.js revolution

Finally I had time to play with React.js and see what is the hype all about.
Long story short: instead of dynamic modification of DOM tree elements one
can just re-render everything from scratch.

Isn't this slow? Well, if behind the scenes React.js had really re-render
browser's representation of DOM tree it *would* have been slow. But React.js
does *not* render directly to the DOM.

The key idea behind React.js is that we don't make changes directly on
brower's DOM representation. We create a datatype of what the DOM tree
will look like. And after some processing we *interpret* it in the
real (browser's) DOM, Everything else follows from this idea.

It isn't clear yet how we avoid re-rendering whole DOM tree on every change.
The magic is happening before interpretation. React.js compares your new tree
and currently displayed tree to calculate minimum set of actions to mutate
current tree into next one.

Let me stop for a moment and consider what allows React.js to do that kind
of operations. It's certainly the abstract tree representation of DOM tree.
Instead of directly creating target DOM structure we are using *first order
representation*.

## Academic jargon warning

You hadn't stop reading after encountering phrase *first order representation*?
Good, good, the dark side is more powerful. When we say that something is first
order we mean that it does not contains functions (or rather closures). For example
this structure is first order:

{% highlight javascript %}

var firstOrder = [10, "abc", {1: "a"}];

{% endhighlight %}

and that one isn't:

{% highlight javascript %}

var higherOrder = [10, function(a) { return 10 * a; }];

{% endhighlight %}

Why do we care about such distinction? It is because first order structures has one
nice property: we can *inspect them fully*. Just look at the `higherOrder` example:
if somebody passed us this structure (as a argument to some function, for example)
we cannot inspect what this passed function *does*. Sure, we can pass some arguments
to it but we cannot decode it's whole meaning.

## Calculator example

Let's drift futher away from React.js (but only seemingly): we will write simple RPN
calculator in Javascript. It would look something like that:

{% highlight javascript %}

function eval(inputString) {
    var stack = [];
    var parts = inputString.split(" ");

    while(!parts.empty()) {
        if(parts[0] == '+') {
            var a = stack.pop();
            var b = stack.pop();

            stack.push(a + b);
        } else if(parts[0] == '*') {
            var a = stack.pop();
            var b = stack.pop();

            stack.push(a * b);
        } else {
            stack.push(parseInt(parts[0], 10));
        }

        parts.pop();
    }

    return stack[0];
}

{% endhighlight %}

Of course we are not concerned here with such trivialities as error handling ;) Example usage:

{% highlight javascript %}
    eval("+ * 2 3 4"); // => 10
    eval("* * * 2 2 2"); // => 8
{% endhighlight %}

Now consider following expression: `* 0 [some really big expression]`. It's clear what would
`eval` function do: even that we now that the result will be 0 the whole right hand side
part will be evaluated. How can we fix this? I'll show you one: we will separate commands
from mechanism:

{% highlight javascript %}
function parse(inputString) {
    var stack = [];
    var parts = inputString.split(" ");

    while(!parts.empty()) {
        if(parts[0] == '+' || parts[0] == '*') {
            var a = stack.pop();
            var b = stack.pop();

            stack.push({type: 'operator', op: parts[0], args: [a, b]});
        } else {
            stack.push({type: 'literal', value: parseInt(parts[0], 10)});
        }

        parts.pop();
    }

    return stack[0];
}

function evalTree(ast) {
    if(ast.type == 'literal') {
        return ast.value;
    } else if(ast.op == '+') {
        return eval(ast.args[0]) + eval(ast.args[1]);
    } else if(ast.op == '*') {
        return eval(ast.args[0]) * eval(ast.args[1]);
    }
}

// composition of parse and evalTree

function eval(inputString) {
    return evalTree(parse(inputString));
}

{% endhighlight %}

How do we perform optimisation mentioned before? We will insert another step between
`parse` and `evalTree`:

{% highlight javascript %}

// parse and evalTree are same as before

function literalEquals(ast, value) {
    return ast.type == 'literal' && ast.value == value;
}

function optimize(ast) {
    if(ast.type == 'literal') {
        return ast;
    }

    ast.args[0] = optimize(ast.args[0]);
    ast.args[1] = optimize(ast.args[1]);

    if(ast.op == '*' && (literalEquals(ast.args[0], 0) || literalEquals(ast.args[1], 0))) {
        return {type: 'literal', value: 0};
    }

    if(ast.op == '+' && literalEquals(ast.args[0], 0)) {
        return ast.args[1];
    }

    if(ast.op == '+' && literalEquals(ast.args[1], 0)) {
        return ast.args[0];
    }

    return ast;
}

function eval(inputString) {
    return evalTree(optimize(parse(inputString)));
}

{% endhighlight %}

## Yeah, that just basics of writing compilers/interpreters!

Yeah, you're right, but you know what? **That what's React.js all about!** React.js is "just"
a specialized intepreter (where browser's DOM tree is the assembly language we compile to).

## What enabled us to perform optimizations?

I the previous example our first `eval` function returned only a number and we didn't have
any insight into what's been computed. To aid that we have created *first order* intermediate
representation. Why it must be first order? Look at the folowing modification of parse function:

{% highlight javascript %}
function evalOp(op) {
    return function(a, b) {
        if(op == '+') {
            return a + b;
        } else if(op == '*') {
            return a * b;
        }
    };
}

function parse(inputString) {
    var stack = [];
    var parts = inputString.split(" ");

    while(!parts.empty()) {
        if(parts[0] == '+' || parts[0] == '*') {
            var a = stack.pop();
            var b = stack.pop();

            stack.push({type: 'operator', evalOp(op): , args: [a, b]});
        } else {
            stack.push({type: 'literal', value: parseInt(parts[0], 10)});
        }

        parts.pop();
    }

    return stack[0];
}

function evalTree(ast) {
    if(ast.type == 'literal') {
        return ast.value;
    }

    return ast.op(evalTree(ast.args[0], ast.args[1]));
}

{% endhighlight %}

So cleanly written, right? We have decomplected operator evaluation from rest of the code.
But look what happens when we want to write `optimize` function: we cannot check if we
are looking at multiplication or addition!

## Back to React.js

I hope that you have a clearer image of how React.js works. It creates from JSX template
language a abstract representation of a DOM tree. Now it compares newly generated and
already present tree (and it's fast, because we only scan first order structure and we don't
need to ask browser for information about DOM - and that would be slow) to produce "assembly
code" - sequence of low-level DOM operations that will transform current tree into desired.
The rest is just convinience wrappers.

## What about free monads?

TDB

## What about defunctionalization?

TDB

## Other examples

Doctrine and other ORMs, ...
