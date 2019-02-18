+++
title = "Kleisli composition"
date = "2019-02-15"
tags = ["fp"] 
categories = ["fp"]
+++

So say, we moved all of our side-effects into a nice exclict effects space, and we ended up having all these boilerplate for unpacking and inspecting the internal state of our effects in order to proceed further:

```scala
val maybeUser: Option[User] = getUserById(UserId(1))
val maybeAccountId: Option[AccountId] = maybeUser match {
    case Some(user) => getUserAccountId(user)
    case None => None
  }
val maybeUserAccount: Option[UserAccount] = maybeAccountId match {
    case Some(accountId) => getAccount(accountId)
    case None => None
  }
```

There's a lot of friction here to compose these types together and it's very hard to follow what this code actually intended to do. What if we can add an abstraction that could hide all this boilerplate from us? Let's try to figure something out. We can define a trait for the composition of two effectful functions. It'll have just one method that'll look like a fancy arrow (or fish arrow, depending on how you look at it):

```scala
// parameterized by any type which is itself parameterised by single type argument
//          \/
trait Fancy[F[_]] {
  def >=>[A, B, C](fab: A => F[B], fbc: B => F[C]): A => F[C]
}

object Fancy {
  // infix syntax definition in companion object
  implicit class FancyOp[F[_], A, B](private val fab: A => F[B]) extends AnyVal { 
    // here we require that our F[_] should have an implicit instance of Fancy
    //                                \/
    def >=>[C](fbc: B => F[C])(implicit F: Fancy[F]): A => F[C] = 
      F.>=>(fab, fbc)
  }

  // implicit Fancy trait instances for syntax

  implicit val fancyOption = new Fancy[Option] {
    override def >=>[A, B, C](fab: A => Option[B], fbc: B => Option[C]): A => Option[C] = { a => 
      fab(a) match {
        case Some(b) => fbc(b)
        case None => None
      }
    }
  }
}
```

Having these definitions in place we can rewrite our code into something like this:

```scala
val getAccount: UserId => Option[UserAccount] = getUserById _ >=> getUserId _ >=> getUserAccount _
getAccount(UserId(1))
```

It seems that our fancy arrow gives us the missing ability to compose functions of shape `A => F[B]` in the same way as we used to compose pure functions. And it turns out that this composition that we just came up with, is well known in a branch of mathematics called category theory and is called Kleisli composition (named after mathematician [Heinrich Kleisli](https://en.wikipedia.org/wiki/Heinrich_Kleisli "wiki: Heinrich Kleisli")). Our fancy arrow is called Kleisli arrow and as long as they are pure, and referentially transparent they form a Kleisli category. 
