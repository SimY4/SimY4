+++
title = "Mapping between categories"
date = "2019-03-01"
tags = ["fp"] 
categories = ["fp"]
+++

So, once we defined a category and played with them a little bit we start wondering if categories can themselves be mapped somehow. And some of them actually can. Today we're going to talk about functors which is a mapping between two categories: `Category[A] => Category[F[B]]`. In order to map one category to another, we need to map all the objects and arrows of the original category. Using the model of category from the previous post we can define a functor like so:

```scala
trait Functor[F[_]] {
  type FromCat[A, B]
  type ToCat[A, B]

  def map[A, B](f: FromCat[A, B])
               (implicit FromCat: Category[FromCat], 
                         ToCat: Category[ToCat]): ToCat[F[A], F[B]]
}
```

Obviously, if we can map one category to another we can also map one category to itself. In this case, we call our functor an endofunctor ('endo' means the same). Most of the time when we use functors in Scala or Haskell we actually deal with endofunctors.

```scala
trait EndoFunctor[F[_]] extends Functor[F] {
  type Cat[A, B]
  override type FromCat[A, B] = Cat[A, B]
  override type ToCat[A, B] = Cat[A, B]

  def mapEndo[A, B](f: Cat[A, B])(implicit cat: Category[Cat]): Cat[F[A], F[B]] =
    map(f)(cat, cat)
}

// given a function A -> B and F[A], we can produce F[B] given that F[_] is an endofunctor
type Function1EndoFunctor[F[_]] = EndoFunctor[F] {
  type Cat[A, B] = A => B
}

// if A <:< B then F[A] <:< F[B] given that F[_] is an endofunctor
type TypeHierarchyEndoFunctor[F[_]] = EndoFunctor[F] {
  type Cat[A, B] = A <:< B
}
```

`Function1EndoFunctor` and `TypeHierarchyEndoFunctor` are also called covariant functors because they preserve the direction of arrows. If in the resulting category the arrows go in the opposite way such functor is called contravariant functor. Now, since we kinda already know that functions and types form a category, we can remove the `Category[Function1]` instance evidence from `Function1EndoFunctor` to obtain a quite familiar definition of a functor:

```scala
// Actually it's endofunctor
//         \/
trait Functor[F[_]] {
  def map[A, B](f: A => B): F[A] => F[B]
}
```

And the definition on a monad we came up with in the last blog post is sufficient to derive a functor for it:

```scala
trait Monad[F[_]] extends Functor[F] {
  def pure[A](a: A): F[A]
  def >>=[A, B](fa: F[A])(f: A => F[B]): F[B]
  override def map[A, B](f: A => B): F[A] => F[B] = { fa => fa >>= (f andThen pure) }
}
```
