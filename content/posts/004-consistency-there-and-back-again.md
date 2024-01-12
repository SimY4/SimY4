+++
title = "Consistency: There and Back Again"
date = "2021-07-18"
tags = ["fp", "java"]
categories = ["dev-blog"]
+++

## What is Consistency token?
A consistency token represents the state of a resource or entity at a given point in time. It is opaque to consumers, returned in the `ETag` header by data modifying APIs. Consumers may cache some of these tokens in a request-local or short-term cache when coordinating a number of calls to control the consistency they require via a custom `Event-Stream-State` request header(s) with the following format (as per standard RFC definition):

```
Event-Stream-State = ( "At-Least" (token)
                     | "Check-At-Least" (token)
                     )
```

`At-Least` means the underlying data used for the request will be brought up to at least the state denoted by the token. This may incur a delay.

`Check-At-Least` means the state of the underlying data used for the request will be checked against the given token. If it is older than the token, the request will fail.

Multiple tokens can be passed through as per standard HTTP header concatenation such as multiple `Event-Stream-State` headers.

If the consumer does NOT pass an `Event-Stream-State` header, the services will use whatever data they currently have cached (which may be slightly behind the latest persisted data due to the eventual consistency nature of the system).

## Initial consistency API design
Since consistency state was passed around in request headers, we’ve parsed and kept consistency tokens in an implicit threadlocal request context. And because of implicit nature of request context, consistency can be summoned at any point in time into an explicit scope via framework’s utilities. That clears code a lot in places where you don’t care about consistency and allows you to easily access it if you’re calling it from a request scoped thread.

```java
interface InitialConsistencyService {
  Either<Error, Stuff> updateStuff(Stuff stuff);
  
  Either<Error, Optional<Stuff>> fetchStuff(...);
  
  Either<Error, Stuff> readAndWriteStuff(...);
}
```

But in this approach, there’s a major disadvantage - the work with consistency is not transparent. People might not even be aware that consistency is there as a thing to worry about. It’s not the problem in itself until you think of start doing certain things asynchronously. Once you escape the request scoped thread suddenly your consistency state will be lost but you wouldn’t even notice that since everything will continue to work as before.

So we tried to improve that by moving consistency into the light and making work with it explicit.

## Explicit consistency and why it’s bad
By making consistency explicit, we meant to start passing it all the way from a request parsing step to a response generation step. Which means that all of our services now need to become aware of it.

```java
interface ConsistencyAwareService {
  Either<Error, WithConsistency<Stuff>> updateStuff(Stuff stuff);
  
  Either<Error, Optional<Stuff>> fetchStuff(..., Consistency consistency);
  
  Either<Error, WithConsistency<Stuff>> readAndWriteStuff(..., Consistency consistency);
}

@Value
final class WithConsistency<T> {
  public final Consistency consistency;
  public final T value;
  ...
}
```

That solves the problem of consistency being implicit. But suddenly it makes consistency instances passed all over the place! Suddenly the code that was mostly focused on expressing the business relevant logic was bloated with passing, converting and aggregating consistency. It looked like it’s the only thing that code does. Which is not relevant since it’s a technical detail. Here’s the real-life example on how we aggregate consistencies when we process a list of consistency aware operations:

```java
Consistency initial = ...;
listOfStuff.stream()
    .map(stuff -> consistencyAwareService.doReadAndWriteWith(stuff, initial)) // <- this is the only bit here that actually does something other than managing consistency
    .reduce(Either.<Error, WithConsistency<List<Stuff>>>right(WithConsistency.of(new ArrayList<>(), initial)),
        (acc, maybeStuffWithConsistency) -> acc.flatMap(withConsistency -> maybeStuffWithConsistency
            .map(stuffWithConsistency -> { 
                withConsistency.value.add(stuffWithConsistency.value); 
                return WithConsistency.of(withConsistency.value, withConsistency.consistency.append(stuffWithConsistency.consistency));
            })),
        (l, r) -> l.flatMap(lv -> r.map(rv -> { 
            lv.value.addAll(rv.value); 
            return WithConsistency.of(lv, lv.consistency.append(rv.consistency));
        })));
```

We can hide certain things behind some common consistency combinator methods but even so, this code is very hard to read and it’s even more painful to write. It emphasises on the things that are purely technical and makes the thing that actually matters (do stuff) barely visible in the code.

To make things even worse this is not even a correct encoding of the original intent. Write API should issue the new consistencies for every write or fail (in which case you get no consistency) - that's encoded in an `Either<Error, WithConsistency<T>>` type. But what about reads? Reads should revoke consistencies on successful read or return an error with outstanding consistencies that haven't been seen yet. Was that encoded in a type `Either<Error, T>`? No. Consistency tokens are lost on a first unsuccessful read. We can try to correct this by changing a read return type signature to be `Either<WithConsistency<Error>, T>` but that will mean that in a complex method scenario where you read and write with consistency your full return type would be `Either<WithConsistency<Error>, WithConsistency<T>>`.

## FP to the rescue!
One thing I decided to do here was since we’re trying to practice an FP (at least to some extent) let's try to find the answers in FP patterns.

First of all, by looking at the last fully encoded type that I came up with we can clearly see that we can change the order or our type arguments to make it simpler:

```java
Either<WithConsistency<Error>, WithConsistency<T>> readAndWriteStuff(..., Consistency constency)
// equivalent to
WithConsistency<Either<Error, T>> readAndWriteStuff(..., Consistency constency)
```

That simplifies things a little but still, we will face the same problem as before: encoding all of the type transformations. Let’s improve that code further. We can move the `Consistency` argument from the right-hand side to the left and see what that can give us.

```java
WithConsistency<Either<Error, T>> readAndWriteStuff(..., Consistency constency)
// equivalent to
Function<Consistency, WithConsistency<Either<Error, T>>> readAndWriteStuff(...)
```

From the looks of that signature, we can notice that this method becomes lazy. Before that, we return an actual value and now we return a function, calling which with initial consistency state you can get a pair latest evaluated consistency as well as calculated value. We can also notice something similar in the shape of that signature… We take consistency and return consistency back associated with some value. That can be encoded as a State monad.

```java
Function<Consistency, WithConsistency<Either<Error, T>>> readAndWriteStuff(...)
// equivalent to
State<Consistency, Either<Error, T>> readAndWriteStuff(...);
```

And because the state is a monad we can compose lazily evaluated states to create a bigger state transition. That, in the end, is not attached to any thread - you can run that state transition wherever you feel like it. You can pass state evaluator as a value to a different thread as well as initial consistency value.

But still, you'll need two monad transformation calls in order to access the type `T` which is encoded in this type. If only we can have something like a monad transformer similar to those that alternative JVM languages have… Well, let’s write it ourselves, shall we?

## Higher kinds in Java
There’s a [good article about how we can encode higher-kinded types in Java](https://medium.com/@johnmcclean/simulating-higher-kinded-types-in-java-b52a18b72c74). Basically we I'm going to follow that article in order to encode the state monad and either transformer. I'll assume that we have State already implemented because it's not that interesting. State is a type that wraps an arrow `(S) -> (S, A)` with flatMap and pure methods that allow you to compose one state with another. To encode the higher kinds in Java, we'll create an interface with two type parameters:

```java
interface Kind1<WITNESS, A> {}
```

The first argument will encode the witness of our current data type and the other will be the argument of that type. Then for our state monad we’ll add a `StateKind` extending this interface:

```java
final class StateKind<W extends Witness<S>, S, A> extends Kind1<W, A> {
  interface Witness<S> {}
  
  protected final State<S, A> delegate;
  
  private StateKind(State<S, A> delegate) {
    this.delegate = delegate;
  }
  
  /**
   * Transforms state into higher kind.
   */
  public static <W extends Witness<S>, S, A> StateKind<W, S, A> of(State<S, A> state) {
    return new StateKind<>(state);
  }
  
  /**
   * Transforms higher kind into state.
   */
  public static <W extends Witness<S>, S, A> State<S, A> narrow(Kind1<W, A> state) {
    // ugly cast but we assume it's safe because the type witness should only exist for StateKind.
    return ((StateKind<W, S, A>) state).delegate;
  }
}
```

Based on this we can define a state monad instance:

```java
interface Monad<WITNESS> {
  <A> Kind1<WITNESS, A> pure(A a);
  
  <A, B> Kind1<WITNESS, B> flatMap(Kind1<WITNESS, A> kind, Function<? super A, ? extends Kind1<WITNESS, B>> fmap);
  
  default <A, B> Kind1<WITNESS, B> map(Kind1<WITNESS, A> kind, Function<? super A, ? extends B> map) {
    return flatMap(kind, map.andThen(this::pure));
  }
}

final class StateMonad<W extends StateKind.Witness<S>, S> implements Monad<W> {
  public <A> Kind1<W, A> pure(A a) {
    return StateKind.of(State.pure(a));
  }
  
  <A, B> public Kind1<W, B> flatMap(Kind1<W, A> kind, Function<? super A, ? extends Kind1<W, B>> fmap) {
    return StateKind.of(StateKind.narrow(fa).flatMap(fmap.andThen(StateKind::narrow)));
  }
}

interface ConsistencyState extends StateKind.Witness<Consistency> {
  public static final Monad<ConsistencyState> MONAD = new StateMonad<ConsistencyState, Consistency>();
}
```

And having all that we can implement an `Either` transformer for any monad instance:

```java
final class EitherT<WITNESS, L, R> {

  private final Monad<WITNESS> monad;
  private final Kind1<WITNESS, Either<L, R>> kind;
  
  private EitherT(Monad<WITNESS> monad, Kind1<WITNESS, Either<L, R>> kind) {
    this.monad = monad;
    this.kind = kind;
  }
  
  public static <WITNESS, L, R> EitherT<WITNESS, L, R> of(Monad<WITNESS> monad, Kind1<WITNESS, Either<L, R>> kind) {
    return new EitherT<>(monad, kind);
  }
  
  public <B> EitherT<WITNESS, L, B> flatMap(Function<? super R, ? extends EitherT<WITNESS, L, B>> fmap) {
    return new EitherT<>(monad, monad.flatMap(kind, either -> either.fold(
      l -> monad.pure(Either.left(l)),
      r -> fmap.apply(r).kind
    )));
  }
  
  ...
}
```

Having this we can replace our types with `EitherT<ConsistencyState, Error, T>` and we can work with as if we would work with an `Either` type directly! But of course, it will do the same dual monad transformation under the curtains. But we don’t care since now it’s hidden from us. So the final Consistency API will look like this:

```java
interface FpThroughTheRoofService {
  EitherT<ConsistencyState, Error, Stuff> updateStuff(Stuff stuff);
  
  EitherT<ConsistencyState, Error, Optional<Stuff>> fetchStuff(...);
  
  EitherT<ConsistencyState, Error, Stuff> readAndWriteStuff(...);
}
```

So we returned back to where we started. Our API operates the same way as before - consistency is not part of method parameters list but it's encoded within the method's return argument. Which means that we can easily distinguish methods that can change the consistency state and from ones that don't care about consistency. And using the power of `EitherT` type we can easily mix these APIs together.

## Conclusions
Of course, I would not expect anyone to use this practice anywhere in their production code. That was an interesting exploratory work that gave me an idea on how we can achieve pretty much the similar effect in Java by decreasing the level of abstraction a little and throwing a few OO design patterns into the mix. But that was fun. I enjoyed doing it and I hope you enjoyed reading this. 😀

Fin

