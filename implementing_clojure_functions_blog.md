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

#####Sequences

`reduce` is the backbone of many of the Clojure *sequence* functions. Before we start implementing it, what exactly is a sequence?

Alex Miller wrote a very clear [introduction to sequences](http://insideclojure.org/2015/01/02/sequences/). They are essentially *the key abstraction that connects immutable persistent collections and the sequence library*. The key sequence abstraction functions are `first`, `rest`, and `cons`. Calling `sequence` on a seqable - a collection, or other thing, that can produce a sequence - returns a sequence. Calling `seq` has the same effect, except that empty collections will return `nil`, rather than an empty sequence. Seqable collections include lists, vectors, sets, and maps. The sequence abstraction enables sequence functions - those that implement the sequence abstraction, including `reduce`, `count`, `filter`, `map`, and `pmap` - to handle any seqable data structure. This means that we can use the same function with any seqable collection. The sequence functions implicitly convert the input collection to a sequence.  

Now we know what a sequence is, we can look at `reduce`, and implement our own version, `my-reduce`.

#####Implementing my-reduce

From the [Clojure docs](https://clojuredocs.org/clojure.core/reduce):

`(reduce f coll)``(reduce f val coll)`
*f should be a function of 2 arguments. If val is not supplied, returns the result of applying f to the first 2 items in coll, then applying f to that result and the 3rd item, etc. If coll contains no items, f must accept no arguments as well, and reduce returns the result of calling f with no arguments.  If coll has only 1 item, it is returned and f is not called.  If val is supplied, returns the result of applying f to val and the first item in coll, then applying f to that result and the 2nd item, etc. If coll contains no items, returns val and f is not called.*

There is a lot to do, so let's build it up slowly in a Test Driven Development (TDD) way. We can start with the case of two arguments and where `coll` contains no items, so should return `val`. A first test could use the addition function, an initial value of 1, and an empty list.

    (it "result 1 for addition function, with val of 1 and empty collection"
		(should= 1 (my-reduce + 1 '()))))

We can get this test to pass by defining `my-reduce` with three parameters, and returning `val`.

    (defn my-reduce [f val coll]
	    val)

With the test passing, we can add a second test where we have a single item in the collection: `(should= 2 (my-reduce + 1 '(1)))`. This fails, so let's fix it. In order to get this test to pass, we need to apply the function `+` to `val` and the first element of `coll`: `(f val (first coll))`. We can utilize our knowledge that calling `seq` on an empty seqable returns `nil`, which is falsy, whereas calling `seq` on a seqable with at least one item returns a sequence, which is truthy. This means we can direct the function to either return `val`, or the product of applying `f` to `val` and `(first coll)`: 

	(defn my-reduce [f val coll]
		(if (seq coll)
			(f val (first coll))
			val))

What about when we have more than one item in the collection? A next test could be: `(should= 6 (my-reduce + 1 [2 3]))` 

To get this to pass we need to apply `f` to the result of applying `f` to `val` and `(first coll)`, and the second item. We can do this by recursively calling `my-reduce` with `f`, `(f val (first coll))`, and `(rest coll)`, and continuing until the collection is empty (when `(seq coll)` returns `nil`), at which point we will return `val`.

	(defn my-reduce [f val coll]
		(if (seq coll)
			(my-reduce f (f val (first coll)) (rest coll))
			val))   

We can add more tests using other functions and Clojure seqables, including vectors and sets, and they still pass with our `my-reduce` function.

All good, so let's move to the situation where `val` is not supplied. In this case it should return the result of applying `f` to the first 2 items in `coll`, then apply `f` to that result and the 3rd item, etc.

We can write a new test, `(should= 1 (my-reduce + [1]))`, which fails becuase the wrong number of arguments are passed to `my-reduce`. To fix this, we can add an arity taking in just `f` and `coll`, which then calls `my-reduce`, passing in `(first coll)` to `val`, and `(rest coll)` to `coll`. Our `my-reduce` function already works when there is an empty `coll`, so this should work as long as we have at least one item in our collection.

	(defn my-reduce 
		([f coll]
		 	(my-reduce f (first coll) (rest coll)))
		([f val coll]
			(if (seq coll)
				(my-reduce f (f val (first coll)) (rest coll))
				val)))

This works as we wanted, and the test passes. We can add some additional tests to confirm that `my-reduce` handles multiple items in a collection in the way that we expect.

Finally, if we pass in no `val` and an empty `coll`, we need to return the result of evaluating `f` with no arguments. Evaluating `+` with no arguments returns 0, so let's add a test for that: `(should= 0 (my-reduce + []))`.

We need `my-reduce`, when passed `f` and `coll`, to evaluate `f` if `coll` is empty:

	(defn my-reduce 
		([f coll]
			(if (seq coll)
		 		(my-reduce f (first coll) (rest coll))
		 		(f)))
		([f val coll]
			(if (seq coll)
				(my-reduce f (f val (first coll)) (rest coll))
				val)))

#####Property-based testing

How many unit tests are enough to have confidence that the code is working as intended? Are there ever enough? We could keep adding more and more, but this doesn't gurantee that we'll cover all the edge cases where bugs like to lie, and every additional test is more code that needs maintaining.

Given `reduce` can take such a large range of inputs - different functions, different collection types, different value types - it is impossible to write unit tests to cover the entire scope of possibilities. It is also easy to miss potential edge cases. Property-based tests give us an alternative approach to testing, that makes it easier to automate testing across a wider range of the possible inputs.

Rather than asserting that specific inputs to your code should result in a specific output, as with unit tests, property-based tests make statements about the expected behaviour of the code that should hold true for the entire domain of possible inputs. These statements are then verified for many different (pseudo)randomly generated inputs.

Clojure has [test.check](https://github.com/clojure/test.check), written by [Reid Draper](https://twitter.com/reiddraper), to do the heavy lifting for us.

Each property-based test includes the number of examples to be tested and the universal quantification, `prop/for-all`, where for all valid inputs to the function this property should hold true. `prop/for-all` takes two arguments. The first is a binding of a generator to a variable name, in a similar way to a `let` clause. The generators are peudorandom, and start with the simplist cases so that failures from common edge-cases can be found quickly. The second argument is the behaviour statement, where an assertion that a boolean condition will return true is made.

To test that our `my-reduce` function is valid, we simply need to assert that it returns the same as `reduce`, when passed any seqable collection, containing any Clojure value, either with or without an initial value. We can create the generator `gen/any` for the initial value to be passed in. To generate the collections we can use combinator generators, so that each value can be a different seqable collection, containing any Clojure values.  

It is possible to run the test with thousands of inputs, and in this way vastly increase our confidence that our implementation behaves the same as the core Clojure one. For more detail on property-based testing of these Clojure function check out [Property-Based Testing in Clojure](http://jonathangraham.github.io/2016/01/07/property_based_testing_clojure_functions/).


