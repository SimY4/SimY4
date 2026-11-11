+++
title = "We Began to Forget: Demystifying Gradle"
date = "2026-10-11"
tags = ["java", "we-began-to-forget"]
categories = ["dev-blog"]
+++

We began to forget that a build is just a function from files to files.

Make, 1976: a target, its prerequisites, a recipe, and a timestamp comparison. That's the whole idea.

```
app.jar: Main.class
	jar cf app.jar Main.class
Main.class: Main.java
	javac Main.java
```

Every JVM build tool since then is a remix of it: Ant, Maven, Gradle. Gradle is make with a few upgrades: 
content hashes instead of timestamps and a typed lazy model instead of strings.

{{<notice info>}} **Disclaimer**: This is my mental model of Gradle, not the official docs. Some of it is simplified,
and some details may be plain wrong. But it has predicted Gradle's behaviour well enough that I rarely get surprised anymore.
When I do, it's usually because I broke one of the rules at the end of this post. {{</notice>}}

## Primitives of Gradle

Open the Gradle documentation and you'll find projects, plugins, extensions, source sets, configurations, variants,
conventions, artifact transforms and a few dozen more concepts, each with its own page. It's easy to walk away thinking
Gradle is complicated. My mental model of it fits into one picture, built from only three primitives: 
**Tasks**, **Properties** and **FileCollections**. It took me years to draw, so let me draw it slowly, one shape at a time.

{{<mermaid>}}
flowchart LR
  subgraph Task
    Action[Action]
  end
{{</mermaid>}}

Remember the recipe line in the Makefile, the `javac Main.java`? That's an Action. In Gradle it might be a compiler
invocation, a `JavaExec` running a java program, or a `doLast {}` closure doing arbitrary other stuff. This is where
the actual work happens, and it's the only part of the picture Gradle never looks inside. Gradle doesn't parse your
Action, doesn't analyse it, and can't tell which files it reads or writes. Gradle doesn't know what your Action does.

The Action is also the only part of the picture that runs at _execution_ time. Everything else is put together earlier,
at _configuration_ time, while Gradle evaluates your build scripts. This is the first thing we began to forget: the build
script is not the build.  It's a program that draws the picture, and Gradle starts running Actions only once the picture
is complete. Mixing up the two phases gives you a whole genre of bugs, like this one from a real build script:

```groovy
tasks.register('doStuff', JavaExec) {
  standardOutput = new FileOutputStream(rootProject.file('my-file.txt'))
  // ...
}
```

The `FileOutputStream` is created when the task is configured, not when it runs. So anything that only configures the task,
like listing tasks or an IDE sync, empties the file.

### Properties

{{<mermaid>}}
flowchart LR
  subgraph Task
    direction TB
    Properties@{ shape: notch-rect, label: "Properties" }
    Action[Action]
    Properties --> Action
  end
{{</mermaid>}}

An Action rarely works alone. It needs to know which main class to run, which arguments to pass, which Java release to
target, or how many test forks to spawn. These are Properties, and they provide a way to configure the tasks behaviour.
They sit inside the box because they belong to this task and nobody else.

What makes Properties interesting is that they're lazy. A `Property<T>` is a box that gets filled during configuration
and opened by the Action during execution. At the moment your line of build script runs, the final value may not be known
yet, because a plugin applied later, a convention or another script can still change it. A Property lets you wire values
together now and read them later, and `.map {}` lets you derive one value from another without opening the box.
Calling `.get()` at configuration time opens it too early, and that's the most common way to quietly break laziness.

Properties don't only live in tasks. When you write

```java
java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}
```

you're setting a Property on an extension, and the Java plugin wires it into the Properties of every compile, test and
javadoc task. Strip away the conventions, and most of what a plugin does is exactly this: register extensions, register
tasks, and connect their Properties.

Keep an eye on the arrow from Properties to the Action. Anything that flows into the Action can change what it produces,
so Gradle will want to know about it. We'll come back to this arrow when we get to the up-to-date checker.

### FileCollections

{{<mermaid>}}
flowchart LR
  subgraph Task
    direction TB
    Properties@{ shape: notch-rect, label: "Properties" }
    Action[Action]
    Properties --> Action
  end
  Input@{ shape: docs, label: "**Input**\nFileCollection"}
  Output@{ shape: docs, label: "**Output**\nFileCollection"}
  Input --> Task
  Task --> Output
{{</mermaid>}}

Now the picture is complete, and its most important detail is where Input and Output sit: outside the Task.
Properties are what a task _is_. FileCollections are what a task _exchanges_ with the world.

This isn't only a drawing choice. Gradle has no notion of a task returning a value. Every kind of output a task can
declare is a file. A task can take both Properties and files as inputs, but it can only produce files. There's no
`return` in Gradle: if you want to hand something to the next task, you write it to a file.

A FileCollection isn't a `List<File>` either. It's a lazy description of files: everything under `src/main/java`,
whatever the `jar` task produces, the runtime classpath. The dependencies you declare end up as FileCollections too.
`compileClasspath` and `runtimeClasspath` are configurations, and every configuration is a FileCollection.
Gradle resolves it as late as possible. More importantly, a FileCollection remembers who produces the files it describes.
Every FileCollection can tell you which tasks have to run before its files exist.

## Building the Graph

{{<mermaid>}}
flowchart LR
  subgraph CompileJava
    direction TB
    CompileProperties@{ shape: notch-rect, label: "CompileJava Task Properties" }
    CompileAction[javac]
    CompileProperties --> CompileAction
  end
  subgraph Jar
    direction TB
    JarProperties@{ shape: notch-rect, label: "Jar Task Properties" }
    JarAction[jar]
    JarProperties --> JarAction
  end
  
  Sources@{ shape: docs, label: "Sources"}
  Classes@{ shape: docs, label: "Classes"}
  JarFile@{ shape: docs, label: "JarFile"}
  Sources --> CompileJava
  CompileJava --> Classes
  Classes --> Jar
  Jar --> JarFile
{{</mermaid>}}

Here are two of them side by side: compileJava turns sources into Java classes, and jar turns classes into `jar` file.
Now look for the arrow between the two tasks. There isn't one. Nothing in this picture says "`jar` depends on `compileJava`".
The edge between them _is_ the classes FileCollection, because the Output of one box and the Input of the other are the same thing.

So the edges are made of files. A Property can carry an edge only when its value is derived from another task's Output,
and that Output, as we've seen, is always a file.

{{<notice tip>}} Tasks that has no direct dependency on each other can be executed concurrently.

`./gradlew build --dry-run` (`-m`) prints the plan without running it.{{</notice>}}

### Drawing an Edge

Gradle gives you two ways to draw an edge. The first is explicit, and you've certainly seen it before:

```groovy
check.dependsOn integrationTest
```

`dependsOn` adds `integrationTest` to the graph and makes it run first. Its relatives only arrange the order:
`mustRunAfter` and `shouldRunAfter` say "if both of these tasks happen to be in the graph, run them in this order",
and `finalizedBy` attaches a task that runs after another one, even when that one fails.

The second way is implicit, and it's the one the picture is about. You don't describe the edge at all. You wire one
task's Output into another task's Input and let Gradle read the edge off the FileCollection. When Gradle schedules `jar`,
it asks each of its Inputs which tasks produce their files. The sources FileCollection answers `compileJava`, and the edge appears.
Every other task that reads the main source set, like `sourcesJar` or `javadoc`, gets the same answer,
and you didn't have to list a single one of them.

## Did Anything Change?

Before running a task, Gradle fingerprints every arrow that goes into the Action. For Properties, it hashes their values.
For Input FileCollections, it hashes the contents of the files along with their normalised paths. It still never looks
inside the Action, but it does remember which Action it was. After the Action finishes, Gradle takes a snapshot of the
Outputs and stores the snapshot and the fingerprints together in the execution history, under the project's `.gradle` directory.

On the next build, Gradle compares. If every Input fingerprint matches, and the Outputs still look exactly as they did
when the task finished, the task is `UP-TO-DATE` and the Action doesn't run. If anything differs, the task runs again.
This is make with hashes instead of a clock: touching a file doesn't make it new, only changing what's in it does.

### The Rules

Everything below comes from one idea: the checker can only see what you put in the picture. Four rules follow from it.

- **Declare every Output.** A task without Outputs is never up-to-date, because Gradle has nothing to compare.
  It also has nothing to share, so, nobody can draw an edge to it without `dependsOn`.
- **Declare every Input.** This is the dangerous one. A task that reads a file its Action knows about but its picture
  doesn't will happily stay `UP-TO-DATE` after that file changes. Breaking the first rule makes builds slow. Breaking this one makes them wrong.
- **Keep volatile values out of Inputs.** Timestamps, hostnames, random seeds and environment variables all change the
  fingerprint without changing anything that matters.
- **One Output, one owner.** If another task changes your Outputs after you've finished, the snapshot Gradle took no
  longer matches, and your task runs again on every single build. Gradle also refuses to cache a task whose Outputs
  overlap with another task's, because it can no longer tell whose files are whose. Which brings me to my favourite bug.
 
{{<notice tip>}} When it still doesn’t work: run your build with `./gradlew build --info -Dorg.gradle.caching.debug=true` to learn why

And when you just need the Action to run regardless of up to date checker state, `--rerun` forces a single task and `--rerun-tasks` forces all of them.{{</notice>}}

## Fin

Under the DSL, the plugins, the conventions, the version catalogs and the configuration cache, Gradle is still doing what make
did in 1976. It draws a graph of tasks, connects them with files, and before running anything it asks the same old question:
did anything change? It just learned to answer with hashes instead of a clock.

So when a task runs for no reason, or doesn't run when it should, it helps to go back to the picture. 
Somewhere in it there's an arrow you forgot to draw, or a FileCollection with two owners.

We began to forget that a build is just a graph of files. This series exists to help us remember.
