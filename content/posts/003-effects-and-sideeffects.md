+++
title = "Effects and side-effects"
date = "2019-02-09"
tags = ["fp"] 
categories = ["fp"]
+++

So, to satisfy the property of referential transparency we need to keep our code free from any side-effects. One of the ways to achieve this is by lifting all side-effects into an effect. What is an effect in functional programming? Effect is a result of a function that isn't pure. So, what's a pure function then? A pure function is a function where the return value is determined only by its input arguments. A few examples of pure functions are: 

```scala
def identity[A](a: A): A = a
def min(a: Long, b: Long): Long = if (a < b) a else b
def reverse(s: String): String = s.foldLeft("") { (acc, ch) => ch + acc }
```

And what's are effectful functions then? We can call any function in the form of `A => F[B]` where `F` is a container that encodes the possibility of the function to perform some sort of an effectful action (i.e. return abnormally by throwing an exception, produce some output, etc). Keep in mind that having an effect as a return type doesn't give us the right to break referential transparency (for example `scala.concurrent.Future` look like a suitable type, it's by design not referentially transparent therefore can't be used to represent pure FP effects). Some examples of standard effects and effectful functions are:

```scala
def div(a: Int, b: Int): Option[Int] = b match {
  case 0 => None
  case _ => Some(a / b)
}
//          F[B] is Either[String, B]
//                    \/
def toInt(s: String): Either[String, Int] = try {  
    Right(s.toInt) 
  } catch { 
    case _: NumberFormatException => Left("NaN") 
  }
def range(i: Int): List[Int] = List.range(0, i) // <- list is effect too
```

The major problem with effectful return types is that the standard rule of functional composition can't be applied to them. You can easily compose functions of type `A => B` and `B => C` to get the function `A => C` but you can't compose effectful functions `A => F[B]` and `B => F[C]`. Or can you?