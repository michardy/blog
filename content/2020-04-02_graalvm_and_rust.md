+++
title = "Running Rust Crates in GraalVM"
slug = "graalvm-and-rust-1"
draft = true
+++
### Intro to GraalVM
When programming you will occasionally run into a library that is not available in the language you are writing in.
Your options are usually:
- Port the library
- Make a compiled library using the C ABI and access it over a foreign function interface
- Call a program written in another language from the shell
- Run a server written another language and call it over HTTP
- Include an interpreter for another language in your code

Obviously the last two options come with the most overhead.

There is another method used in the .net framework and on Java Virtual Machine (JVM) based langages where you can call directly and nearly natively into the other language's code.
These two language families run on common virtual machines allowing languages with in the families to call each other natively.

GraalVM is an effort by Oracle to extend the JVM to run a wide range of languages including: Python, Java, Javascript, and any language that can be compiled to LLVM bitcode.

LLVM bitcode is a binary format of LLVM intermediate representation, (or IR) a common language compilers targeting LLVM produce.

### The Issues with running Rust on GraalVM
As Rust uses LLVM as a back end it can theoretically be compiled to LLVM bitcode.
Unfortunatly actually running Rust Crates in GraalVM is not as easy as just changing the compiler target.

The reason is that there are a few problems with LLVM bitcode:
- It bitcode not stable. This is made particularly problematic by the fact that Rust and GraalVM both bundle there own versions of LLVM.
- It bitcode not nessisarily cross platform
- There is no bitcode compilation target for Cargo
- A lot of Rust Crates assume they are running on a real computer
    
    For example [ring](https://github.com/briansmith/ring) depends on dynamically generated assembly, which would have to be manually ported to Rust or LLVM IR

- Something Cargo adds to main seems to break GraalVM

### Lets start by reading the documentation
While I may make the task of running a Crate sound impossible, Rust code does already run in Graal
Better yet, Oracle has provided some [documentation](https://www.graalvm.org/docs/reference-manual/languages/llvm/#running-rust) for running Rust in GraalVM.
The docs suggest building a simple application (with no dependencies), compiling it manually with rustc using the `--emit=llvm-bc` flag, and then running the bitcode produced as a byproduct.  
The code does work assuming your LLVM versions are compatable.
Unfortunatly, this method does not support adding dependencies from crates.io.

### Trying the naive way
So lets just try and apply that method to a project:
```sh
cargo rustc --release -- --emit=llvm-bc
```
This command calls cargo rustc, a cargo subcommand designed to pass arguments directly to rustc.
The command does compile your code but it fails to include any libraries in the resulting bitcode.