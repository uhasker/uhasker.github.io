---
layout: post
title:  "Write a Lisp in 16 lines of Python"
date:   2022-11-20 19:09:26 +0100
categories: jekyll update
---

_Disclaimer: Obvious inspiration is [obvious](https://norvig.com/lispy.html).
You can find the code for this writeup on [my GitHub](https://github.com/uhasker/stlisp)._

Imagine the following scenario: Due to an economic crisis the LOC strategic reserve is depleted.
Roving bands of programmers roam the streets, unable to use their favourite languages like *Java™®* or *JavaScript (ECMAScript ~~2016~~ ~~2017~~ ~~2018~~ ~~2019~~ ~~2020~~ ~~2021~~ 2022)*. The president of software development (of course there is a president, we are not *savages*) summons you to his office and tells you that his underlings have managed to fill the strategic reserve with 16 lines of code. Your job is to create an interpreter for a programming language that fits within those 16 lines. From there on *less sophisticated* developers will recreate the software ecosystem.

Sounds impossible?
Think writing your own programming language is hard?
Ok, yeah, it definitely is.
Actually, scratch that - creating a practical programming language is *super hard*.
However, writing an *basic* interpreter for a *simple* language that will satisfy the president of software development is much easier than it sounds.

During the course of this writeup we will craft an interpreter for a very simple Lisp dialect in just 16 lines of Python.
This will involve just a *tiny* bit of cheating - however *drastic times call for drastic measures*.
That Lisp dialect will be called **stlisp** (short for **s**uper **t**iny **Lisp**).

## An introduction to stlisp

The most basic thing to understand about Lisp-like languages is that there is no difference between statements and expressions.
For example in Python something like `a = 3` is a **statement**, but `x + y` is an **expression**.
However in Lisp there are *only* expressions which are evaluated by the interpreter.

The most basic expressions in Lisp are **atoms**.
The most important atoms are numbers:

{% highlight none %}
stlisp> 2
2
stlisp> 0.5
0.5
{% endhighlight %}

Another important Lisp concept is the **symbol**.
The technical definition of a symbol is "a way of locating expressions".
Basically you can think of symbols as variable or function names.
We introduce a symbol in stlisp using the `define` keyword:

{% highlight none %}
stlisp> ('define', 'x', 2)
None
{% endhighlight %}

Note that this symbol definition is an expression, just one that happens to not return anything meaningful.
We evaluate a symbol the same way we would evaluate a number.
The difference is that the evaluation of a symbol doesn't return the symbol itself, but the evaluation of the expression it refers to:

{% highlight none %}
stlisp> 'x'
2
{% endhighlight %}

Every Lisp needs to have a way to call **procedures** (which everyone knows as functions).
We do this by passing the name of the procedure together with its arguments to our interpreter:

{% highlight none %}
stlisp> ('add', 2, 4)
6
{% endhighlight %}

Our language has a bunch of obvious built-in procedures like `add`, `sub`, `mul`, `truediv`, `abs` etc.

So far, so simple.
But now we need some datastructures.
Luckily for us, this is Lisp - so all we really need are **lists**.
We create a list using the `cons` procedure:

{% highlight none %}
stlisp> ('cons', 1, 2)
(1, 2)
stlisp> ('cons', 1, ('cons', 2, 3))
(1, 2, 3)
stlisp> ('cons', 1, ('cons', 2, ('cons', 3, 4)))
(1, 2, 3, 4)
{% endhighlight %}

Note that we created the list **recursively**.
This is the case for pretty much any operation that would be iterative in a mainstream language.
But of course this is not the *mainstream*, this is *Lisp*.

To quote [James Iry](http://james-iry.blogspot.com/2009/05/brief-incomplete-and-mostly-wrong.html):

> In spite of its lack of popularity, LISP [...] remains an influential language in "key algorithmic techniques such as recursion and condescension"

Our language also has a bunch of useful procedures to work with lists, the most important of which are:
* `car` which gets the head of the list
* `cdr` which gets the tail of the list
* `len` which gets the length of the list

{% highlight none %}
stlisp> ('define', 'lst', ('cons', 1, ('cons', 2, ('cons', 3, 4))))
None
stlisp> 'lst'
(1, 2, 3, 4) 
stlisp> ('car', 'lst')
1
stlisp> ('cdr', 'lst')
(2, 3, 4)
stlisp> ('len', 'lst')
4
{% endhighlight %}

We can also define our own procedures using the `lambda` keyword.
A procedure has parameters and a body.
When we call a procedure with argument...
Actually, everyone reading these obscure writeups knows what happens when you call a function, so here are some examples to ensure you have seen the syntax of a procedure definition and call:

{% highlight none %}
stlisp> ('define', 'double', ('lambda', ('x',), ('mul', 'x', 2))))
None
stlisp> ('double', 4)
8
{% endhighlight %}

In order to have a capable language, we also need a way to capture values outside of a procedure (which is called a **closure**).
For example this should be valid:

{% highlight none %}
stlisp> ('define', 'x', 10)
stlisp> ('define', 'incX', ('lambda', (), ('set!', 'x', ('add', 'x', 1))))
None
stlisp> 'x'
11
{% endhighlight %}

Note that there are two important things going on in this example.
First we can reference the symbol `x` despite the fact that `x` is not a parameter - instead it comes from the **outer environment** of this function (which in this case is the environment of our interpreter).
Second we can use the `set!` expression to change the value from an outer environment.

These environments can be arbitrarily nested if we nest the procedures.
Whenever a symbol is not found in the environment of a procedure, we walk through the environments from the innermost to the outermost environment until we find the symbol.

We also need a way to express conditions, which is pretty simple:

{% highlight none %}
stlisp> ('if', ('eq', 4, 4), 2, 3)
2
{% endhighlight %}

Finally, we need the `quote` keyword which "evaluates" its argument by returning simply returning it.
This allows us to do some very powerful things, but for now we will mostly use it to get lists and strings:

{% highlight none %}
stlisp> ('quote', (1, 2, 3))
(1, 2, 3)
stlisp> ('quote', 'x')
x
stlisp> ('len', ('quote', 'hello'))
5
{% endhighlight %}

## Bulding an Abstract Syntax Tree

We will approach building the interpreter in two steps.
First, we need to build an abstract syntax tree.
Then we need to evaluate that tree.

Since this is a supposed to be a gimmick, we want to write as little code as possible, even if that means sacrifices and layoffs in the syntax department.
Additionally no one writes Lisp anyway, so everyone will think that is the way Lisp is supposed to look (*mwahahahaha*).

In order to cut down on syntax, we will ~~borrow~~ steal the syntax from Python tuples and then simply `eval` them:

{% highlight python %}
>>> eval("(1, 2, (3, 4), (5, 6, 7))")
(1, 2, (3, 4), (5, 6, 7))
{% endhighlight %}

This is not how you would create an AST for an actual Lisp - especially since we will now need to separate expressions with commas and write symbols inside single (or double) quotes.
But remember - we have a limited amount of lines of code and we do not want to waste them on something as mundane as a *parser*.

Another upside of this "syntax" choice is that this section is now over and we can move on to something more interesting.

## Evaluating the AST

The AST evaluation will be inside function called `stlisp_eval`.
This function takes an expression `expr` (remember than in Lisp everything is an expression) and returns the value `expr` evaluates to:

{% highlight python %}
def stlisp_eval(expr):
	match expr:
		...
{% endhighlight %}

The first case is the also simplest - if the expression is a number, it evalutes to that number:

{% highlight python %}
def stlisp_eval(expr):
	match expr:
		case int(expr) | float(expr): return expr
		...
{% endhighlight %}

Wow.
So much Lisp.
Such AST evaluation.

The next case is a bit more interesting.
If the expression is a symbol, we need to find the value the symbol refers to and return it.
Now we have an obvious problem - how do we find that value?
We need a way *store and retrieve symbols together with their values*.
To accomplish that, let us utilize one of most powerful datastructures known to man.
Behold the *mighty dictionary*!

In all seriousness, all we need to do is to evaluate our expressions is the concept of an **environment**.
Such an environment is just a dictionary, where the keys are the symbols and the values are the values the symbols refer to.
For example - this is how an environment could look like:

{% highlight python %}
{ "x": 2, "y": 4, "z": 6 }
{% endhighlight %}

To evaluate a symbol, we just get the respective value from the environment - this is a simple dictionary access:

{% highlight python %}
def stlisp_eval(env, expr):
	match expr:
		...
		case str(expr): return env[expr]
		...
{% endhighlight %}

In order to add symbols to that environment, we will utilize the `define` keyword.
Whenever a symbol is added, we evaluate the corresponding expression and store the symbol in the environment:

{% highlight python %}
def stlisp_eval(env, expr):
	match expr:
		...
		case ("define", sym, sym_expr): env[sym] = stlisp_eval(env, sym_expr)
		...
{% endhighlight %}

For example if we execute `("define", "z", ("add", 1, 2))` in the REPL we evaluate `("add", 1, 2)` to `3` and then store the symbol `z` together with the value `3` in the environment of the REPL.

## Procedures

So far, so good.
However to do anything useful in our language, we need a way to create and call procedures (yes, yes, functions, I know).
Since procedure calls are easier to implement than procedure creation, we will do this backwards and start by writing the logic nessary to call procedures.

Basically whenever the list doesn't begin with a reserved stlisp keyword (like `define`) we assume that we are dealing with a procedure call.
In that case we first get the procedure by calling `stlisp_eval(env, proc_name)`.
Then we evaluate all arguments by doing `[stlisp_eval(env, arg) for arg in args]`.
Finally we call the procedure with the evaluated arguments, i.e. we call `f(*evaluated_args)`.

{% highlight python %}
def stlisp_eval(env, expr):
	match expr:
		...
		case (proc_name, *args):
			f = stlisp_eval(env, proc_name)
			evaluated_args = [stlisp_eval(env, arg) for arg in args]
			return f(*evaluated_args)
{% endhighlight %}

Let us test this.
First we manually create an environment with a single procedure that adds two numbers together:

{% highlight python %}
env = {
	"add": lambda x, y: x + y
}
{% endhighlight %}

Now let us understand how our code would evalute an expression like `("add", 3, ("add", 4, 5))`:
* here `proc_name` is the string `add`
* `args` is the tuple of the arguments, i.e. `(3, ("add", 4, 5))`
* `f` is the result of evaluating the symbol `add`, thus `f` becomes `lambda x, y: x + y`
* `evaluated_args` is the tuple `(3, 9)` since `3` evaluates to `3` and `("add", 4, 5)` evaluates to `9` (here is that recursion again)
* finally `f(*evaluated_args)` is equivalent to `add(3, 9)`, resulting in `12`

Pretty simple.

However we are still lacking a mechanism for introducing a procedure to our environment without manually adding it to the underlying dictionary.

Recall the syntax for defining a procedure:

{% highlight none %}
stlisp> ('define', 'double', ('lambda', ('x',), ('mul', 'x', 2))))
None
{% endhighlight %}

The implementation idea is relatively obvious - whenever we see an expression of the form `('lambda', params, body)` we return a function that takes an arbitrary number of arguments (`*args`) and evaluates the body of the procedure within *some* environment (which should probably contain the arguments and parameters of the procedure):

{% highlight python %}
def stlisp_eval(env, expr):
	match expr:
		...
		case ("lambda", params, body): return lambda *args: stlisp_eval(..., body)
{% endhighlight %}

But how do we create that environment?
Procedures usually take parameters which we need to store somewhere.
However, it's probably (by which I mean *definitely*) a terrible idea to add all parameters to the global environment, so we can't just do `stlisp_eval(env, body)`.

Therefore every procedure should have it's own environment which stores its parameters and arguments.
But now we have another problem - we want to be able to reference the values from the outer environment of the procedure as discussed above:

{% highlight python %}
stlisp> ('define', 'x', 10)
stlisp> ('define', 'incX', ('lambda', (), ('set!', 'x', ('add', 'x', 1))))
None
stlisp> 'x'
11
{% endhighlight %}

We will accomplish this by creating an environment containing the parameters with the arguments as well as a reference to the "outer" environment of the procedure (which is the environment the procedure had access to on creation).

Therefore the environment of the procedure should be `dict(zip(params, args)) | {"__outer__": env}`, where `params` are the parameters, `args` the arguments and `env` the "outer" environment of the procedure.
For example this is how the environment of a procedure called with `x = 2, y = 4` would look like:

{% highlight python %}
env = {
	"x": 2,
	"y": 4,
	"__outer__": ...
}
{% endhighlight %}

Summarily, this code creates a brand new procedure:

{% highlight python %}
def stlisp_eval(env, expr):
	match expr:
		...
		case ("lambda", params, body): return lambda *args: stlisp_eval(dict(zip(params, args)) | {"__outer__": env}, body)
		...
{% endhighlight %}

This is how to create job security...

We need to make one more change.
Since we want to be able to find a symbol in any environment, we need to write some recursive logic to find the environment in which a symbol occurs in:

{% highlight python %}
def find_env(env, sym):
    return env if (env is None or sym in env) else find_env(env.get("__outer__", None), sym)
{% endhighlight %}

Basically, whenever we don't find a symbol in the environment `env`, we try looking in its outer environment by invoking `find_env` for the outer environment of `env`.

Finally we need to call `find_env` in for the case when an expression is a symbol so that symbol evaluation works correctly for procedures:

{% highlight python %}
def stlisp_eval(env, expr):
	match expr:
		...
		case str(expr): return find_env(env, expr)[expr]
		...
{% endhighlight %}


## Dirty hacking a global environment

Every good scripting language needs to have a bunch of useful built-in procedures in the global environment.
Even thought stlisp is a gimmick, it should not be a *completely* useless gimmick.
However, remember that we are tight on strategic code line reserves, so we need a cheap way to at least get a bunch of arithmetic operations, comparison operators and functions that can work with lists.
The builtins and the `operator` module of Python seem like a good way to accomplish that.

First we get the variables in `__builtins__` and the `operator` module:

{% highlight python %}
import operator as op
global_env = dict((vars(__builtins__) | vars(op)).items())
{% endhighlight %}

We should also remove everything that is not callable or a class.
Threfore the global environment becomes:

{% highlight python %}
global_env = {k: v for k, v in (vars(__builtins__) | vars(op)).items() if callable(v) and not isinstance(v, type)}
{% endhighlight %}

Please don't write dictionary comprehensions like this in production code...

The above comprehension provides us a bunch of necessary functions like `eq`, `ne`, `add`, `sub` and other arithmetic, comparison and list functions:

We will also add `cons`, `car`, `cdr` and `prog` (called `begin` in Scheme), because that's just good manners when writing a Lisp dialect:

{% highlight python %}
global_env |= {
	"cons": lambda x, y: (x, *y) if isinstance(y, tuple) else (x, y),
	"car": lambda x: x[0],
	"cdr": lambda x: x[1:],
	"prog": lambda *x: x[-1]
}
{% endhighlight %}

## More keywords

The implementations of `if`, `quote` and `set!` are pretty simple.
For a useful exercise, I suggest you think about them for yourself, otherwise here they are:

{% highlight python %}
match expr:
	...
        case ("quote", quote_arg): return quote_arg
        case ("if", test, conseq, alt): return stlisp_eval(env, (conseq if stlisp_eval(env, test) else alt))
        case ("set!", sym, sym_expr): find_env(env, sym)[sym] = stlisp_eval(env, sym_expr)
        ...
{% endhighlight %}

## The REPL

Now all we need is to add a REPL, so that we can actually use our interpreter.
We simply read input from the user, build an AST with `eval` and then evaluate the AST in the global environment by calling `stlisp_eval`:

{% highlight python %}
while True: print(stlisp_eval(global_env, eval(input("stlisp> "))))
{% endhighlight %}

This REPL is of course super dirty (because apart from the REPL everything else is perfect...).
The most obvious (and easily fixable) problem is that it is will crash on syntax errors.
Therefore to go beyond the gimmick, we would probably change it to something more readable and robust:

{% highlight python %}
while True:
	ast = eval(input("stlisp> "))
	try:
		eval_result = stlisp_eval(global_env, ast)
		print(eval_result)
	except Exception as e:
		print(f"Can't evaluate expression: {e}")
{% endhighlight %}

However, for the gimmick we will keep the REPL as is.
After all, the strategic reserves are now depleted and *real* men don't make syntax errors.

## We are done

So this is it!
We hacked together an interpreter for a simple Lisp dialect in just 16 lines of completely unreadable Python.
Since writing seemingly (but not really) impressive and completely unreadable code are the hallmarks of a 10x developer™, welcome to the club! Go write some 10x code.

Oh and please don't write stuff like that in production or I will haunt your dreams.


