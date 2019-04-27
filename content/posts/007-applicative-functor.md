+++
title = "Applicative functor"
date = "2019-04-27"
tags = ["fp"] 
categories = ["fp"]
+++

We now have two of the most powerful abstractions in our hands - Monad and Functor. And we can do a lot with having just that. But at a certain point, they may still seem a little bit limited to you. The questions that may ask yourself is: how can I structure my code to be able to compose multiple effects running in parallel? Let's see if what we can do with functors and monads first.

```scala
def doSomeWork[F[_]]: F[String] = ??? // <- unit or work

// we want to compose two units of work to get the results of two running in parallel

def composeTwoWithFunctor[F[_]: Functor]: F[(String, String)] = 
  doSomeWork[F] map { r1 => doSomeWork[F] map { r2 => (r1, r2) } }
  //                                /\
  // Not gonna work. this map will produce the value of type F[C] which is not 
  // the expected type of the outer map function.

def composeTwoWithMonad[F[_]: Monad]: F[(String, String)] = 
  doSomeWork[F] >>= { r1 => doSomeWork[F] >>= { r2 => (r1, r2).pure[F] } }
  //                  /\
  // This would work and type signatures would match but because of the stucture of 
  // our bind function (>>=) we can only make this work sequantially.
```

In the example above we can see that monads and functors are naturally sequential. In order to produce the value of type `F[B]` we need to know the value of `A` to feed it into the function `A => F[B]`. And in order for us to know the value of `A` we need to wait on the first effect to complete. So, monads can't model parallelism. That feels like our abstractions missing something very crusial to start using them for any CS problem. We would want something with the signature that would allow us to compose two effects without waiting on previous value. Something like `zip[F[_], A, B](fa: F[A], fb: F[B]): F[(A, B)]` but for an arbitrary number of effects. And in fact, there’s one abstraction that we didn’t talk about yet but that can address this exact problem and it’s called an applicative functor. An applicative functor is defined like this:

```scala
trait ApplicativeFunctor[F[_]] extends Functor[F] {
  def pure[A](a: A): F[A] // <- straight from the monad definition

  def ap[A, B](ff: F[A => B]): F[A] => F[B]

  // derived functions

  override def map[A, B](f: A => B): F[A] => F[B] =
    ap(pure(f))

  def zip(fa: F[A], fb: F[B]): F[(A, B)] = 
    ap(map(a => (b: B) => (a, b))(fa))(fb)
}
```

As we can see from the definition of an `ap` function it takes two effects `F[A => B]` and `F[A]` and produces next effect `F[B]`. No runtime value dependency, pure effect type composition. Having an applicative functor we can derive a function that can compose an arbitrary number of effects together:

```scala
// map instances for 2, 3 and 4 effects
    
def map2[A, B, Z](fa: F[A], fb: F[B])(f: (A, B) => Z): F[Z] =
  F.map({ case (a, b) => f(a, b)})(F.zip(fa, fb))

def map3[F[_], A, B, C, Z](fa: F[A], fb: F[B], fc: F[C])(f: (A, B, C) => Z)(implicit F: Applicative[F]): F[Z] = 
  F.map({ case (a, (b, c)) => f(a, b, c)})(F.zip(fa, F.zip(fb, fc)))
  
def map4[F[_], A, B, C, D, Z](fa: F[A], fb: F[B], fc: F[C], fd: F[D])(f: (A, B, C, D) => Z)(implicit F: Applicative[F]): F[Z] =  
  F.map({ case (a, (b, (c, d))) => f(a, b, c, d)})(F.zip(fa, F.zip(fb, F.zip(fc, fd))))
  
// our composition using Applicative for the problem statement above will look like this

def composeTwoWithApplictive[F[_]](implicit F: Applicative[F]): F[(String, String)] = 
  F.zip(doSomeWork[F], doSomeWork[F])
```

We now have three the most widely used combinators of pure functional programming - functor, applicative functor and monad. If you think about it, they look very similar but they serve completely different purposes.

```scala
map(f: A => B)   : F[A] => F[B] // <- functor.             Transforms values inside an effect
ap (f: F[A => B]): F[A] => F[B] // <- applicative functor. Composes two effects together
>>=(f: A => F[B]): F[A] => F[B] // <- monad.               Transforms effect into a new effect
```

And of course since applicative is a functor we can again include it in our monad definition:

```scala
trait Monad[F[_]] extends ApplicativeFunctor[F] {
  def pure[A](a: A): F[A]
  def >>=[A, B](f: A => F[B]): F[A] => F[B]
  override def ap[A, B](ff: F[A => B]): F[A] => F[B] = { fa => >>=({ (f: A => B) => map(f)(fa) })(ff) }
  override def map[A, B](f: A => B): F[A] => F[B] = >>=(f andThen pure)
}
```
