---
title: "The Secret of Crystal"
author: beta-ziliani
summary: "TBD"
categories: community
tags: [language]
---

In this post I place Crystal in the map of programming languages. If you haven't yet dirtied your hands with the language, you need to know that the language sits in a pretty much uncharted part of the territory of languages. Here I explain what's so special about it, and what is Crystal secret weapon.

In short: Crystal is _statically typed_ and compiled like Java and C++, but free of bureaucracy like Ruby or Python.

# A bureaucracy-free language

Crystal lets you write code as if you were in a dynamic language like Ruby or Python. In particular, it has two capabilities that are rarely found in a static language: _duck typing_ and _monkey patching_.

## Duck typing

Look at the following valid Ruby and Crystal code:

```cr
def adder(x, y)
  x + y
end

adder 1, 2 # => 3
adder "hi", "world" # => hiworld
```

It's using duck typing:

> If it walks like duck, and talks like a duck, it's a duck

In this case, since both numbers and strings can be operated with `+`, then they indeed are!

This might not be surprising if you come from a dynamic language. But if you come from a static language, this certainly is. In those language, you need to specify in some way that `x` and `y` can be added, via an interface or base class. In Crystal you _can_ do it, but you _don't need to_.

## Monkey patching

Monkey patching is the ability to change code of a class without changing the source code of the class. An example I see quite often is when patching the code of a library before the maintainer of the library makes a release with the fix. For instance, in [this PR](https://github.com/crystal-lang/crystal/pull/13050) Brian provides the code to monkey-patch a system's library for those people that can't wait for the next release to be on the streets:

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

Without looking much into the details, what this code does is _to replace_ the `lock` and `unlock` methods of `Crystal::SpinLock` with a version that adds a _fence_. `previous_def` is the call of the previous definition. The effect is that now every call to these methods will be replaced with the new definition.

Again, if you come from a dynamic language, this is not biggie, but if you come from a static language, this _is_ surprising. After all, how are we replacing the _call_ of a method with a call to a new method? In Java, you can't certainly not do this, and the only way to extend the functionality of a class is with a wrapper class, or inheriting from the class. But then, you're just creating _a new type_, and not replacing the existing calls, making this attempt mute to solve an issue in existing code.

Note: Here we're talking about just the methods of the class, but in Crystal you can also add instance or class variables through monkey-patching.

## The bureaucracy is necessary for static languages

When Java or C++ forces you to add an interface or to inherit a type in order to extend its functionality, they do it for a good reason. When in Java you write the following code:

```java
class MyNumber implements Adder {

  public MyNumber add(MyNumber other) {
    /* implementation of add method */
  }

}
```

You're basically adding a _virtual table_ in `MyNumber` that specifies where to find the `add` method that is specified in the interface `Adder` (not shown).