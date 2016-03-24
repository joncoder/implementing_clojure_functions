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

###REDUCE

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

#####What we learned about reduce

If we previously thought that we needed to pass an initial value as well as a collection to `reduce`, we now have extended the power and utility we wield over this fundamental function. Also, even if we have a function that requires two arguments, we know that we can successfully pass this to `reduce` with only the collection. We can demonstrate this with `add`: `(defn add [x y] (+ x y))`. If we just pass a single argument to `add` we will get an argument error. However, calling `(reduce add [1])` will return 1, because the first item of the collection will be defined as `val`, and given the rest of the collection is empty, `reduce` will just return `val` and the function is not called.  

We also now know some of the limitations of `reduce`. If we are going to use `reduce` and there is a possibility that it could get called with no initial value and an empty collection, then the function passed to `reduce` will be evaluated. If the function, like `-`, won't accept zero arguments then our program will error. To avoid this scenario we need to make sure that we write our functions so that they can be evaluated with no arguments. Alternatively, we now know that we can write our own `reduce` function, and so could modify it so that it defines differently what to do when passed just an empty collection.

###COUNT

Now we have written our implementation of `reduce`, let's move on to `count`:

`(count coll)`
*Returns the number of items in the collection. (count nil) returns 0.  Also works on strings, arrays, and Java Collections and Maps.*

We can start with a test where we pass an empty collection, `(should= 0 (my-count '())`, and get this to pass just by returning 0. This would also allow the test with `nil` to pass. If we add a test where we pass a collection containing a single item, we need to modify our function, as this time it needs to return 1. Again, we can use `(seq coll)`, which will return true if there are one or more items in the collection.

	(defn my-count [coll]
		(if (seq coll)
			1
			0))

Great, so now let's get this working for a case where we have more than one item in the collection: `(should= 2 (my-count '(1 2)))`. We can do this recursively by starting `result` at 0 and adding 1 every time we move a position along the collection until the collection is empty, at which point we simply return `result`:

	(defn my-count [coll]
		(loop [coll coll result 0]
			(if (seq coll)
				(recur (rest coll) (inc result))
				result)))

This passes, and we can add more tests to confirm that the function is working as we expect. Now we have a passing test suite, we can refactor. We have already written `my-reduce`, so let's see if we can use this to define `my-count`. We would want to set an initial `val` to 0 so that `my-reduce` returns 0 for an empty collection. If the collection is not empty, we need a function that will increment the count result each time that the function is called. The function will take two arguments, the count result which is initially 0 and the collection, but it is only concerned with the result, which it will increment by 1 every time that it is called. Since the function passed to `my-reduce` doesn't care about the collection, we can use an `_` rather than name it. If we put all this together then we get:

	(defn my-count [coll]
		(my-reduce (fn [result _](inc result)) 0 coll))

This successfully passes the test suite, and now looks much cleaner than the code we had before. We can also write a property-based test to confirm that the function returns the same results as `count` across generated inputs. 

A disadvantage of the refactor, though, is that we have lost the recur special operator that we used before, which does constant-space recursive looping by rebinding and jumping to the nearest enclosing loop or function frame. With this gone we consume stack space as we make our recursive calls, and so risk stack overflow.

###FILTER

`(filter pred)` `(filter pred coll)`
*Returns a lazy sequence of the items in coll for which (pred item) returns true. pred must be free of side-effects. Returns a transducer when no collection is provided.*

We can build up a recursive function in a similar way that we did for `my-count`. If there are no items in our `coll` then intuitively we might expect that we could just return the `coll`. However, `filter` needs to return a lazy sequence.

A lazy sequence can be created by the macro `lazy-seq`. This takes a body of expressions that returns an ISeq or nil, and yields a Seqable object that will invoke the body only the first time seq
is called, and will cache the result and return it on all subsequent seq calls.

We can write a first test, where we use the predicate `zero?` (will return true if the item is 0) and pass an empty vector. The output should be of class clojure.lang.LazySeq (`(should= clojure.lang.LazySeq (class (my-filter zero? [])))`), and have zero items (`(should= 0 (my-count (my-filter zero? [])))`).

We can get these tests to pass just by returning `(lazy-seq coll)`, which will create the lazy sequence.

Let's now add a test that is going to filter a collection that contains items: `(should= '(0 2 4 6 8) (my-filter even? (range 10)))`.

To demonstrate how lazy sequences work, let's first build a function that doesn't implement them correctly. 

To handle collections containing items we can start our recursive call by setting `input` to the `coll` that we pass in and `result` to `[]`. We could have `result` initialised with any collection, or indeed an empty collection of the same type as the input, but by using a vector we can use `conj` to add items to the end, and not worry that different data sets use `conj` in different ways. 

We will then consider each item in turn, recursing with `(rest input)`, until the collection is empty, at which point we want to return the `result` as a lazy sequence. So, what do we do with each item? We want to check if `pred` on the item returns true, and if it does `conj` the item to our `result` and recurse this, and if not just recurse the unaltered `result`. 

	(defn my-filter [pred coll]
		(loop [input coll result []]
			(if (seq input)
				(recur 	(rest input) 
					(if (pred (first input))
						(conj result (first input)) 
						result))
				(lazy-seq result))))

This passes, and we can add extra tests, but we are just converting the `result` to a lazy sequence at the end. What we want is for the computation to be lazy.

We could move `lazy-seq` to the front of the recur loop, but we would still not be behaving lazily. Every time we loop through we evaluate if the predicate is true, and we update our `result` accordingly. If we were lazy we would not evaluate until we reached the tail of the recursion.

In terms of laziness, there is a difference between `conj` and `cons`. The behaviour of `conj` depends on the collection type, so will need to be realised immediately. However, `cons` adds an item to the start of a collection, with everything else coming after, and so can be evaluated lazily. 

Let's re-write our `my-filter` function so that it is lazy. We can start with `lazy-seq` and pass it a `when` block, using the predicate `(seq coll)`, as it's body of expression. As before, the `when` block will exit with an empty collection. If we do have a sequence we want to apply the filter predicate to the first item of the collection. If the predicate returns true we can `cons` the `(first coll)` onto the filtered collection to come, `(my-filter pred (rest coll))`. If the predicate returns false then we simply call `my-filter` with the `(rest coll)`. Nothing will get evaluated until we reach the tail of the recursion, when the collection is empty. If we put this altogether we get:

	(defn my-filter [pred coll]
		(lazy-seq 
			(when (seq coll)
				(if (pred (first coll))
					(cons (first coll) (my-filter pred (rest coll))) 
					(my-filter pred (rest coll))))))

and our test suite still passes. We are now evaluating lazily, and we have our basic implementation of `filter`. For the purposes of this blog we will not cover the scenario where no collection is provided to filter, which would return a transducer.

So, what have we learned from writing our own version? Well, `filter` is evaluated lazily, which can give us efficiency benefits, but it also means that the collection type we return with our filtered data set is different from the input type. If we want to continue processing with the same collection type that we had then we will have to pass the output from filter `into` the required collection type, e.g. `(into [] (my-filter even? [0 1 2 3 4 5]))` will return `[0 2 4]`.

###MAP

`(map f)` `(map f coll)` `(map f c1 c2)` `(map f c1 c2 c3)` `(map f c1 c2 c3 & colls)`
*Returns a lazy sequence consisting of the result of applying f to the set of first items of each coll, followed by applying f to the set of second items in each coll, until any one of the colls is exhausted.  Any remaining items in other colls are ignored. Function f should accept number-of-colls arguments. Returns a transducer when no collection is provided.*

Again, for the purpose of this blog post we are not going to consider the situation where no collection is passed. 

We can use the same function structure as we had in `my-filter`, but in the `when` block we simply need to `cons` the result from applying the function `f` to the first item to everything that will follow. This will then all evaluate when we reach the tail position of the recursion, which is when we have an empty collection.

	(defn my-map [f coll]
			(lazy-seq (when (seq coll)
					(cons (f (first coll)) (my-map f (rest coll))))))

With passing tests using single collections we can move to consider the situation where two collections are passed to `my-map`. We need to apply the function to the first items of each collection, and then to the second, etc, and continue whilst both collections are non-empty. For example, `(should= '(4 6) (my-map + [1 2] [3 4]))`.

We can extend `my-map` to also take `[f c1 c2]`, and we want to continue `when` both `c1` and `c2` are sequences. We need to apply `f` to the first item of both `c1` and `c2` and then recur with the rest of both collections.

	(defn my-map 
		([f coll]
			(lazy-seq (when (seq coll)
					(cons (f (first coll)) (my-map f (rest coll))))))
		([f c1 c2]
			(lazy-seq (when (and (seq c1) (seq c2))
					(cons (f (first c1) (first c2)) (my-map f (rest c1) (rest c2)))))))

This works! So, how about when we have more than two collections: `(should= '(9 12) (my-map + '(1 2) '(3 4) '(5 6)))`.

Now we can again extend `my-map`, and this time take in any number of arguments `[f c1 c2 & more]`. To handle an unknown number of arguments we can start a `recur` loop, setting `c1` to the first collection passed in, `c2` to the second collection, and `r` to the rest. We will then `recur`, passing `(my-map f c1 c2)` back in as `c1`, `(first r)` as `c2` and `(rest r)` as `r`, and continue this until we only have two collections left. At this point `(seq r)` will return nil and we can escape from our recursive loop. We can then evaluate the final result by calling `(my-map f c1 c2)`.

	(defn my-map 
		([f coll]
			(lazy-seq (when (seq coll)
					(cons (f (first coll)) (my-map f (rest coll))))))
		([f c1 c2]
			(lazy-seq (when (and (seq c1) (seq c2))
					(cons (f (first c1) (first c2)) (my-map f (rest c1) (rest c2))))))
		([f c1 c2 & more]
			(loop [c1 c1 c2 c2 r more]
				(if (empty? r)
					(my-map f c1 c2)
					(recur (my-map f c1 c2) (first r) (rest r))))))

This now passes our test suite, but what about non-commutative functions? 

`(should= '([:a :d :g] [:b :e :h] [:c :f :i]) (apply my-map vector [[:a :b :c][:d :e :f][:g :h :i]]))`

This test fails, because we get a lazy sequence of vectors by calling `(my-map f c1 c2)`, and this then becomes `c1` for the next iteration, resulting in a multi-dimensional vector. So, how do we solve this?

What we want is to convert the input collections to a single sequence, where the first item is a collection of all the first elements, the second item is a collection of all the second elements, etc. If we have this collection of reordered collections we can just map the result of applying the function to each collection in turn. We can create a single collection by adding the first collection, `c1`, to the other collections, `colls`. Let's then pass this concatenated collection to a function named reorder. 

We need `reorder` to take the first item from each collection and map it to a new collection, and add this to all the other reordered collections to come until one of the collections is empty. Since the two functions, `my-map` and `reorder`, are mutually recursive, we need to delare them. If we put this all together we get:

	(declare my-map)

	(defn reorder [c]
	        (when (every? seq c)
	           	(cons (my-map first c) (reorder (my-map rest c)))))

	(defn my-map 
		([f coll]
			(lazy-seq (when (seq coll)
					(cons (f (first coll)) (my-map f (rest coll))))))
		([f c1 & colls]
	     		(my-map #(apply f %) (reorder (cons c1 colls)))))

We now have much cleaner code, and it works for non-commutative functions!

###PMAP

Finally we are going to look at `pmap`, which is *like map, except f is applied in parallel*. This function is *only useful for computationally intensive functions where the time of f dominates the coordination overhead*.

We need `my-pmap` to return the same result as `map`, so we can write a first test: `(should= (map inc [1 2]) (my-pmap inc [1 2]))`. We can get this test to pass by writing the function to just call `my-map`.

	(defn my-pmap
		([f coll]
			(my-map f coll)))

To mimic a computationally intense function, we can write a long-running function by inserting a sleep (`Thread/sleep 1000` will cause the thread to block for 1000 ms), and then adding `10` to the input:

	(defn long-running-job [n]
	    (Thread/sleep 1000)
	    (+ n 10))

We can test to check that the long-running-function takes more that 1 s to run for each number (the 1 s sleep time plus some execution time) by using the system time, and checking it before and after calling the function:

		(should (< 1.0 (let [st (System/nanoTime)]
					(long-running-job 1)
					(/ (- (System/nanoTime) st) 1e9))))

If we try to do the same thing for `(map long-running-fuction [1 2 3 4])` the test will fail. We might expect this function to take a little over 4 s to run (the `long-running-function` is called four times by `map`), but it actually completes in a fraction of a second. Why? Remember that `map` is lazy, and the time we are measuring is the time to make a new lazy-seq, not the time to actually execute it. Let's write a test where we call a function named `test-time` and pass it a map type, function and collection: `(should (< 4.0 (test-time map long-running-job [1 2 3 4])))`.

We can write `test-time`, and measure the time to evaluate `realize-lazy-seq`. `realize-lazy-seq` will recursively loop through the lazy sequence after each part is evaluated. 

	(defn realize-lazy-seq [map-type f c]
		(loop [res (map-type f c)]
	    		(when res (recur (next res)))))

	(defn test-time [map-type f c]
		(let [st (System/nanoTime)]
			(realize-lazy-seq map-type f c)
				(/ (- (System/nanoTime) st) 1e9)))

Now when we run our tests they pass, so let's put in a new test where we will `test-time` for `my-pmap`, using the same function and collection as before. If our implementation of pmap works then it should complete in less than 4 s. Indeed, if it completes each function in parallel then it should take just over 1 s. `(should (> 4.0 (test-time my-pmap long-running-job [1 2 3 4])))`

This fails, because currently `my-pmap` is just calling `my-map`, so it will apply `f` to each item in the collection in turn. What we want is for `f` to be applied in parallel, and we can achieve this using Clojure [futures](https://clojuredocs.org/clojure.core/future). If we invoke `future` every time we call `f` on each item, they will be evaluated in an available thread from the thread pool and the results cached until we call them with `deref`, and so in this way we can utilise multiple threads.

We can set a local variable name `results` to be `(my-map #(future (f %)) coll)`. This will generate a lazy sequence of future results, where `f` has been applied to each item in `coll` as a future. We might then imagine that we could map the futures to a new lazy sequence by applying the function `deref` to `results` to return the evaluations.

	(defn my-pmap
		([f coll]
			(let [results (my-map #(future (f %)) coll)]
					(my-map deref results))))

However, when we run our tests the last test fails. We can modify the test by wrapping the function that we are calling in `time`, and this will print the execution time: `(should (> 1.1 (time (test-time my-pmap long-running-job [1 2 3 4]))))`

Now when we run the test it tells us that it takes just over 4000 ms to execute - the same time that it takes `map`. So, what is wrong? Again, it is all down to lazy sequences. When we set `results` we are just generating the lazy sequence, and not evaluating it. We only evaluate `results` when we call `deref`, so each future is generated and then immediately dereferenced, with the thread blocked until the result is available. In this way, we generate the result of applying the function to each item in the array in sequence, and not in parallel.

To make the function work in parallel we need to generate all of the futures before we start to dereference. To do this we can make use of the fact that when we use `conj` the resut will be realised immediately. We can recursively iterate through the collection, applying `f` as a future to each item and `conj`ing this to an accumulator, `acc`, initialised as an empty vector. In this manner we initiate each future as we iterate through. We can then return `acc` as the escape from the recursion after we have iterated through all items. Now we have a collection of futures, which we can bind to `results`, and so we can map this collection by applying `deref` to return the futures.

	(defn my-pmap
		([f coll]
			(let [results
				(loop [remaining coll acc []]
					(if (seq remaining)
						(recur (rest remaining) (conj acc (future (f (first remaining)))))
						acc))]
				(my-map deref results))))


Our tests now pass! Indeed, our implementation `my-pmap` completes the test in just over 1 s, compared to 4 s for `map`.

Does what we are doing in `results` look familiar? We are taking an input collection, applying a function to each element in turn, and returning a single item - a vector. We know that we can perform this type of data manipulation with `my-reduce`, so let's refactor. Our initial `val` is the empty vector, and we want to `conj` into `val` the `future` of applying `f` to each element of the `coll`.

	(defn my-pmap [f coll]
			(let [results (my-reduce #(conj %1 (future (f %2))) [] coll)]
				(my-map deref results))) 

Our refactored code using `my-reduce` looks cleaner, and the tests still pass. We just need to be aware that removing the recur special operator exposes us to potential stack overflows.

Let's now make sure that `my-pmap` can work with multiple collections. We can write tests that will map a function over two collections and check that they return the correct result and in a time that confirms the functions were applied in parallel.

Using the same approach that we did for `my-map`, we can convert our collections into a single sequence of reordered collections, and reuse the `reorder` function.

	(defn my-pmap
		([f coll]
			(let [results (my-reduce #(conj %1 (future (f %2))) [] coll)]
				(my-map deref results)))
		([f c1 & colls]
	    		(my-pmap #(apply f %) (reorder (cons c1 colls)))))

This works, and as with all the previous functions we can write a property-based test to confirm that the function behaves as the original.







