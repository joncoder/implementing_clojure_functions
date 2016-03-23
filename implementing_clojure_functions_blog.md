#Jump into your code with confidence
##*Understanding core Clojure Functions*
[Jonathan Graham](https://twitter.com/graham_jp), Mar 23rd, 2016

##Before you jump...

Imagine you are hurtling toward the ground at 122 mph (the [approximate terminal velocity of a skydiver](https://en.wikipedia.org/wiki/Terminal_velocity)). You feel both exhilerated and in control. You understand your equipment, and have full confidence in your ability to land safely, and on target.

Fast-forward a few days. You've been convinced to the top of a cliff to take your first [BASE jump](https://en.wikipedia.org/wiki/BASE_jumping). How confident are you now in your parachute, and your ability to safely use it in this new environment? Will you pass on the opportunity for the adrenaline-high of a lifetime, because you don't understand the full capabilities of the tools you have? Or worse, will you jump without knowing the limitations of your equipment, potentially resulting in disasterous consequences?

We face similar, albeit normally less directly life-altering, dilemmas on a regular basis as we code.

Every programming language has a bunch of core functions already defined for us. This is great - it means we can quickly get on building what we want, without having to worry about writing everything from scratch.

This power and ease of use does come at a cost, though. Do we really understand how the functions are working, and if not, how do we really know the power of the magic that we are wielding? Are we in danger of breaking our code through the misuse of functionality that we thought, but didn't truly, understand? And are we utilizing the functions to their full effect, or missing out on some of the benefits of the language?

##Implementing models of core language features

Whatever language you are working in, writing your own implementations of some of the core features is a great way to gain the fundamental understanding that you need to be fully in control of your code. It is especially valuable to do as an exercise when learning a new language, although is worth doing at any time. The implementations do not necessarily need to replicate all aspects of the core function, and can just model certain behaviours.  

In this blog, we will be jumping into [Clojure](https://clojure.org/). We will implement versions of `reduce`, `count`, `filter`, `map`, and `pmap`. In doing so, we will explore, amongst other things, recursion, lazy sequences, and futures. Along the way, we will also look at different testing paradigms.

The goal for the models of the core functions that we implement is to generate the same output as those produced by the core language, given any valid input. For the purpose of this blog, we will not be considering the processing efficiency of the functions that we implement.

##Writing our Clojure functions

###Setup

The source code for the functions and tests that are described below is on [github](https://github.com/jonathangraham/clojure_functions). They are structured in a [leiningen](http://leiningen.org/) project. The unit-test framework used is [speclj](http://speclj.com) (run with `lein spec`), and [test.check](https://github.com/clojure/test.check) is used for the property-based tests (run with `lein test`).

###Reduce

`reduce` is the backbone of many of the Clojure *sequence* functions. Before we start implementing it, what exactly is a sequence?

Alex Miller wrote a very clear [introduction to sequences](http://insideclojure.org/2015/01/02/sequences/). They are essentially *the key abstraction that connects immutable persistent collections and the sequence library*. The key sequence abstraction functions are `first`, `rest`, and `cons`. Calling `sequence` on a seqable - a collection, or other thing, that can produce a sequence - returns a sequence. Calling `seq` has the same effect, except that empty collections will return `nil`, rather than an empty sequence. Seqable collections include lists, vectors, sets, and maps. The sequence abstraction enables sequence functions - those that implement the sequence abstraction, including `reduce`, `count`, `filter`, `map`, and `pmap` - to handle any seqable data structure. This means that we can use the same `reduce` function with any seqable collection. 

Now we now what a sequence is, we can look at `reduce`.

From the [Clojure docs](https://clojuredocs.org/clojure.core/reduce):

`(reduce f coll)``(reduce f val coll)`
*f should be a function of 2 arguments. If val is not supplied, returns the result of applying f to the first 2 items in coll, then applying f to that result and the 3rd item, etc. If coll contains no items, f must accept no arguments as well, and reduce returns the result of calling f with no arguments.  If coll has only 1 item, it is returned and f is not called.  If val is supplied, returns the result of applying f to val and the first item in coll, then applying f to that result and the 2nd item, etc. If coll contains no items, returns val and f is not called.*

There is a lot to do, so let's build it up slowly in a Test Driven Development (TDD) way.

