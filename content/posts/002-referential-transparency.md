+++
title = "First rule: we make everything referentially transparent"
date = "2019-02-02"
tags = ["fp"] 
categories = ["fp"]
+++

Referential transparency is a nice property of pure functional programming languages that allow us to reason about the behaviour of our programs. It means that an expression always evaluates to the same result in any context. What it gives us is the ease of equational reasoning about our programs as we used to do in math. 

```scala
2 + (2 * 2) <-> 2 + 4 <-> 6
"Hello" + " " + "world" <-> "Hello " + "world" <-> "Hello" + " world" <-> "Hello world"
List(1, 2, 3) ++ List(4, 5, 6) <-> List(1, 2, 3, 4, 5, 6)
```

If you think about it, most of the modern compilers (non-FP included) can already do some optimisations for referentially transparent code fragments. For example, they can perform dead code elimination which can evaluate arithmetic expressions into a constant value, interpolate string literals or get rid of staged builders. But some fragment they are not allowed to optimise just because they can't prove that this change will not break code's original semantics. So not only making our code referentially transparent give us better code readability but also it helps the compiler to perform certain optimisations that can make your code more performant.

So, why something can be non-referentially transparent? Well, basically every expression containing side-effect or mutable state breaks the property of referential transparency. 

```scala
val iterator = List(1, 2, 3)
val next = iterator.next
next + next <!> iterator.next + iterator.next

import scala.concurrent._
import ExecutionContext.Implicits.global
import scala.concurrent.duration._
import scala.io.StdIn._

def readNumber = Await.result(Future { readLine("Give me a number>").toInt }, 10.seconds)
val number = readNumber
number + number <!> readNumber + readNumber
```

You may wonder, how can our programs do anything useful without being able to perform side-effects? Well, let's talk about effects and side effects in the next blog post.