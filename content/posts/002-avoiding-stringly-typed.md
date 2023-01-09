+++
title = "Avoiding Stringly Typed"
date = "2022-02-24"
tags = ["java"]
categories = ["dev-blog"]
+++

There’s this saying that you shouldn’t have String as your function input unless you expect three volumes of War and Peace as valid input.
So unless it’s your case and you want to improve your codebase to be more precise in type signatures, I invite you to discuss one possible option. I’ll use the following API as our case study:

```java
interface EmailSender {
  void sendEmail(String from, String to, String subject, String body);
}
```

there’s an obvious ambiguity in the type signature here. All of the arguments are of the same type and can be easily misplaced.
Not to mention that any string is accepted as a valid input that, as it seems, suppose to only be a valid email.

In relation to the first problem, languages like Scala and Kotlin let you choose to pass arguments by name which improves the situation a little bit.
Depending on how stable your API contracts are you may choose to rely on argument names.
But with one downside of whenever you have to interop with Java the ability to pass arguments by name disappear because compiler artifacts produced by javac don't preserve argument names by default.

Ok, so what about the second problem?

{{< notice tip >}} You can enablejavac to store method argument names by passing the -parameters compiler flag. Scala can honour the additional info and will let you use named arguments. Kotlin unfortunately ignores this metadata and won’t let you do it either way. {{< /notice >}}

## Refined types

Though some other languages have nicer ways to define type refinements, the most straightforward way to narrow down the set of acceptable incoming arguments in Java is by introducing a value-based class.
The idea is quite simple: you can take the tainted value and lift it into a class instance that can only carry valid inputs or produce an error with details about why this refinement fails. Here’s what refined email type may look like:

```java
final class Email {
  private String email;

  // conventional method name
  public static Email valueOf(String tainted) {
    if (!tainted.matches(".+@.+")) {
      throw new IllegalArgumentException("Malformed email: " + tainted);
    }
    return new Email(tainted);
  }

  private Email(String email) {
    this.email = email;
  }

  @Override
  public String toString() {
    return email;
  }
}

interface EmailSender {
  void sendEmail(Email from, Email to, String subject, String body);
}
```

This won’t make the problem suddenly go away but it will push it further out to the boundaries of your system where all these constraints were supposed to be checked anyways which is a great plus.
Suddenly there’s no way you can accidentally carry incorrect data across the system.
You can get rid of the unnecessary validation steps at arbitrary points in your system just to make sure that you’re passing the arguments in the right format.
Which will in turn simplify the number of testing scenarios you need to maintain - there’s no way you can pass an incorrect email to the function that accepts refined email type.

But there are some downsides. Let’s go through them one by one and see what we can do about them.

### Unnecessary boxing

I’m going to discuss this really quickly because this is the first thing people usually talk about when considering using these patterns in practice.
Though it might not always be possible for the JIT to optimize away the unnecessarily boxed String instance, I doubt that many of us are solving problems that are going to have type boxing as a bottleneck in their system.
For those who do: we’re all hoping for project Valhalla to bring us zero-cost type refinements really really soon.

### Packing and Unpacking hassle

For some problems, it could be a hassle to pack and unpack your types so you can use your refined types with APIs from the standard library. Consider two options of defining a complex entity factory method that will let you create it out of smaller chunks of refined data:

```java
final class Local { /* refined email local part. Usually case-sensitive */ }
final class Domain { /* refined email domain. Usually case-insensitive */ }

final class Email {
  // option A
  public static Email from(Local local, Domain domain) {
    return new Email(local.toString() + "@" + domain.toString());
  }

  // option B
  public static Email from(String local, String domain) {
    return new Email(Local.valueOf(local) + "@" + Domain.valueOf(domain));
  }

  //...
}
```

Regardless of what you chose you’ll unavoidably force more `.valueOf(String)` and `.toString()` method usages across all of your codebases.
This is not a great thing because these methods aren’t carrying any useful implementation detail - they are just making types align with each other.
Plus it won’t cover cases where you already have the value of some refined components and you just need to add another one.

The solution to this problem is to find a good common ground between your refinements and the rest of the APIs that might not want to think in terms of refinements.
Sometimes it can be achieved by providing more implementation to standard library interfaces.
Some great candidates that eliminate quite a lot of reasons for unpacking are `java.lang.Number` interface if you’re refining a numeric type, `java.lang.CharSequence` if you’re refining string type, `java.lang.Comparable` if your refinement has universal comparison rules.
Having those implementations will eliminate a decent chunk of packing and unpacking reasons and make your codebase focused on actual argument passing and calling APIs directly on them.

```java
final class Local implements CharSequence, Comparable<Local> { ... }
final class Domain implements CharSequence, Comparable<Domain> { ... }
final class Email implements CharSequence, Comparable<Email> {
  public static Email from(CharSequence local, CharSequence domain) {
    if (!(local instanceof Local)) Local.valueOf(local);
    if (!(domain instanceof Domain)) Domain.valueOf(domain);
    return new Email(local + "@" + domain);
  }
}
```

You’ll be surprised how many standard java APIs are willing to work on `CharSequence` and not on `String` directly. And since most of the type refinements are implemented by delegating to underlying string instance, implementing all these contracts is very trivial.

## In conclusion

Some of these proposed steps were retrofitted into the principles on which principled-ari-java library in one of the recent library releases. Totally worth the effort. If you were delaying introducing refined types in your codebase for one of the reasons above, I hope I was able to give you more reasons to reconsider. If there are still obstacles in adopting refines in your codebase that I didn’t think of, please let me know, I’ll think of something to make to work for your case also.