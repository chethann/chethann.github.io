---
title: Breaking Down KMP and CMP - The Backbone of Kotlin Multiplatform Development
date: 2024-12-12 12:00:00 +0000
categories: [kmp, cmp]
tags: [cmp, kmp]
---

## What is Kotlin Multiplatform (KMP)?

Kotlin Multiplatform (KMP) is a technology that allows developers to share code across multiple platforms while retaining the ability to use platform-specific APIs and tools. It primarily targets common business logic and aims to reduce duplication by enabling a write once, run everywhere model for non-UI code.

**Core Highlights:**<br>
	1.	**Shared Codebase:** Write platform-agnostic logic (e.g., data handling, networking) in Kotlin and share it across Android, iOS, Desktop, Web, and other platforms.<br>
	2.	**Interoperability:** KMP integrates seamlessly with platform-specific languages like Swift (Through Objective headers as of now) for iOS and Java/Kotlin for Android.<br>
	3.	**Customizability:** Developers can write platform-specific code when necessary, enabling fine-grained control.<br>

KMP (without CMP) shines in projects with substantial shared logic but leaves the UI implementation to native frameworks like Jetpack Compose, SwiftUI, or React. <br>

### Why its not just KN (Kotlin Native) or KMM (Kotlin Multiplatform Mobile) anymore!

**Kotlin/Native** Kotlin/Native specifically refers to the part of KMP that uses Kotlin compiled to native binaries via LLVM for platforms like iOS. It focuses exclusively on platforms where no JVM is available, like iOS or embedded systems. KMP includes Kotlin/Native, but also supports other backends like JVM and JavaScript. KMP is a superset that enables code sharing across all supported platforms, not just native ones. <br>

**KMM** was JetBrains’ term for a KMP use case focused specifically on mobile platforms: Android and iOS. KMP is a broader framework that targets not just mobile, but also Web, Desktop, and beyond. KMM is still valid as a subset of KMP, but the term KMP better reflects its expanded reach.

<br>

## How Kotlin Multiplatform Compiles to Target Platforms

Kotlin Multiplatform (KMP) leverages Kotlin’s [compiler](https://github.com/JetBrains/kotlin/tree/master/compiler) infrastructure to produce platform-specific binaries. The compilation process involves transforming Kotlin source code into intermediate representations and then to executable code tailored for each target platform. Here’s an overview of the key steps: <br>

### A bit on code compilation at high level

A compiler typically operates in two main phases: **front-end** and **back-end**, each with distinct roles in the compilation process. Here’s an overview:

#### Front-End Phases

The front-end focuses on analyzing the source code and ensuring it is syntactically and semantically correct. It generates an intermediate representation (IR) of the code. The phases are:

- **Lexical Analysis (Scanning):** Converts the source code into a stream of tokens, where tokens represent atomic elements like keywords, operators, or identifiers.
- **Syntax Analysis (Parsing):** Checks if the tokens form a valid program according to the grammar rules of the language (Kotlin in this case) and produces a parse tree or abstract syntax tree (AST), representing the program’s structure.
- **Semantic Analysis:** Verifies the semantic correctness of the code, does type checking, variable scope and usage verification. Also, updates the AST with additional information, such as data types.
- **Intermediate Representation (IR) Generation:** Converts the AST into a simpler, platform-independent IR. IR is often a linear or graph-based structure that is easier for back-end processing. Example: LLVM IR or JVM bytecode for intermediate processing.

Complete details of front end IR phase of Kotlin can be accessed at [Short Frontend IR overview](https://github.com/JetBrains/kotlin/blob/master/docs/fir/fir-basics.md) and [IR](https://github.com/JetBrains/kotlin/tree/master/compiler/ir/)  

> **Note**:
- FIR (Frontend Intermediate Representation) is an intermediate representation used in the front-end of the Kotlin compiler. It bridges the gap between the Abstract Syntax Tree (AST) and the back-end IR (Kotlin IR). FIR is primarily designed to improve the performance, maintainability, and modularity of the front-end. 
- The Kotlin IR is converted into LLVM IR in case of Native targets like iOS, JVM bytecode when targetting JVM tragets like Android / desktop. LLVM IR is a low-level, platform-independent intermediate representation that LLVM can process further for optimization and code generation.
- Konan is the internal codename for Kotlin/Native, a part of the KMP ecosystem.

####  Back-End Phases

The back-end translates the intermediate representation into optimized, platform-specific machine code. It focuses on efficiency and compatibility.
- **Optimization:** Performs transformations on the IR to improve performance without changing the program’s behavior.
- **Code Generation:** Translates the optimized IR into target machine code (e.g., assembly or binary instructions). Ensures the generated code adheres to the conventions of the target architecture (e.g., x86, ARM).
- **Code Emission:** Converts the machine code into a format suitable for execution or linking, such as object files or executables.
- **Linking (Optional):** Combines compiled object files with libraries to produce a complete executable.




### Common Code Compilation

KMP supports writing platform-agnostic common code in the shared module. This code is compiled into an Intermediate Representation (IR), which serves as a platform-independent, lower-level abstraction of the code. The IR phase ensures: <br>
- Shared logic remains consistent across platforms.
- Common Kotlin constructs (e.g., classes, functions) are transformed into a unified format.

### Platform-Specific Code Generation

KMP compiles platform-specific modules by transforming the [IR](https://en.wikipedia.org/wiki/Intermediate_representation) into platform-native code using the appropriate backend. <br>

**Backends and Their Roles**
1.	**[JVM Backend](https://github.com/JetBrains/kotlin/tree/master/compiler/ir/backend.jvm):**
    - For platforms like Android, Desktop, or server-side applications.
	- The IR is converted into Java Bytecode (.class files) compatible with the JVM.
2.	**[JS Backend](https://github.com/JetBrains/kotlin/tree/master/compiler/ir/backend.js):**
	- For targeting browsers or Node.js.
	- The IR is transpiled into JavaScript (or TypeScript, if needed).
3.	**[Native Backend](https://github.com/JetBrains/kotlin/blob/master/kotlin-native/README.md) (Powered by LLVM):**
	- For iOS, macOS, Linux, Windows, and other native platforms.
	- The IR is converted into [LLVM](https://en.wikipedia.org/wiki/LLVM) IR, a standard intermediate representation widely used in     modern compiler toolchains.
	- LLVM then processes this IR to generate optimized machine code specific to the target CPU and architecture (e.g., ARM for iOS, x86 for macOS).
4. **[Wasm Backend](https://github.com/JetBrains/kotlin/blob/master/compiler/ir/backend.wasm/ReadMe.md):**
Kotlin’s experimental Wasm backend allows Kotlin code to compile to Wasm, leveraging the same IR (Intermediate Representation). The IR is lowered into WebAssembly Text Format (WAT) and then compiled into Wasm binary format (.wasm). 

### LLVM and Kotlin/Native

Kotlin/Native is the foundation of KMP’s native compilation. It leverages LLVM (Low-Level Virtual Machine) to enable seamless targeting of non-VM platforms like iOS. Here’s how it works: <br>

IR to LLVM IR: Kotlin’s IR is translated into LLVM IR, which is platform-agnostic. <br>
- **Optimization Passes:** LLVM applies numerous optimization passes (e.g., dead code elimination, inlining) to improve performance.
- **Platform-Specific Codegen:** LLVM IR is compiled into native machine code for the target platform (e.g., iOS ARM binaries or macOS x86 binaries).
- **Interop Layer:** KMP includes tools for interop with native codebases (e.g., Objective-C/Swift on iOS).

### Benefits of This Approach
- **Flexibility:** Shared logic through IR while allowing platform-specific optimizations.
- **Performance:** LLVM’s optimization capabilities ensure efficient machine code generation.
- **Interop:** The ability to call platform-native APIs seamlessly.

In essence, the IR phase and LLVM backend are at the heart of KMP’s ability to support diverse platforms, bridging the gap between Kotlin’s high-level abstractions and native performance.

<br>

## What is CMP (Compose Multiplatform) and how it works on different platforms

Compose Multiplatform (CMP) is a declarative UI framework that extends Jetpack Compose, initially developed for Android, to support multiple platforms including Desktop, Web, and iOS. It’s part of Kotlin Multiplatform (KMP) and focuses on enabling a shared codebase for creating modern, reactive UIs across platforms.

### How does it relate to Jetpack Compose for Android?

From  the official documentation - *Compose Multiplatform shares most of its API with Jetpack Compose, the Android UI framework developed by Google. In fact, when you are using Compose Multiplatform to target Android, your app simply runs on Jetpack Compose. Other platforms targeted by Compose Multiplatform may have implementation details under the hood that differ from those of Jetpack Compose on Android, but they still provide you with the same APIs.* <br>

We will delve a bit on Skia and Skiko to understand things in a little more depth   

### Skia: The Core Rendering Engine

**Skia** is an open-source 2D graphics library developed by Google. It is widely used in rendering frameworks and platforms, including Chrome, Android, Flutter, and now Compose Multiplatform (CMP). Skia handles low-level graphics operations, such as:
- Drawing shapes, text, and images.
- Applying transformations, gradients, and shaders.
- Managing high-performance rendering across various operating systems.

Key Features of Skia:
- Cross-Platform Support: Works seamlessly on macOS, Windows, Linux, Android, and iOS.
- Hardware Acceleration: Uses GPU APIs like Vulkan, OpenGL, and Metal for faster rendering.
- High Fidelity: Ensures pixel-perfect, platform-consistent rendering.

In CMP, Skia is the backbone for rendering UI components on non-Android platforms like Desktop and iOS.

### Skiko: The Kotlin Multiplatform Wrapper for Skia

**Skiko (Skia for Kotlin Multiplatform)** is a JetBrains-developed library that acts as a bridge between Kotlin and Skia. It is designed to:
- Integrate Skia rendering into Kotlin Multiplatform projects.
- Provide an abstraction layer for rendering UI components using Skia’s powerful graphics capabilities.

Skiko plays a crucial role in making Compose Desktop and Compose Multiplatform work by:
- Handling rendering pipelines for Desktop platforms (Windows, macOS, Linux).
- Bridging Skia’s C++ implementation with Kotlin through Kotlin/Native.

Why Skiko?
- Kotlin-first API tailored for Compose Multiplatform.
- Simplifies cross-platform rendering by abstracting Skia’s complex graphics operations.

How Skia and Skiko Fit into Compose Multiplatform (CMP)

In CMP, Skia and Skiko work together to provide a unified rendering solution for platforms outside of Android. Here’s how they are used across CMP’s target platforms:
1.	**Desktop (Windows, macOS, Linux):**
- CMP leverages Skiko to integrate Skia’s rendering capabilities.
- The UI is drawn directly using Skia, independent of native windowing toolkits, ensuring consistent rendering.
2.	**iOS:**
- CMP uses Skia via Kotlin/Native for rendering UI components.
- This approach allows CMP to deliver cross-platform UI elements while enabling interop with UIKit for native functionality.
3.	**Web:**
- While Skia isn’t directly used for the Web target, CMP employs a Canvas-based rendering approach inspired by Skia for drawing UI elements.
4.	**Android:**
- CMP uses Jetpack Compose, which relies on Android’s own Skia-based graphics stack for rendering.

With this, Compose Multiplatform (CMP) represents a significant leap forward in building **truly cross-platform user interfaces**. By unifying development for Android, iOS, desktop, web, and beyond under a single Kotlin-based framework, CMP simplifies the traditionally fragmented world of UI development. Its reliance on Kotlin Multiplatform (KMP) allows developers to share not just logic but also UI elements across platforms