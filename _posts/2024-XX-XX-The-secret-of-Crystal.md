---
title: "The Secret of Crystal"
author: beta-ziliani
summary: "TBD"
categories: community
tags: [language]
---

In this post I place Crystal in the map of programming languages. If you haven't yet dirtied your hands with the language, you need to know that the language sits in a pretty much uncharted part of the territory of languages. Here I explain what's so special about it, and what is its secret weapon.

In short: Crystal is _statically typed_ and compiled like Java and C++, but free of bureaucracy, similar like Ruby or Python. In order to achieve that:

> NOTE Crystal trades modularity for expressivity.

## Crystal: an _unbureaucratic_ language

Crystal lets you write code as if you were in a dynamic language, like Ruby or Python. In particular, it has two capabilities that are rarely found in a static language: _duck typing_ and _monkey patching_. Let's explain these with examples.

### Duck typing

Look at the following valid Ruby and Crystal code:

```cr
def adder(x, y)
  x + y
end

adder 1, 2 # => 3
adder "hi", "world" # => hiworld
```

It's uses duck typing:

> If it walks like duck, and talks like a duck, it's a duck

Applying the analogy, our code is valid because, _if it can be added, then it's added_. And since both numbers and strings can be operated with `+`, then they indeed are!

This might not be surprising if you come from a dynamic language. But if you come from a language like Java or C++, it certainly is. In such languages, you need to specify in some way that `x` and `y` can, in fact, be added. For instance, via an _Addable_ interface, or an _Adder_ base class. In Crystal you _can_ use similar concepts, but you are not forced to.

### The cost of freedom, part one

Such freedom comes with some cost. For instance, in this example, the reporting of errors might be surprising. Imagine that we write `adder "hi", 2`. Appending a number to a string is not possible in Crystal, so this is an error, and the compiler let us know:

![alt text](image-1.png)

Now, look at where the compiler puts the blame: in the usage of the `+` operator. In Ruby, this code will fail at this exact same location _at runtime_. Instead, Crystal let us know before hand, so the program won't explode in our face. That's nice, but still, the error is located too deep in the code (here it's obvious, but imagine what would happen in a longer chain of calls). If we have to find a better place for the error to surface, it should be in the call of `adder` with two incompatible types.

One option is to restrict the types of `x` and `y` to be the same. This restricts our function, and forbids adding a `Char` to a `String` (a valid operation), but at the benefit of having better error reporting. In order to specify that `adder` needs both arguments to be of the same type, we specify that it's _parametric_ in a type `T` that should be shared for the two arguments. Now the error is better:

![alt text](image.png)

As mentioned before, we can decide how much bureaucracy we want to add. We can, for instance, go one step further and create an interface of "things that can be added". This won't help with the problem of mixing valid types (like `String` and `Char`), but will improve the error if we call `adder` with anything that is not _addable_.

Let's turn our attention to another interesting feature of Crystal.

### Monkey patching

Monkey patching is the ability to change the behavior of a class without changing its source code. A good example I see quite often is when patching the code of a library before the maintainer of the library makes a release with the fix. For instance, in [this PR](https://github.com/crystal-lang/crystal/pull/13050) Brian provides the code to monkey-patch a system's library for those people that can't wait for the next release to be on the streets:

```cr
class Crystal::SpinLock
  def lock
    previous_def
    ::Atomic::Ops.fence :sequentially_consistent, false 
  end

  def unlock
    ::Atomic::Ops.fence :sequentially_consistent, false 
    previous_def
  end
end
```

Without looking much into the details, first note that this code is expected to be on the user code, not on the code where `SpinLock` class is. Next, this code replaces the `lock` and `unlock` methods of `Crystal::SpinLock` with a version that adds a _memory fence_. `previous_def` is the call of the previous definition. The effect is that now every call to these two methods in the program or standard library will be replaced with these new definitions.

Again, similar to duck typing, if you come from a dynamic language this might not raise an eyebrow. But if you come from a static language, this _is_ surprising. After all, how are we replacing the _call_ of a method with a call to a new method? In Java or C++ you simply can't do this, and the only way to extend the functionality of a class is with a wrapper class, or inheriting from the class. But then, you're just creating _a new type_, and not replacing the existing calls, making this attempt mute to solve an issue in existing code.

Side note: Here we're talking about replacing the methods of the class, but in Crystal you can also add instance or class variables through monkey-patching.

Let's make one thing clear: monkey patching is frown upon, and for a good reason: changing the way a class work for an entire program might come with unexpected consequences. This post is not encouraging its use. In fact, there's some abuse of it in the standard library that I'd like to get rid of at some point... Yet, as the example described here, it is still a very useful tool to help overcome limitations in code that is out of reach.

Having said that, it's not the ideological purism of traditional static languages that makes them reject duck typing and monkey patching.

### The bureaucracy is necessary for _most_ static languages

Languages like Java and C++ force you to add a ton of bureaucracy
—that is, to be explicit about types— for a very good reason. As it turns out, traditional compilers break down a program into _units of compilation_ (for instance, the classes of the program), and performs the compilation on each of them. This process is _modular_: each unit contains all the information required to compile it in isolation with the other units.

But in order to achieve modular compilation, the compiler needs to be sure that local changes in one unit won't impact other units. By having interfaces and declared types in each function, the compiler makes sure that, as long as there is no change in the _public interface_ of the unit, then other units depending on it won't have to be recompiled.

Let's cement this concept with an example in Java. We have classes `A` and `B`, the former depending on the latter:

```java
class A {
    public static void main(String[] args) {
        var b = new B();
        System.out.println(b.hello("world"));
    }
}

class B {
    public String hello(String something) {
        return "Hello " + something;
    }
}
```

Under the assumption that each class is a unit of compilation, the compiler will split this program in two: unit `A` and unit `B`. When compiling `B`, it will record that it has a public method, `hello`, that takes a `String` argument and returns a `String`. It uses this information when compiling `A`, since its `main` function creates an instance of `B` and calls that method.

If tomorrow we decide to change the `main` method to call `b.hello` again, say, with another `String`, the compiler will only build the unit `A` again, re-using the same _object code_ (that is, the output of the compilation) of unit `B` that was already produced.

So, in essence, modularity gives us a fast compilation process, at the expense of requiring precise information about the input and output types of each method and, moreover, forbidding the laxity provided by duck typing and monkey patching. If, for instance, monkey patching were allowed, a change like in our `SpinLock` example would require a recompilation of every unit that uses it (and, maybe, other units recursively!). This is unacceptable to such languages.

## Unveiling the hard truth: Crystal throws away modular compilation

In Crystal, every time the compiler runs, it analyses the entire program, including the code of any library in which it depends on. Then, it takes advantage of having a global view of the program to modify any behavior based on what's been monkey-patched.

### The cost of freedom, part two

Before, we mentioned that the freedom of Crystal, and of dynamic languages, comes with the cost of having errors be marked in places away from the originating place. In Crystal, this issue can be easily overcome by adding proper typing annotation, for instance, once the program's architecture is settled and we're certain not to introduce new changes.

But there's another, seemingly unavoidable, cost: the cost of not being able to share much information from one compiler run to the next one. Let's see this with the example about classes `A` and `B` that we presented above using Java. In Crystal, we can code it as follows:

```crystal
class A
  def A.main
    b = B.new
    puts b.hello("world")
  end

  A.main
end

class B
  @[NoInline]
  def hello(something : String) : String
    "Hello " + something
  end
end
```

Note: The `NoInline` attribute is important for the story, I will later comment on what happens without it.

When we compile it using `crystal build --stats`, the first time we obtain:

```
Parse:                             00:00:00.001240202 (   1.07MB)
Semantic (top level):              00:00:00.475975578 ( 113.76MB)
Semantic (new):                    00:00:00.001448348 ( 113.76MB)
Semantic (type declarations):      00:00:00.029361038 ( 121.76MB)
Semantic (abstract def check):     00:00:00.013879329 ( 121.76MB)
Semantic (restrictions augmenter): 00:00:00.006792311 ( 121.76MB)
Semantic (ivars initializers):     00:00:00.027150540 ( 129.76MB)
Semantic (cvars initializers):     00:00:00.097840260 ( 137.76MB)
Semantic (main):                   00:00:00.048381045 ( 153.76MB)
Semantic (cleanup):                00:00:00.000374572 ( 153.76MB)
Semantic (recursive struct check): 00:00:00.000649120 ( 153.76MB)
Codegen (crystal):                 00:00:00.413804076 ( 169.76MB)
Codegen (bc+obj):                  00:00:00.631833429 ( 169.76MB)
Codegen (linking):                 00:00:00.445881542 ( 169.76MB)
dsymutil:                          00:00:00.177000360 ( 169.76MB)

Codegen (bc+obj):
 - no previous .o files were reused
```

If we then change the `main` function to call twice `b.hello`, the second run shows the following:

```
Parse:                             00:00:00.004092567 (   1.07MB)
Semantic (top level):              00:00:00.461138153 ( 113.70MB)
Semantic (new):                    00:00:00.001845896 ( 113.70MB)
Semantic (type declarations):      00:00:00.031534704 ( 121.70MB)
Semantic (abstract def check):     00:00:00.013377533 ( 121.70MB)
Semantic (restrictions augmenter): 00:00:00.006629351 ( 121.70MB)
Semantic (ivars initializers):     00:00:00.032184823 ( 129.70MB)
Semantic (cvars initializers):     00:00:00.154897815 ( 145.76MB)
Semantic (main):                   00:00:00.038617113 ( 145.76MB)
Semantic (cleanup):                00:00:00.000740812 ( 145.76MB)
Semantic (recursive struct check): 00:00:00.000580844 ( 145.76MB)
Codegen (crystal):                 00:00:00.442429527 ( 169.76MB)
Codegen (bc+obj):                  00:00:00.070046242 ( 169.76MB)
Codegen (linking):                 00:00:00.399856339 ( 169.76MB)
dsymutil:                          00:00:00.192376071 ( 169.76MB)

Codegen (bc+obj):
 - 296/297 .o files were reused

These modules were not reused:
 - A (A-.o0.bc)
```

If we look closely, we note the following:

1. The times up to `Codegen` are more or less the same: there was no gain in the two successive runs.
2. From `Codegen`, the step `bc+obj` was the one that benefited from having been run before: from 631ms it went down to 70ms. And the explanation is in the following text: except for `A`, all of the other units were reused in the second run.

What this example is showing is that every process that happens before the generation of the object files is not reused. The `Parser` phase is the one that reads the file and produces an internal representation of the program. The `Semantic` phase has several steps, but basically it annotates with types and validates the code as produced by the parser. The `Codegen` phase is split in three steps: the `crystal` step takes the result from the `Semantic` phase and produces the code for LLVM, the library that is in charge of producing the binaries. That is, from a Crystal program it produces an LLVM-program (in what's called the _Intermediate Representation Language_, or IR, of LLVM). Here Crystal makes the only saving of time: if the IR of a unit is byte-by-byte identical to that of a previous run, then it's object code is reused. This is what the compiler informs us with the number of files reused.

With this little example, a few milliseconds might not seem much. But in large developments, compilation times start being noticeable. With a hundred thousand lines of code, it can well take some seconds to compile.

## Can we do better?

Crystal creator Ary explained in detail [in a series of posts](https://dev.to/asterite/incremental-compilation-for-crystal-part-1-414k) what's the difficulty of making the compilation incremental. 