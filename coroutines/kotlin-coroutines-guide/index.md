# Kotlin Coroutines Complete Guide

A comprehensive guide to mastering Kotlin coroutines for asynchronous programming.

## Table of Contents

1. [Lesson 1: Introduction to Coroutines](lesson-01-introduction.md)
2. [Lesson 2: Coroutine Builders and Contexts](lesson-02-builders-contexts.md)
3. [Lesson 3: Exception Handling and Cancellation](lesson-03-exception-cancellation.md)
4. [Lesson 4: Flow - Asynchronous Data Streams](lesson-04-flow-streams.md)
5. [Lesson 5: Advanced Coroutines Patterns](lesson-05-advanced-patterns.md)
6. [Lesson 6: Real-World Applications](lesson-06-real-world.md)
7. [Quick Reference Guide](quick-reference.md)

## Overview

This guide covers:
- **Basic concepts** of coroutines and why they matter
- **Practical examples** with real-world scenarios
- **Advanced patterns** for production applications
- **Integration examples** for Android, web, and microservices
- **Best practices** for building robust asynchronous systems

## Getting Started

### Prerequisites
- Basic Kotlin knowledge
- Understanding of asynchronous programming concepts

### Dependencies
Add to your `build.gradle.kts`:
```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
}
```

### Quick Start
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    launch {
        delay(1000L)
        println("Hello from coroutine!")
    }
    println("Hello from main!")
}
```

## Author
MiniMax Agent - Kotlin Coroutines Expert

## License
Educational content for learning purposes.