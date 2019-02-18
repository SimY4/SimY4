+++
title = "Brief introduction to Category theory"
date = "2019-02-22"
tags = ["fp"] 
categories = ["fp"]
+++

Let's briefly talk about Category theory. Category theory is a branch of set theory where the focus was shifted on relationships between sets rather than sets themselves. So, in category theory, we say that all sets are an atomic object that we can't look into and all relationships between sets are arrows that go from one atomic object to another with a few extra rules. First, all objects must have at least one identity arrow `id: A -> A` that starts and ends on the same object. Second, all arrows have to be composable, meaning that if you have two arrows `f: A -> B` and `g: B -> C` there should always be at least one arrow `g . f: A -> C` and these compositions must be associative (`(f . g) . h == f . (g . h)`). It's pretty much all we need to know at this point, having these rules we can model them in scala:

```scala
  // Parameterized with a type with two type holes in it
  //           \/
trait Category[F[_, _]] {
  def id[A]: F[A, A]
  def compose[A, B, C](f: F[A, B], g: F[B, C]): F[A, C] 
}

object Category extends CategoryInstances0 with CategoryInstances1
```

Given this definition of a category, we can implement a few the most frequently used examples of categories. These are the category of functions and the category of type hierarchies.

```scala
trait CategoryInstances0 {
  // Function1 category instance
  implicit val functionsCategory: Category[Function1] = new Category[Function1] {
    def id[A]: A => A = identity
    def compose[A, B, C](f: A => B, g: B => C): A => C = g compose f
  }

  // Type hierarchy category instance
  //      <:<[-A, +B] is an standard type evidence that A is a subtype of B
  //                                           \/
  implicit val typeHierarchyCategory: Category[<:<] = new Category[<:<] {
    def id[A]: A <:< A = implicitly[A <:< A] // A is always a subtype of itself
    def compose[A, B, C](f: A <:< B, g: B <:< C): A <:< C = g compose f
  }
}
```

Now, let's try to define the Kleisli category in a generic way. To do this we'll need one extra helper trait with some standard scala plumbing to make it into a nice syntax.

```scala
trait Shmancy[F[_]] {
  def pure[A](a: A): F[A]
  def >>=[A, B](fa: F[A])(f: A => F[B]): F[B]
}

object Shmancy {
  implicit class ShmancyOp[F[_], A, B](private val fa: F[A]) extends AnyVal { 
    def >>=[B](f: A => F[B])(implicit F: Shmancy[F]): F[B] = 
      F.>>=(fa)(f)
  }
}

trait CategoryInstances1 {
  // Kleisli category instance
  //                 Type lamdba that encodes type constructor [A, B] => Function1[A, F[B]]
  //                                                                         \/
  implicit def kleisliCategory[F[_]](implicit F: Shmancy[F]): Category[({ type T[A, B] = Function1[A, F[B]] })#T] = 
    new Category[({ type T[A, B] = Function1[A, F[B]] })#T] {
      def id[A]: A => F[A] = F.pure _
      def compose[A, B, C](f: A => F[B], g: B => F[C]): A => F[C] = { a => f(a) >>= g }
    }
}
```

Here we are. So, to make things compose in Kleisly category we need only two things, a function that can lift a pure value into an effect `F[_]` and a binding function that can apply function `A => F[B]` to `F[A]`. This definition is called a monad.
