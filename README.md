# Gradle vs Bazel performance reproduction instructions

## Install the Gradle Profiler

This page explains how to reproduce the [Gradle vs Bazel](https://blog.gradle.org/gradle-vs-bazel-jvm) performance numbers yourself. For that, you need to install the [Gradle profiler](https://github.com/gradle/gradle-profiler), a tool which
will automate benchmarking by running several builds in a row, with the appropriate options (it also runs Bazel builds). For non-incremental scenarios, you might want 
alternatively want to use a tool like [hyperfine](https://github.com/sharkdp/hyperfine).  

## Generate the test projects

Our performance comparison uses 4 test projects:

- a large, monolithic app
   - single project
   - 50000 source files and 50000 test source files
- a small sized, multi-project build
   - 10 subprojects
   - each subproject has 100 source files and 100 test source files
- a medium sized, multi-project build
   - 100 subprojects
   - each subproject has 100 source files and 100 test source files
- a large, multi-project build
   - 500 subprojects
   - each subproject has 100 source files and 100 test source files

They can be found in the [Gradle repository](https://github.com/gradle/gradle). To generate a test project, checkout
the Gradle sources, then execute:

- `./gradlew largeMonolithicJavaProject` for the large monolithic build
- `./gradlew mediumJavaMultiProject` for the medium multiproject build
- `./gradlew smallJavaMultiProject` for the small multiproject build
- `./gradlew largeJavaMultiProject` for the large multiproject build

The test projects will then be found in `subprojects/performance/build/<test project name>`.

## Setup

All benchmarks are run with a similar setup. Beware that due to tool differences, some scenarios may not be perfectly
comparable. We try to align the scenarios as much as possible to keep them related.

### Warmup
Given all build tools are implemented in Java, runtime performance can vary greatly depending on class caches, JVM load times and JIT compiling.
To reduce the noise of that and to have more stable measurements, all benchmarks have been run with
* 10 warmup rounds (not measure)
* 10 measurement rounds  

### Parallelism
For all benchmarks, we used a machine with 8 cores and configured the tools accordingly
* Gradle: org.gradle.workers.max=8 (e.g. in `gradle.properties`)
* Maven: `-T 8`
* Bazel: `--jobs 8`

### Remote Cache
For the benchmarks involving remote caches, we choose the following, all residing in the same data center

* Gradle: Gradle Enterprise 2020.2 (Frankfurt)
* Maven:  Gradle Enterprise 2020.2 (Frankfurt)
* Bazel: Google Cloud Storage Bucket (`europe-west3` Frankfurt)

### Instant Execution/File watching
Gradle builds should be configured to use the newly available feature in Gradle 6.4 (officially released in 6.5) using
```
org.gradle.unsafe.instant-execution=true
org.gradle.unsafe.vfs.retention=true
```

## Different scenarios
* Full Build
    * No prior project state (no output files, no caches, usually the result of runing the tool-specific `clean`)
    * Uses warm daemons (Gradle/Bazel) due to warmup rounds
    * Needs to ensure to have a clean state before each measurement round (e.g. `clean` or `clear-project-cache-before` in the scenarios file)
    * Includes compile (`*.class`), assemble (`*.jar`) and running tests
    * Measure the time using the CLI
* Incremental builds
    * Makes a single change (ABI or non-ABI)
    * Uses up-to-date state to determine what to rerun
    * No need to clean between builds  
    * Measures `assemble` time
    * Measure the time using the tooling API (used by IDEs)

### Example of incremental scenario
```
abiChange {
  tasks = ["assemble"]
  apply-abi-change-to = "project0/src/main/java/org/gradle/test/performance/smalljavamultiproject/project0/p0/Production0.java"
  system-properties {
        org.gradle.unsafe.instant-execution = "true"
        org.gradle.unsafe.vfs.retention = "true"
  }
  bazel {
    targets = ["build", "//..."]
  }
}
```

## Running a performance test

Make sure you have installed the Gradle Profiler and that it's available on your PATH.
The generated projects may have different defaults so please ensure to set up the projects as described in "Setup".
Change directory to a test project and run the following commands. This will execute the `abiChange` scenario.

### Measuring the performance of Gradle (Incremental)

```
gradle-profiler --gradle-version 6.4 --benchmark --scenario-file performance.scenarios abiChange
``` 

### Measuring the performance of Bazel (Incremental)

```
gradle-profiler --benchmark --scenario-file performance.scenarios abiChange --bazel
```



