# Gradle Checker Framework Plugin

[![License](https://img.shields.io/badge/license-apache%202.0-blue.svg)](http://www.apache.org/licenses/LICENSE-2.0)
[![Build Status](https://travis-ci.com/kelloggm/checkerframework-gradle-plugin.svg?branch=master)](https://travis-ci.com/kelloggm/checkerframework-gradle-plugin)

This plugin configures `JavaCompile` tasks to use the [Checker Framework](https://checkerframework.org) for pluggable type-checking.

## Download

Add the following to your `build.gradle` file:

```groovy
plugins {
    // Checker Framework pluggable type-checking
    id 'org.checkerframework' version '0.3.3'
}

apply plugin: 'org.checkerframework'
```

## Configuration

### Configuring which checkers to use

The `checkerFramework.checkers` property lists which checkers will be run.

For example:

```groovy
checkerFramework {
  checkers = [
    'org.checkerframework.checker.nullness.NullnessChecker',
    'org.checkerframework.checker.units.UnitsChecker'
  ]
}
```

For a list of checkers, see the [Checker Framework Manual](https://checkerframework.org/manual/#introduction).

### Providing checker-specific options to the compiler

You can set the `checkerFramework.extraJavacArgs` property in order to pass additional options to the compiler when running
a typechecker.

For example, to use a stub file:

```groovy
checkerFramework {
  extraJavacArgs = [
    '-Astubs=/path/to/my/stub/file.astub'
  ]
}
```

### Configuring third-party checkers

To use a third-party typechecker (i.e. one that is not distributed with the Checker Framework),
add a dependency to the `checkerFramework` dependency configuration.

For example, to use the [Glacier](http://mcoblenz.github.io/Glacier/) immutability checker:

```groovy
dependencies {
  ...
  checkerFramework 'edu.cmu.cs.glacier:glacier:0.1'
}
```

### Specifying a Checker Framework version

This plugin uses Checker Framework version 2.8.2 by default.
Anytime you upgrade to a newer version of this plugin,
it might use a different version of the Checker Framework.

If you wish to use a specific Checker Framework
[version](https://github.com/typetools/checker-framework/releases),
add text like the following to `build.gradle`, after `apply plugin: 'org.checkerframework'`:

```groovy
dependencies {
  compile 'org.checkerframework:checker-qual:2.8.0'
  checkerFramework 'org.checkerframework:checker:2.8.0'
  checkerFrameworkAnnotatedJDK 'org.checkerframework:jdk8:2.8.0'
}
```

You can also use a locally-built version of the Checker Framework:

```groovy
def cfHome = String.valueOf(System.getenv("CHECKERFRAMEWORK"))
dependencies {
  compile files(cfHome + "/checker/dist/checker.jar")
  checkerFramework files(cfHome + "/checker/dist/checker-qual.jar")
  checkerFrameworkAnnotatedJDK files(cfHome + "/checker/dist/jdk8.jar")
}
```


### Other options

By default, the plugin applies the selected checkers to all `JavaCompile` targets.
The plugin includes a rudimentary option for preventing checkers from being applied
to test targets. To use it, add the following to the `checkerFramework` block:

```groovy
checkerFramework {
  excludeTests = true
}
```

The check for test targets is entirely syntactic: this option will not apply the checkers
to any task whose name includes "test", ignoring case. The default value is `false`.

You can also choose different versions of the annotations for the source and
test compilation targets. To do so, do not use the `compile` dependency
configuration. Instead, use the `compileOnly` configuration for source
code, and the `testCompile` configuration for test code. For example, to
use version `2.8.1` of `checker-qual` in the source code, but version `2.8.2`
in the test code, use these dependencies:

```groovy
dependencies {
  compileOnly 'org.checkerframework:checker-qual:2.8.1'
  testCompile 'org.checkerframework:checker-qual:2.8.2'
}
```

### Incompatibility with Error Prone

[Error Prone](https://errorprone.info/)
uses the Checker Framework's dataflow analysis library.
Unfortunately, Error Prone uses an old version of the library, so you
cannot use both Error Prone and the current Checker Framework (because each
one depends on a different version of the library).

You can resolve this via a switch that causes your build to use either
Error Prone or the Checker Framework, but not both.
Here is how do change the above instructions:


```
plugins {
  id "net.ltgt.errorprone-base" version "0.0.16" apply false
  // To do Checker Framework pluggable type-checking (and disable Error Prone), run:
  // ./gradlew compileJava -PuseCheckerFramework=true
  id 'org.checkerframework' version '0.3.3' apply false
}

if (!project.hasProperty("useCheckerFramework")) {
    ext.useCheckerFramework = "false"
}
if ("true".equals(project.ext.useCheckerFramework)) {
  apply plugin: 'org.checkerframework'
} else {
  apply plugin: 'net.ltgt.errorprone-base'
}


def checkerFrameworkVersion = "2.8.2"

dependencies {
  if ("true".equals(project.ext.useCheckerFramework)) {
    checkerFramework 'org.checkerframework:checker:' + checkerFrameworkVersion
    checkerFramework 'org.checkerframework:jdk8:' + checkerFrameworkVersion
    checkerFramework 'org.checkerframework:checker-qual:' + checkerFrameworkVersion
  } else {
    errorprone group: 'com.google.errorprone', name: 'error_prone_core', version: '2.3.3'
  }
}

if ("true".equals(project.ext.useCheckerFramework)) {
  checkerFramework {
    checkers = [
      'org.checkerframework.checker.interning.InterningChecker',
      'org.checkerframework.checker.signature.SignatureChecker'
    ]
  }
} else {
  // Configuration for the Error Prone linter.
  tasks.withType(JavaCompile).each { t ->
    if (!t.name.equals("compileTestInputJava") && !t.name.startsWith("checkTypes")) {
      t.toolChain ErrorProneToolChain.create(project)
      t.options.compilerArgs += [
        '-Xep:StringSplitter:OFF',
        '-Xep:ReferenceEquality:OFF' // use Interning Checker instead
      ]
    }
  }
}
```


## Using a locally-built plugin

You can build the plugin locally rather than downloading it from Maven Central.

To build the plugin from source, run `./gradlew build`.

If you want to use a locally-built version of the plugin, you can publish the plugin to your
local Maven repository by running `./gradlew publish`. In the `build.gradle` file for each
project for which you want to use the locally-built plugin, make sure that `mavenLocal()`
is the first entry in the `repositories` block within the `buildscript` block. A full example
will look like this:

```groovy
buildscript {
  repositories {
    mavenLocal()
  }

  dependencies {
    classpath 'gradle.plugin.org.checkerframework:checkerframework-gradle-plugin:0.3.3-SNAPSHOT'
  }
}

apply plugin: 'org.checkerframework'
```

## Credits

This project started as a fork of [a plugin built by jaredsburrows](https://github.com/jaredsburrows/gradle-checker-framework-plugin).
[![Twitter Follow](https://img.shields.io/twitter/follow/jaredsburrows.svg?style=social)](https://twitter.com/jaredsburrows)


## License

    Copyright (C) 2017 Jared Burrows, 2019 Martin Kellogg

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
