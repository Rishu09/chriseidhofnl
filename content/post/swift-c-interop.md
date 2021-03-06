---
aliases:
  - /posts/swift-c-interop.html
date: 2015-07-31
title: Swift and C functions
headline: A short look at wrapping qsort
---


One of Swift's greatest strengths is the low friction when interoperating with C and Objective-C. Swift can automatically bridge Objective-C types, and can even bridge with many C types. This allows us to use existing libraries, and provide a nice interface on top. In our book [Advanced Swift](http://www.objc.io/books/advanced-swift/), we will write a wrapper around [cmark](https://github.com/jgm/cmark), and a wrapper around [libuv](http://libuv.org). This post is an excerpt from a first draft of one of the C interop chapters.

## Function Pointers

Rather than wrapping a larger library, let's have a look at wrapping the standard C `qsort` function. The type as it imported in Darwin is given below (we've added the argument names back in, unfortunately they get lost during the import process):

     func qsort(base:   UnsafeMutablePointer<Void>,
                nel:    Int,
                width:  Int,
                compar: ((UnsafePointer<Void>, UnsafePointer<Void>) -> Int32)!)

The manual (`man qsort`) describes how to use it:

> The `qsort()` and `heapsort()` functions sort an array of `nel` objects, the initial member of which is pointed to by `base`.  The size of each object is specified by `width`.
>
>  The contents of the array base are sorted in ascending order according to a comparison function pointed to by `compar`, which requires two arguments pointing to the objects being compared.

And here is a wrapper function that uses `qsort` to sort an array of Swift strings:

    func qsortWrapper(inout array: [String]) {
        qsort(&array, array.count, strideof(String)) { a, b in
            let l: String = UnsafePointer(a).memory
            let r: String = UnsafePointer(b).memory
            return r > l ? -1 : (r == l ? 0 : 1)
        }
    }

Let's look at each of the arguments being passed to `qsort`:

 - A pointer to the base of the array. Swift arrays automatically convert to C-style base pointers when you pass them into a function that takes an `UnsafePointer`. We use the `&` prefix because it is an `UnsafeMutablePointer` (in the C declaration, a `void *base`). If the function didn't need to mutate its input so declared in C as `const void *base`, the `&` wouldn't be needed. This matches the difference with `inout` arguments in Swift functions.
 - The number of elements. This one is easy, just the count property of the array.
 - To get the width of each element, we use `strideof` – _not_ `sizeof`.  In Swift, `sizeof` returns the true size of a type, but when locating elements in memory, platform alignment needs might mean the gap between each element (the "stride") could be the size _plus_ some padding.  In case of strings they will be the same, but when dealing with some types (specific structs and enums), `sizeof` doesn't include the memory padding, and `strideof` does. 
  - For the `compar`, we can just pass in a trailing closure expression (as long as it doesn't capture any variables).  The `compar` function accepts two void pointers. A C void pointer can be a pointer to anything: the first thing to do is cast it to a pointer to the actual type you (hope) it is. In the case of `qsort`, the they will be to elements in the array, pointers to two Swift strings. Finally, the function needs to return an `Int32`: if the first element is greater than the second, it should be larger than 0, if they're equal, 0, and if it's small a number, less than zero.

It's easy enough to create another wrapper that works for a different type; we can copy-paste the code, change `String` to a different type, and then we're done. However, when we try to make the code generic, we hit the limit of C function pointers. When writing the function below, the Swift compiler segfaulted on the code below. Even if it wouldn't segfault, the code is still impossible: it captures a variable from outside the closure. Specifically, it captures the comparison and equality operators: they are different for each A. There is nothing we can do about this: we just hit a hard limitation of the way C works.

    func qsortWrapper<A: Comparable>(inout array: [A]) {
        qsort(&array, array.count, strideof(A)) { a, b in
            let l: A = UnsafePointer(a).memory
            let r: A = UnsafePointer(b).memory
            return r > l ? -1 : (r == l ? 0 : 1)
        }
    }

However, in practice this is a problem for many C programmers as well. On OS X, there is a variant of `qsort` called `qsort_b`, which takes a block as the last parameter, instead of a function pointer. If we replace the `qsort` by `qsort_b` in the code above, our code will compile and run fine.

However, on many platforms `qsort_b` might not be available. Specifically, in other APIs, there might not be a block-based alternative. Oftentimes, there is a different solution. Many C functions and datatypes take an extra unsafe void pointer as a parameter, which is user-defined context that can be used. In C, you can store the address of any type of variable in that context (it is untyped). In the case of `qsort`, there is a variant called `qsort_r` which does exactly this. Comparing the type with `qsort`, we can see that just before the block, it takes an extra parameter `thunk`, an unsafe mutable void pointer. This parameter also gets passed in to the comparison function pointer.

    func qsort_r(base:   UnsafeMutablePointer<Void>,
                 nel:    Int, 
                 width:  Int, 
                 thunk:  UnsafeMutablePointer<Void>,
                 compar: ((UnsafeMutablePointer<Void>, UnsafePointer<Void>, UnsafePointer<Void>) -> Int32)!)


We can use this mutable void pointer to pass arbitrary context to `qsort`, and use it inside the block. In order to do that, we first need to be able to convert any data into an unsafe mutable pointer. First, when we pass data to C, we want to make sure the data is retained (if it is a reference type). To do that, we can create an unmanaged reference:

    let unmanaged = Unmanaged.passRetained(Box(data))

Next, we want to create an unsafe mutable void pointer out of this. We can take the unmanaged value, make it an opaque value and initialize an unsafe mutable pointer with it. This is a couple of steps, and hopefully in the future, the Swift Standard Library will provide a shorter way to do this:

    let pointer: UnsafeMutablePointer<Void> = UnsafeMutablePointer(unmanaged.toOpaque())

We can pass that pointer around. Finally, once we are done, we want to make sure to release the reference again:

    unmanaged.release()

To make sure we don't make any mistakes with retains and releases, we can wrap this up in a function. This function takes a value of any type, and a callback function which performs a computation with the void pointer. By using a callback function, we can make sure to release our object after we are done. The moment the callback returns, the value is released. Therefore, this is only safe to use when the void pointer is not stored somewhere, or used asynchronously.

    func withVoidPointer<A>(x: A, @noescape f: UnsafeMutablePointer<Void> -> ()) {
        let unmanaged = Unmanaged.passRetained(Box(x))
        f(UnsafeMutablePointer(unmanaged.toOpaque()))
        unmanaged.release()
    ```

As a companion function, we can also write an `unsafeFromVoidPointer`, which removes all the wrapping we added in the previous function. It uses `takeUnretainedValue` to make sure to not change the retain count when using the value.

    func unsafeFromVoidPointer<A>(thunk: UnsafeMutablePointer<Void>) -> A {
        return Unmanaged<Box<A>>.fromOpaque(COpaquePointer(thunk)).takeUnretainedValue().unbox
    }

Before we use the two functions, we will introduce a new function on `Comparable`, which will have the form of a C comparison callback function:

    extension Comparable {
        static func compare(a: UnsafePointer<Void>, _ b: UnsafePointer<Void>) -> Int32 {
            let l: Self = UnsafePointer(a).memory
            let r: Self = UnsafePointer(b).memory
            return r > l ? -1 : (r == l ? 0 : 1)
        }
    }

We have all the pieces together to finally wrap `qsort_r`. Instead of a block, we will use a separate function `cmp` which we define below. By using `withVoidPointer`, we can convert the `A.compare` function into a void pointer, which we pass as the fourth argument to qsort_r. The rest of the code remains unchanged.

    func qsortRWrapper<A: Comparable>(inout array: [A]) {
        withVoidPointer(A.compare) { p in
            qsort_r(&array, array.count, strideof(A), p, cmp)
        }
    }

Now, our `cmp` function needs to take the first void pointer, convert it back into a comparison function, and call it with the two other arguments.

    typealias Compare = (UnsafePointer<Void>, 
                         UnsafePointer<Void>) -> Int32
    
    private func cmp(thunk: UnsafeMutablePointer<Void>,
                     a: UnsafePointer<Void>,
                     b: UnsafePointer<Void>) -> Int32 {
        let f: Compare = unsafeFromVoidPointer(thunk)
        return f(a, b)
    }

It might seem like this is a lot of work to use a sorting algorithm from the C Standard Library, after all, the default Swift sorting algorithm is much more optimized in many cases. However, what we have done is reusable in many cases. Not only are we now able to succesfully use C's builtin functions in Swift, we can now also use them in a type-safe and generic way. There are many other interesting libraries and functions out there, written in C, which we can wrap using the same technique.


If you liked this, consider buying our book [Advanced Swift](http://www.objc.io/books/advanced-swift/), where we'll go into much more detail on how to work with C libraries (among many other things). Thanks to my co-author [Airspeed Velocity](http://airspeedvelocity.net) for reading through and making edits to this text.

