+++
title = "Cross-Compiling Scala in Gradle project"
date = "2019-05-20"
tags = ["gradle", "scala"]
categories = ["dev-blog"]
+++

It is quite common and widely spread practice for Scala projects to cross-compile and publish several artifacts for multiple versions of Scala compiler. As a rule, for the purposes of creating several versions of one artifact teams using [SBT](https://www.scala-sbt.org/) where this feature is avaiable right out of the box and can be configured in a couple of lines. But what if we want to cross-publish our Scala project the same way without using SBT?

For one of my Java projects, I decided to create a Scala bridge. Historically, the entire project is built using [Gradle build tool](https://gradle.org/) and it was decided to add the bridge to the same project as a separate submodule. Gradle provides basic Scala compilation support. It can build an artifact and bundle it with documentation and sources for any specific version of Scala compiler - everything you need, except cross-compilation. There is an open [ticket](https://github.com/gradle/gradle/issues/3530) and a couple of plugins ([1](https://github.com/ADTRAN/gradle-scala-multiversion-plugin), [2](https://github.com/prokod/gradle-crossbuild-scala)) focused on adding this support but current state of things is nowhere near to ease of use and convenience of SBT. So I decided to check for myself how hard it is to configure Gradle for Scala cross-compilation without special plugins or Gradle support.

## Wonder

What I want is the same set of sources to be compiled for multiple versions of the Scala compiler: 2.11, 2.12 and 2.13. And since Scala 2.13 has a bunch of backward-incompatible changes to built-in collections, I would like to be able to add additional version-specific source sets for compiler-specific code. Again, it's a trivial task in SBT, let's see what we can do in Gradle:

{{<figure src="/images/posts/cross-compiling-scala-in-gradle-1.png" alt="Initial Project Structure" width="600">}}

The first difficulty to face is that the compiler version is calculated from the declared scala-library dependency version. In addition to that, all dependencies that are prefixed with the Scala compiler version in their GAV coordinates also need to be changed. And then, the set of compiler flags also differs from one compiler version to another: some flags have been renamed, while others have simply been deprecated or even removed entirely. I decided that trying to capture all the nuances of different compilers in one build file seems like a painfully difficult task and even more difficult to maintain it. Therefore, I decided to explore possible other ways to solve this problem. What if we create multiple configuration builds for the same project directory structure?

## Explore

In the declaration of submodules in the Gradle project, you can specify the directory where the root of the submodule and the name of the file responsible for its configuration will be located. Let's specify the same directory for multiple imports and create multiple copies of the build script for each compiler version:

{{<details "settings.gradle" >}}
```groovy
rootProject.name = 'test'
include 'java-library'

include 'scala-facade_2.11'
project(':scala-facade_2.11').with {
  projectDir = file('scala-facade')
  buildFileName = 'build-2.11.gradle'
}

include 'scala-facade_2.12'
project(':scala-facade_2.12').with {
  projectDir = file('scala-facade')
  buildFileName = 'build-2.12.gradle'
}

include 'scala-facade_2.13'
project(':scala-facade_2.13').with {
  projectDir = file('scala-facade')
  buildFileName = 'build-2.13.gradle'
}
```
{{</details>}}

Not bad, but occasionally we can get strange compilation errors due to the fact that all three build scripts use the same build directory. We can fix this by setting them ourselves for each of the builds:

{{<details "build-2.12.gradle" >}}
```groovy
plugins {
  id 'scala'
}

buildDir = 'build-2.12'

clean {
  delete 'build-2.12'
}

// ...
```
{{</details>}}

Now, it's shaping up nicely. The only problem is that such a build will drive your favorite IDE crazy and most likely you will have to do further editing completely goalless. I thought that this is not a big problem, because. you can always comment out extra imports of submodules and turn your cross build into a regular build that your IDE most likely can work with.

What about additional source sets? Again, with separate files, this turned out to be quite simple, create a new directory and configure it as a source set:

{{<details "build-2.12.gradle" >}}
```groovy
sourceSets {
  compat {
    scala {
      srcDir 'src/main/scala-2.12-'
    }
  }
  main {
    scala {
      compileClasspath += compat.output
    }
  }
  test {
    scala {
      compileClasspath += compat.output
      runtimeClasspath += compat.output
    }
  }
}
```
{{</details>}}

{{<details "build-2.13.gradle" >}}
```groovy
sourceSets {
  compat {
    scala {
      srcDir 'src/main/scala-2.13+'
    }
  }
  main {
    scala {
      compileClasspath += compat.output
    }
  }
  test {
    scala {
      compileClasspath += compat.output
      runtimeClasspath += compat.output
    }
  }
}
```
{{</details>}}

The final structure looks like this:

{{<figure src="/images/posts/cross-compiling-scala-in-gradle-2.png" alt="Final Project Structure" width="600">}}

Here you can also extract individual common configuration pieces in external configuration files and import them into the build in order to reduce the number of repetitions. But for me it turned out well anyway, declarative, isolated and compatible with all possible Gradle plugins.

## Outcome

The problem was solved, Gradle's flexibility was enough to quite elegantly express a very non-trivial setup, and Scala cross-building was proven to be possible not only using SBT, and if for one reason or another you use Gradle to build a Scala project, cross-compilation as an opportunity for you is also available.

[Project GitHub link](https://github.com/SimY4/xpath-to-xml/tree/2.x/xpath-to-xml-scala) if you want to reproduce this explore yourself.