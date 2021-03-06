---
title: Higher Order Programming
slug: higher-order-programming
date: 2014-11-24
layout: post
published: true
tags: [blog]
abstract: "map(divide(__, 2), __) and other excursions into elegant functional programming in PHP"
image: "http://verraes.net/img/posts/2014-11-24-higher-order-programming/sandwiches.png"
---



Let's have some fun with higher order programming in PHP. I'll start by showing how to program with Lambdalicious (or λlicious for friends) and introduce the real meat along the way. Don't worry too much about the dark magic that may appear to power some of the features of Lambdalicious. It's [on GitHub](https://github.com/mathiasverraes/lambdalicious) if you're curious. Just follow along and keep track of all the functions. 

**Note:** The code in this blogpost works with [Lambdalicious v0.1.0](https://github.com/mathiasverraes/lambdalicious/tree/v0.1.0). If you want to adapt this blog post to the current version, please send a [pull request](https://github.com/mathiasverraes/mathiasverraes.github.com/edit/master/_posts/2014-11-24-higher-order-programming.markdown).

But first, something smaller:

## Atoms

`@foobar` is an atom. You can think of an atom as a constant that has itself as a value. So the value of `@foobar` is `foobar`.

{% highlight php %}
<?php
require_once __DIR__ . 'path/to/src/Verraes/Lambdalicious/load.php';
assert(
    isatom(@foobar)
);
{% endhighlight %}

(Note that if you combine all the code in this post in a single file, it is executable and all assertions will pass.)

Unlike constants in PHP, atoms don't need to be defined before you use them. They are global and immutable.

Atoms are very useful for many things. Later on, we'll use them to refer to global functions, without needing to resort to quotes. In fact, the philosophy of λlicious is to remove syntactical noise, verbosity, and ceremony. To that effect, let's get rid of the `@` prefix. We can do that by affixing an atom using the `atom` function:

{% highlight php %}
<?php
atom(@foobar);
assert(
    isatom(foobar) // no @ prefix needed anymore
);
{% endhighlight %}

Once an atom is affixed, you can use it everywhere without the prefix. 

## Lists

Let's affix some atoms and make a list.

{% highlight php %}
<?php
atom(@a, @b, @c, @d);
assert(
    islist([a, b, c])
);
assert(isequal(
    cons(d, [a, b, c]),
    [d, a, b, c]
));
{% endhighlight %}

`cons($head, $tail)` constructs a list by adding `$head` to the front of the list `$tail`. In fact, the square brackets are nothing but a shortcut for a bunch of nested `cons`.

{% highlight php %}
<?php
assert(isequal(
    cons(a, cons(b, cons(c, []))),
    [a, b, c]
));
{% endhighlight %}

We can deconstruct lists by taking the `head`, which is the first element, or the `tail`, which is all the elements without the head.

{% highlight php %}
<?php
assert(isequal(
    head([a, b, c]),
    a
));
assert(isequal(
    tail([a, b, c]),
    [b, c]
));
{% endhighlight %}

Remember that `head` always returns an element, and `tail` returns a list.

## Functions

Now let's make a global function. Global means that we can reach it from anywhere.

{% highlight php %}
<?php
function half($x){ return divide($x, 2);}
assert(isequal(
    half(6),
    3
));
{% endhighlight %}

If we affix the function's name, we can refer to it by that atom.

{% highlight php %}
<?php
atom(@half);
assert(isequal(
    call(half, [6]),
    half(6)
));
{% endhighlight %}

We can also make locally scoped functions. We assign them to a variable that we can use to pass the function to another function.

{% highlight php %}
<?php
$half = function($x) { return divide($x, 2);};
assert(isequal(
    $half(3),
    call($half, [3])
));
{% endhighlight %}


## Conditionals

We'll need conditionals. In λlicious, they look like this:

{% highlight text %}
return
    condition1 ? result1 :
    (condition2 ? result2 :
    (condition3 ? result3 :
    (finalResult)));
{% endhighlight %}    

In other words, conditions are tried one after another until one of them evaluates to true. The value of the complete expression is the result that follows the first true condition. If none of the conditions evaluate to true, the finalResult is returned.

<img style="float:left;margin-right: 10px" src="/img/posts/2014-11-24-higher-order-programming/recursive_percy_small.jpg" alt="Recursive Percy">

## Loops & List Processing

Now we want to get the halves of a whole list of integers. Our test will look like this:
`assert(isequal($halves([2, 4, 6]), [1, 2, 3]));`


In λlicious, we only ever assign a value to a variable once. PHP does not constrain this: you can override the value of a variable as much as you like. Worry not: a λlicious-programmer is disciplined and strong-minded, and does not require any hand-holding from the compiler. This also means that we never use loops like foreach, while, ..., as they mutate variables. The alternative way to loop, is by using recursion.

`$halves` takes a list as an argument. We'll calculate the half of the head of that list, and then recurse to `$halves` again with the tail of the list. We cons up the calculated half with the result of the recursive call to `$halves`. Here's our first attempt:

{% highlight php %}
<?php
$halves1 = function($list) use(&$halves1) {
    return
         cons( // create a new list consisting of:
             divide(head($list), 2), // the half of the first element
             $halves1(tail($list)) // the halves of the remaining elements
         );
}
{% endhighlight %}

(The `use(&$halves1)` deserves some explanation. At this point in the code, `$halves1` is not yet defined. By closing it in our function by reference with `&`, we can call it at the point in the code where we reach `$halves1(tail($list))`. It's a little trick in PHP to recurse on locally scoped functions. Don't worry about it. When you get a `Notice: Undefined variable: foo`, just remember to add `use(&$foo)`.)

If we try to use `$halves1`, we crash with a message saying `HeadIsDefinedOnlyForNonEmptyLists`. The problem is that we are recursing, but we have not thought about how the loop stops. The λlicious-programmer accepts that mistakes happen, and draws lessons from it. From now on, whenever we recurse, we will think about the termination first.

As we are picking elements of our `$list`, it will become empty at some point. We'll need some place to store elements that we halved, so we add an accumulator called `$acc`, which starts as an empty list. Whe `$list` becomes empty, we return the accumulated values in `$acc`.

{% highlight php %}
<?php
$halves2 = function($list, $acc = []) use(&$halves2) {
    return
        isempty($list) ? $acc : // return $acc when we're done picking off items
        $halves2( // Recurse
            tail($list),
            cons( // Our new $acc will be our newly calculated half, 
                  // followed by the old $acc
                divide(head($list), 2),
                $acc
            )
        );
};
{% endhighlight %}

The assert still fails. Closer inspection learns that `$halves([2, 4, 6])` returns `[3, 2, 1]`. That makes sense: we keep consing up the halved head of the original list, to the accumulated list, so our `$acc` ends up backwards. That's easy to fix. We simply call `reverse` on `$acc` before returning it.

{% highlight php %}
<?php
$halves3 = function($list, $acc = []) use(&$halves3) {
    return
        isempty($list) ? reverse($acc) : // Reversing $acc at the last moment
        $halves3(
            tail($list),
            cons( // This is where our halved heads end up in the wrong order
                divide(head($list), 2),
                $acc
            )
        );
};
assert(isequal(
    $halves3([2, 4, 6]),
    [1, 2, 3]
));
{% endhighlight %}

`$halves3` passes our test. Recursion is great fun, and can be very rewarding. It's especially useful for making obvious jokes. If you feel you don't really get recursion, I advise you to read [this wonderful blog post](/2014/11/higher-order-programming/).

## Deduplication

`$halves3` works, but we're not quite happy with it. `divide(head($list), 2)` duplicates the logic of our `$half` function. Let's inject that function into our 4th version.

{% highlight php %}
<?php
$halves4 = function($half, $list, $acc = []) use(&$halves4) { 
    return
        isempty($list) ? reverse($acc) :
        $halves4(
            $half, // Don't forget to keep passing $half along to $halves4
            tail($list),
            cons(
                $half(head($list)), // Using our injected $half function
                $acc
            )
        );
};
assert(isequal(
        $halves4($half, [2, 4, 6]),
        [1, 2, 3]
    ));
{% endhighlight %}

`$halves4` is a higher order function: it takes another function as an argument. In our case, it applies this function to each element of `$list`. This pattern is actually quite common. So common indeed, that it has a name: map. We can simply rename `$halves4` and `$half` to be more generic.

{% highlight php %}
<?php
$map = function($function, $list, $acc = []) use(&$map) {
    return
        isempty($list) ? reverse($acc) :
        $map( // Recurse over map
            $function, // Passing $function along to the next call of $map
            tail($list),
            cons(
                $function(head($list)), // Using our injected $function
                $acc
            )
        );
};
assert(isequal(
        $map($half, [2, 4, 6]), // Map $half over the list
        [1, 2, 3]
    ));
{% endhighlight %}

(Later on we'll use the built-in global function `map` instead of our own `$map`, as it's a little smarter.)

## Filter and Reduce

`filter` and `reduce` are two other very commonly used higher order functions. They are globally defined in λlicious. The implementation is similar to that of `map`, so I'll leave them as an exercise to the λlicious student. We'll use them later in this post. Here's a skeleton to get you started:


{% highlight php %}
<?php
$filter = function($function, $list, $acc = []) use(&$filter) {
    return
        TERMINATING_CONDITION ? SOME_VALUE :
        $filter(
            $function,
            A_LIST,
            CALCULATE_THE_NEW_ACCUMULATOR
        )
        ;
};


assert(isequal(
    $filter(function($x) { return $x >= 3;}, [1, 2, 3, 4]),
    [3, 4]
));

$reduce = function($function, $list, $init) use(&$reduce) {
    return
        TERMINATING_CONDITION ? SOME_VALUE :
        $reduce(
            $function,
            A_LIST,
            CALCULATE_THE_NEW_INIT
        )
        ;
};

assert(isequal(
    $reduce(function($x, $y) { return add($x, $y);}, [1, 2, 3], 0),
    6
));

{% endhighlight %}
   

## Functions returning functions

Functions that take functions as arguments are just one type of higher order functions. We can also return functions. Let's make a function that makes `$half`.

{% highlight php %}
<?php
$halfMaker = function() {
    return function($x) { // make a new function and return it 
        return divide($x, 2);
    }; 
};
$half2 = $halfMaker();
assert(isequal(
    $half2(8),
    4
));
{% endhighlight %}

This may seem a little pointless. But this is actually a pattern as well. We've hardcoded the division by 2, but what if we want to make that number dynamic? We rename `$halfMaker` into `$divisionMaker` and take a number as an argument:

{% highlight php %}
<?php
$divisionMaker = function($y) {
    return function ($x) use ($y) { // make a closure with $y
        return divide($x, $y);
    };
};
$third = $divisionMaker(3);
assert(isequal(
    $third(9),
    3
));
{% endhighlight %}

We can make it even more generic. Right now, `division` is hardcoded into `$divisionMaker`. We can actually pass that function in as well. We rename `$divisionMaker` to `$partial`.

{% highlight php %}
<?php
$partial  = function($f, $y) {
    return function($x) use($f, $y) {
        return $f($x, $y);
    };
};
$third2 = $partial(divide, 3);
assert(isequal(
        $third2(9),
        3
    ));
{% endhighlight %}

`$partial` takes two arguments: a function `$f` and one argument `$y`. It returns a new function, which takes another argument `$x`, and return the result of `$f($x, $y)`. Partial function application is a nice example of higher order programming: you create new functions from existing functions. 

## Partial Function Application

Wouldn't it be nice if we had an elegant syntax to partially call functions? In λlicious, many functions already feature partial function application by default:

{% highlight php %}
<?php
$half3 = divide(__, 2); // returns a function $f($x){ return divide($x, 2); }
assert(isequal(
    $half3(8), 4
));
$increment = add(1, __); // returns a function $f($y){ return add(1, $y); }
assert(isequal(
    $increment(5), 6
));
{% endhighlight %}

The double underscores are placeholders: You can think of them as arguments that you don't know yet, and that you'll fill in later. Partial function application allows you to define behaviour in one place, and then process data using that behaviour, in another place. It's great way of writing compact, highly reusable, highly readable code. It might take some getting used to, but once you get it, you don't want to code without it anymore. Here's an example with `map`:

{% highlight php %}
<?php
$halves5 = map($half, __);
// elsewhere:
assert(isequal(
    $halves5([2, 4, 6]),
    [1, 2, 3]
));
{% endhighlight %}

At the point where we use `$halves5`, we don't need to know that internally, it's actually a map of `$half`.

## Composition

Sometimes you need a function that is composed of other functions. If we want to half a value and then add one, we can make a function that does the work for us:

{% highlight php %}
<?php
$halfAndIncrementMaker = function($half3, $increment) {
    return function($x) use ($half3, $increment) {
        return $increment($half3($x));
    };
};
$halfAndIncrement = $halfAndIncrementMaker($half3, $increment);
assert(isequal(
    $halfAndIncrement(10), 6
));
{% endhighlight %}

Once again, we'll turn our `$halfAndIncrementMaker` into a more generic function, called `compose`. It's another example of a higher order function: it takes functions as arguments and returns a new function.

{% highlight php %}
<?php
$compose = function($f, $g) {
    return function($x) use ($f, $g) {
        return $g($f($x));
    };
};
$halfAndIncrement = $compose($half3, $increment);
assert(isequal(
        $halfAndIncrement(10), 6
    ));
{% endhighlight %}

`$compose` creates a function that calls `$f` on `$x`, and then calls `$g` on the result of that. The built--in `compose` is a little bit smarter and accepts more than two arguments. 

## Piping

`pipe` is the same as `compose`, but it applies the functions in reverse. In my opinion, it is more natural to read. Think of pipes and filters in Linux. All together now:

{% highlight php %}
<?php
$half5 = divide(__, 2);
$greaterThanSix = gt(__, 6);
$calculate = pipe( // pipe returns a new function
    map($half5, __),  // Half all the elements of a list
    filter($greaterThanSix, __), // Keep only the elements > 6 of the new list
    reduce(add, __, 0) // Add up all the filtered elements, starting with 0
);
assert(isequal(
    $calculate([10, 20, 30]),
    25
));
{% endhighlight %}

If we call `$calculate` with a list `[10, 20, 30]`, the first placeholder is replaced with that list. The outcome is `[5, 10, 15]`. That result is fed into the `filter`, which removes the `5` from that list. Finally, `reduce` adds up `10` to `0`, and than the adds up `15` to that `10`, resulting in `25`  which is the final outcome of `$calculate`.

A great way to read that code, is to follow the pipe top the bottom. The first placeholder means "fill in a list that we don't know about yet". The second placeholder means: "pipe the result of the map to the filter". The third placeholder means: "pipe the filtered list to reduce". You can really follow the flow data through the placeholders, without being distracted by the actual data. 

We can get rid of `$half5` and `$greaterThanTen`, to make it even prettier.

{% highlight php %}
<?php
$calculate2 = pipe(
    map(divide(__, 2), __),
    filter(gt(__, 6), __),
    reduce(add, __, 0)
);
{% endhighlight %}

We've now expressed a calculation as a composition of partially applied functions. It's compact, elegant, and to the point. If you have trouble understanding what's going on, you can inject some `dump` functions along the way:

{% highlight php %}
<?php
$calculate3 = pipe(
    map(divide(__, 2), __),
    dump,
    filter(gt(__, 6), __),
    dump,
    reduce(add, __, 0)
);
{% endhighlight %}


Imagine you had to write all that in traditional procedural PHP. The beauty is that we separates the definition of how data flows through the functions, from having to deal with the actual data. From a small set of primitive functions, we can build ever more complex abstractions. And that is the point of higher order programming.


## Lambdalicious and PHP

Keep in mind that, at the time of writing, λlicious is not stable and might already have changed since you read this. Lambdalicious is not fit for any purpose. If you use it in production and everything breaks, your friends will laugh at you behind your back. I will provide a shoulder to cry on at twice my normal consultancy rates, while saying "I told you so."

The reason it can not work is that PHP is not suitable for recursion. Some languages have something called tail call optimization, which means that if the last expression is recursive, the compiler will make a jump. The call stack does not increase, and all is well.

PHP will either run out of memory (*Fatal error: Allowed memory size of X bytes exhausted*) or, if you have XDebug installed, it will stop at a 100 calls by default. (*PHP Fatal error:  Maximum function nesting level of '100' reached, aborting*). You can increase `xdebug.max_nesting_level` in php.ini. HHVM overflows (*Fatal error: Stack overflow*). I promise to send some fine Belgian beer or chocolates to whoever fixes this in PHP or HHVM.

## Links

- [Lambdalicious on GitHub](https://github.com/mathiasverraes/lambdalicious)
- [The full code for this blog post](https://gist.github.com/mathiasverraes/4b76822c6be565a092f7)
