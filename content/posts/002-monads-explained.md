+++
title = "Monads Explained"
date = "2019-02-01"
tags = ["fp"]
categories = ["dev-blog"]
+++

## Referential transparency

Referential transparency is a nice property of pure functional programming languages that allow us to reason about the behaviour of our programs. 
It guarantees that all expressions are always evaluating to the same result in any context. What it gives to the programmer is an ease of equational reasoning about any pure program as they used to do it in math. 

```scala
2 + (2 * 2) === 2 + 4 === 6
"Hello" + " " + "world" === "Hello " + "world" === "Hello" + " world" === "Hello world"
List(1, 2, 3) ++ List(4, 5, 6) === List(1, 2, 3, 4, 5, 6)
g(f(a)) === (g compose f)(a)
```

If you think about it, most of the modern compilers (non-FP included) can already do some optimisations for referentially transparent code fragments. 
For example, JIT can perform dead code elimination which can evaluate arithmetic expressions into a constant value or interpolate string literals or get rid of staged builders. 
But some fragments they are not allowed to optimise just because they can't prove that this change will not break code's original semantics. 
So not only making our code referentially transparent give us better code readability but also it helps the compiler to perform certain optimisations that can make your code more performant.

So, why something can be non-referentially transparent? Well, basically every expression containing side-effect or mutable state breaks the property of referential transparency. 

```scala
val iterator = List(1, 2, 3).iterator
val next = iterator.next
next + next =!= iterator.next + iterator.next

import scala.concurrent._
import ExecutionContext.Implicits.global
import scala.concurrent.duration._
import scala.io.StdIn._

def readNumber = Await.result(Future { readLine("Give me a number>").toInt }, 10.seconds)
val number = readNumber
number + number =!= readNumber + readNumber
```

You may wonder, how can our programs do anything useful without being able to perform side-effects? Well, let's talk about effects and side effects.

## Effects and side-effects

So, to satisfy the property of referential transparency the programmer needs to keep the code free from any side-effects. One of the ways to achieve this is by lifting all side-effects into an effect. 
What is an effect in functional programming? Effect is a result of a function that simply isn't pure. So, what's a pure function then? A pure function is a function where the return value is determined only by its input arguments. 
A few examples of pure functions are: 

```scala
def identity[A](a: A): A = a
def min(a: Long, b: Long): Long = if (a < b) a else b
def reverse(s: String): String = 
  s.foldRight(new StringBuilder(s.length)) { (ch, sb) => 
    sb.append(ch) 
  }.toString
```

And what's are effectful functions then? We can call any function in the form of `A => F[B]` where `F` is a container that encodes the possibility of the function to perform some sort of effectful action 
(i.e. return abnormally by throwing an exception, produce some output, etc). Keep in mind that having an effect as a return type doesn't give us the right to break referential transparency 
(for example `scala.concurrent.Future` look like a suitable type, it's by design not referentially transparent therefore can't be used to represent pure FP effects). 
Some examples of standard effects and effectful functions are:

```scala
def div(a: Int, b: Int): Option[Int] = b match
  case 0 => None
  case _ => Some(a / b)
//          F[B] is Either[String, B]
//                    \/
def toInt(s: String): Either[String, Int] = s.toIntOption.toRight("NaN")
def range(i: Int): List[Int] = List.range(0, i) // <- list is effect too
```

The major problem with effectful return types is that the standard rule of functional composition can't be applied to them. 
You can easily compose functions of type `A => B` and `B => C` to get the function `A => C` but you can't compose effectful functions `A => F[B]` and `B => F[C]`. Or can you?

## Kleisli composition

So say, we moved all of our side-effects into a nice exclict effects space, and we ended up having all these boilerplate for unpacking and inspecting the internal state of our effects in order to proceed further:

```scala
val maybeUser: Option[User] = getUserById(UserId(1))
val maybeAccountId: Option[AccountId] = maybeUser match
  case Some(user) => getUserAccountId(user)
  case None => None

val maybeUserAccount: Option[UserAccount] = maybeAccountId match
  case Some(accountId) => getAccount(accountId)
  case None => None
```

There's a lot of friction here to compose these types together and it's very hard to follow what this code actually intended to do. 
What if we can add an abstraction that could hide all this boilerplate from us? Let's try to figure something out. 
We can define a trait for the composition of two effectful functions. It'll have just one method that'll look like a fancy arrow (or fish arrow, 
depending on how you look at it):

```scala
trait Fancy[F[_]]:
  extension [A, B](fab: A => F[B]) def >=>[C](fbc: B => F[C]): A => F[C]

object Fancy:
  given Fancy[Option] with
    extension [A, B](fab: A => Option[B]) 
      def >=>[C](fbc: B => Option[C]): A => Option[C] = { a => 
        fab(a) match
          case Some(b) => fbc(b)
          case None => None
      }
```

Having these definitions in place we can rewrite our code into something like this:

```scala
val getAccount: UserId => Option[UserAccount] = 
  getUserById >=> getUserId >=> getUserAccount
  
getAccount(UserId(1))
```

It seems that our fancy arrow gives us the missing ability to compose functions of shape `A => F[B]` in the same way as we used to compose pure functions. 
And it turns out that this composition that we just came up with, is well known in a branch of mathematics called category theory and is called Kleisli composition 
(named after mathematician [Heinrich Kleisli](https://en.wikipedia.org/wiki/Heinrich_Kleisli "wiki: Heinrich Kleisli")). 
Our fancy arrow is called Kleisli arrow and as long as they are pure, and referentially transparent they form the Kleisli category. 

## Brief introduction to Category theory

Let's briefly talk about Category theory. Category theory is a branch of set theory where the focus was shifted on relationships between sets rather than sets themselves. 
So, in category theory, we say that all sets are an atomic object that we can't look into and all relationships between sets are arrows that go from one atomic object to another with a few extra rules. 
First, all objects must have at least one identity arrow `id: A => A` that starts and ends on the same object. 
Second, all arrows have to be composable, meaning that if you have two arrows `f: A => B` and `g: B => C` there should always be at least one arrow `g . f: A => C` and 
these compositions must be associative (`(f . g) . h == f . (g . h)`). It's pretty much all we need to know at this point, having these rules we can model them in scala:

```scala
trait Category[:=>[_, _]]:
  def id[A]: A :=> A

  extension [A, B](f: A :=> B) 
    def compose[C](g: B :=> C): A :=> C 

object Category extends CategoryInstances0 with CategoryInstances1
```

Given this definition of a category, we can implement a few the most frequently used examples of categories:

```scala
trait CategoryInstances0:
  // Function1 category instance
  given Category[Function1] with
    def id[A]: A => A = identity

    extension [A, B](f: A => B)
      def compose[C](g: B => C): A => C = g compose f

  // Type hierarchy category instance
  //      <:<[-A, +B] is an standard type evidence that A is a subtype of B
  //             \/
  given Category[<:<] with
    def id[A]: A <:< A = implicitly[A <:< A] // A is always a subtype of itself

    extension [A, B](f: A <:< B)
      def compose[C](g: B <:< C): A <:< C = g compose f
```

A category with a single object is called monoid. Now, let's try to define the Kleisli category in a generic way. 
To do this we'll need one extra helper trait with some standard scala plumbing to make it into a nice syntax.

```scala
trait Shmancy[F[_]]:
  def pure[A](a: A): F[A]

  extension [A, B](f: A => F[B]) def >>=(fa: F[A]): F[B]

trait CategoryInstances1:
  // Kleisli category instance
  given [F[_]](using F: Shmancy[F]): Category[[A, B] =>> A => F[B]] with 
    def id[A]: A => F[A] = F.pure

    extension [A, B](f: A => F[B]) 
      def compose[C](g: B => F[C]): A => F[C] = a => g >>= f(a)
```

Here we are. So, to make things compose in Kleisly category we need only two things, a function that can lift a pure value into an effect `F[_]` 
and a binding function that can apply function `A => F[B]` to `F[A]`. This definition is called a monad.

## Mapping between categories

So, once we defined a category and played with them a little bit we start wondering if categories can themselves be mapped somehow. 
And some of them actually can. Today we're going to talk about functors which is a mapping between two categories: `Category[A] => Category[B]`. 
In order to map one category to another, we need to map all the objects and arrows of the original category. 
Using the model of category from the previous post we can define a functor like so:

```scala
trait Functor[F[_]]:
  type FromCat[A, B]
  type ToCat[A, B]

  def map[A, B](f: FromCat[A, B])
               (using Category[FromCat], Category[ToCat]): ToCat[F[A], F[B]]
```

Obviously, if we can map one category to another we can also map one category to itself. In this case, we call our functor an endofunctor ('endo' means the same). 
Most of the time when we use functors in Scala or Haskell we actually deal with endofunctors.

```scala
trait EndoFunctor[F[_]] extends Functor[F]:
  type Cat[A, B]
  override type FromCat[A, B] = Cat[A, B]
  override type ToCat[A, B] = Cat[A, B]

  def endo[A, B](f: Cat[A, B])
                (using Category[Cat]): Cat[F[A], F[B]]

// given a function A -> B and F[A], we can produce F[B] given that F[_] is an endofunctor
type Function1EndoFunctor[F[_]] = EndoFunctor[F] {
  type Cat[A, B] = A => B
}

// if A <:< B then F[A] <:< F[B] given that F[_] is an endofunctor
type TypeHierarchyEndoFunctor[F[_]] = EndoFunctor[F] {
  type Cat[A, B] = A <:< B
}
```

`Function1EndoFunctor` and `TypeHierarchyEndoFunctor` are also called covariant functors because they preserve the direction of arrows. 
If in the resulting category the arrows go in the opposite way such functor is called contravariant functor. 
Now, since we kinda already know that functions and types form a category, we can remove the `Category[Function1]` instance evidence from 
`Function1EndoFunctor` to obtain a quite familiar definition of a functor:

```scala
// Actually it's endofunctor
//         \/
trait Functor[F[_]]:
  extension [A, B](f: A => B) def map(fa: F[A]): F[B]
```

And the definition on a monad we came up with in the last blog post is sufficient to derive a functor for it:

```scala
trait Monad[F[_]] extends Functor[F]:
  def pure[A](a: A): F[A]
  extension [A, B](f: A => F[B]) def >>=(fa: F[A]): F[B]
  extension [A, B](f: A => B) def map(fa: F[A]): F[B] = (f andThen pure) >>= fa 
```
