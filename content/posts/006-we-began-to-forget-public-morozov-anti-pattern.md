+++
title = "We Began to Forget: Public Morozov Anti-Pattern"
date = "2026-01-05"
tags = ["java", "we-began-to-forget"]
categories = ["dev-blog"]
+++

I decided to start new series of blogs called “We Began to Forget”. The idea is simple and a little melancholic.
Software engineering has a short memory, patterns get discovered and then forgotten, lessons get relearned.
These posts are small memory refreshers about important aspects of our craft with a strong focus on clever hacks
rather than things you can read in clever books. Books will make sure the good patterns will live on,
but tribal knowledge has to be passed by word of mouth.

## Encapsulation: a 100$ mistake

In the Java ecosystem, shared modules are everywhere. Platform libraries, internal frameworks, shaded dependencies, 
company-wide “commons” jars. They expose a public API, carefully curated, documented, versioned. And then there is 
the *other stuff*. Private classes, package-private helpers, internal methods that do exactly what you need, perfectly,
efficiently, already tested. They sit there like a locked door with light shining underneath.

You want to use them. You are not supposed to.

{{<figure src="/images/posts/rule-of-cool.png" alt="Rule of Cool" width="600">}}

Before reflection, before bridges, before Public Morozov even enters the room, module authors usually do a lot of work 
to say one thing very clearly: *this is not for you*. I can list three approaches that have become especially common in 
the Java ecosystem as an attempt at encapsulation. Each one leaks in its own way. Depending on the authors library 
choices you may end up in three situations:

### The “internal” package convention

This is the most polite form of refusal.

The module exposes its public API in clean, well-named packages, and then quietly tucks everything else away under 
something like:

```java
import com.example.internal.*;
import com.example.impl.*;
```

The contract here is social, not technical. The classes may be `public`, fully accessible, and trivial to import. 
But the package name itself carries a warning label. This works surprisingly well for disciplined teams and disastrously
well for desperate ones. The JVM does not care about naming conventions. Your IDE will happily autocomplete these classes.
And once production code depends on them, the convention stops being a boundary and starts being folklore.

### Package-private by consolidation

The second approach is more structural. Instead of splitting public and internal code into separate packages, 
everything lives together. Public types are `public`. Internal helpers are package-private, protected by Java’s default
visibility. This is stronger than naming conventions. You cannot access these classes without being in the same package.
Reflection aside, the compiler enforces the rule.

The downside is architectural. Large packages become crowded, internal details sit next to APIs, and refactoring gets
harder over time. It also assumes that “same package” is a meaningful trust boundary, blocks casual misuse and keeps
most consumers honest.

### Visibility and constructors lockdown

This is the hard mode. Here, module authors go all in on visibility modifiers. Classes are final. Constructors are
`private`. Factory methods are tightly controlled. Subclassing is impossible. Instantiation paths are intentionally
obscure. Even if you can see the class, you cannot use it in any normal way. This style is often found in
performance-critical code, low-level frameworks, and libraries that have been burned before. It actively resists
extension, mocking, testing, and reuse. It is defensive programming taken to its logical extreme.

For consumers, this is the hardest barrier to cross. Reflection becomes awkward, method handles get involved, and
sometimes even those fail due to JVM checks or module boundaries. This is the point where “just use reflection” stops
being a casual suggestion and starts becoming an engineering project.

-----

Each of these techniques is an attempt to freeze an internal shape while the public API evolves. And each of them, 
sooner or later, meets a developer who really wants to use what’s inside.

That is where the hero of today’s blog is born.

## Enter the Public Morozov pattern

Let’s stitch Public Morozov back into the picture and see how it behaves when it collides with each of these
encapsulation strategies. Think of this less as a recipe and more as a field guide to how boundaries actually fail in 
the wild.

### Case 1: The “internal” package convention

Here, Public Morozov barely deserves the name. The classes are public. The JVM offers no resistance. The only barrier 
is etiquette. In this case, you often do not need reflection, classloader tricks, or clever indirection. A simple
facade is enough. You import the internal class, wrap it, and re-expose only what you need through a clearly named
access point. The Morozov class becomes a signpost rather than a crowbar. It says: *yes, we know this is internal;
yes, we are doing it anyway*.

The value here is not access, but containment. When the internal API breaks, you know exactly where to look.

### Case 2: Package-private internals

This is where Public Morozov becomes more architectural. Package-private access blocks you unless you stand in exactly
the right place. The trick is to stand there on purpose.

You place your Morozov class in the **same package** as the module’s internal classes. From the compiler’s point of
view, you are now family. From the rest of your codebase, you remain an outsider with a single, public handshake.
At that point, you have options:

1. **Delegation**, where the Morozov class holds an instance and forwards calls
1. **Extension**, where it subclasses and selectively exposes behaviour

All these approaches are explicit violations of the intended design. And all of them are far safer than spreading
reflection hacks across multiple call sites. Public Morozov becomes a membrane. Internals on one side. A stable, public
surface on the other.

### Case 3: Constructors and visibility warfare

This is the realm of hard resistance. Final classes. Private constructors. No subclassing. No factories. Nothing to
hook into. In this world, Public Morozov can no longer rely on placement alone. Reflection becomes unavoidable. You
crack constructors open, invoke private methods, and sometimes bypass initialisation logic entirely. The Morozov class
now hides not just intent, but machinery. Method handles, suppressed warnings, and carefully staged access live here.

Some teams go further, though these techniques have limited reach and sharp edges:

1. **Class shadowing**, where a class with the same name appears earlier on the classpath
1. **Classloader delegation tricks**, altering which definition wins
1. **Bytecode weaving**, rewriting classes at load time to relax visibility modifiers

These approaches work only under specific conditions and often collapse under JPMS, security managers, or future JVM
changes. When they fail, they fail spectacularly. In this case, Public Morozov becomes less of a facade and more of a
containment vault. It isolates the most fragile, least portable code in one place so the rest of the system can pretend
it lives in a saner universe.

-----

Across all three cases, the pattern remains the same. Public Morozov does not make forbidden access safe. It makes it 
**legible**. It turns an invisible dependency into a named one. It gives future maintainers a map and a warning sign
instead of a trapdoor.

And that, sometimes, is the best kind of damage control we have left.

We began to forget that not every clever solution is a good one. This series exists to help us remember.
