+++
title = "Natural transformation"
date = "2019-03-09"
draft = true
tags = ["fp"] 
categories = ["fp"]
+++

Now, we finally reached the maximum level of abstraction we would need to build something useful. From functor definition we know that it maps a category to another category. And endo-functors can map one category to itself. If these mappings are associative we can build the functor category.

```scala
trait CategoryInstances2 {
  implicit def functorCategory[F[_]: Functor, G[_]: Functor]: Category[({ type T[A, B] = Function1[F[A], G[B]] })#T] = 
    new Category[({ type T[A, B] = Function1[F[A], G[B]] })#T] {
      def id[A]: F[A] -> G[A]
      def compose[A, B, C](f: F[A] -> G[B], g: F[B] -> G[C]): F[A] -> G[C]
    }
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
