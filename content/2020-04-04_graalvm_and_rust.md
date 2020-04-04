+++
title = "Running Rust Crates in GraalVM's LLVM Bitcode Executor"
slug = "graalvm-and-rust-1"
+++

### Introduction to this post

This post was written with the goal of getting some kind of Rust crate running in GraalVM's LLVM compatibility layer by any means possible in a relatively short period.
While the code in this post does compile and run, it is heavily contrived.
I would not advise executing Rust as LLVM bitcode in any kind of production application.
If you would like to integrate Rust into Graalvm, I would advise having a look Michiel Borkent's [example](https://github.com/borkdude/clojure-rust-graalvm) using native images and JNI.

This post assumes that you have already installed Rust, GraalVM, LLVM, and Clang.

### What is GraalVM?
When programming, you will occasionally run into a library that is not available in the language you are writing in.
Your options are usually:
- Port the library
- Make a compiled library using the C ABI and access it over a foreign function interface
- Call a program written in another language from the shell
- Run a server written another language and call it over HTTP
- Include an interpreter for another language in your code

There is another method used in the .Net framework and on Java Virtual Machine (JVM) based languages, where you can call directly into the other language's code.
These two language families run on common virtual machines allowing languages within these families to call each other natively.

GraalVM is an effort by Oracle to extend the JVM to run a wide range of languages including: Python, Java, JavaScript, and any language that can be compiled to LLVM bitcode.

LLVM bitcode is a binary format of LLVM intermediate representation, (or IR) a common language compilers targeting LLVM produce.

### The Issues with running Rust on GraalVM using bitcode
Rust can theoretically be compiled to bitcode because it uses LLVM as a back end.
Unfortunately, actually running Rust crates in GraalVM is not as easy as just changing the compiler target.

The reason is that there are a few problems with LLVM bitcode:
- It bitcode not stable. This is made particularly problematic by the fact that Rust and GraalVM both bundle their own versions of LLVM.
- It bitcode not necessarily cross platform
- There is no bitcode compilation target for Cargo
- A lot of Rust crates assume they are running on a real computer
    
    For example [ring](https://github.com/briansmith/ring) depends on dynamically generated assembly, which would have to be manually ported to Rust or LLVM IR

Additionally, something Cargo adds to main seems to break GraalVM.

### Let's start by reading the documentation
While I may make the task of running a crate sound impossible, Rust bitcode does already run in Graal.
Better yet, Oracle has provided some [documentation](https://www.graalvm.org/docs/reference-manual/languages/llvm/#running-rust) for running Rust as bitcode in GraalVM. (Note: Since I don't know the license terms for this documentation I will be using modified code.)

1. The docs suggest building a simple application (with no dependencies):
``` rust
fn main() {
    println!("Hello, world!");
}
```
2. Compiling it manually with rustc using the `--emit=llvm-bc` flag:
``` sh
$ rustc --emit=llvm-bc helloworld.rs
```
3. And then running the bitcode which was produced as a byproduct of compilation:
``` sh
$ lli --lib $(rustc --print sysroot)/lib/libstd-* helloworld.bc
```

The code does work assuming your LLVM versions are compatible.
Unfortunately, because rustc is a compiler and not a build tool, this method does not support adding dependencies.

### A more complex test case
In order to test running a complex crate, I will be trying to run a crate containing a basic test of the [scraper crate](https://crates.io/crates/scraper).

`src/main.rs`:
``` rust
use scraper::{Html, Selector};

fn main() {
	// Based heavily on the scraper crate readme
	let html = r#"
		<!DOCTYPE html>
		<html>
			<head>
				<meta charset="utf-8">
				<title>Hello, world!</title>
			</head>
				<body>
				<main>
					<div class="foo">Bar</div>
					<div class="foo">Baz</div>
					<div class="green">Eggs</div>
				</main>
			</body>
		</html>
	"#;

	let document = Html::parse_document(html);
	let title_sel = Selector::parse("title").unwrap();
	let foo_sel = Selector::parse(".foo").unwrap();
	for title in document.select(&title_sel) {
		println!(
			"Document title: {}",
			title.text()
				.map(String::from)
				.collect::<String>()
		);
	}
	println!("Items of class foo in the document:");
	for foo in document.select(&foo_sel) {
		println!(
			"\t{}", foo.text()
				.map(String::from)
				.collect::<String>()
		);
	}
}
```
`Cargo.toml`
``` toml
[package]
name = "graalhello"
version = "0.1.0"
authors = ["Michael Hardy"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
scraper = "0.11"
```

### Trying the naive way
So let's just try to apply that method to the crate above:
```sh
$ cargo rustc --release -- --emit=llvm-bc
```
This command calls `cargo rustc`, a cargo subcommand designed to pass arguments directly to rustc during a cargo build.

Now that our code has compiled we can try running it:
``` sh
$ lli --lib $(rustc --print sysroot)/lib/libstd-* target/release/deps/graalhello-*.bc
Global variable _ZN12string_cache4atom12STRING_CACHE17h43d70dbc9890e871E is declared but not defined.
	at <llvm> null(Unknown)
```
This error indicates that the bitcode file declares an external dependency on a variable which is not included.
In other words the bitcode file has not been linked.

### Ensuring compilation of libraries
In order to add libraries to the compiled binary we need to enable something called link time optimization.

In ordinary linking, the linker is passed a different libraries as assembly.
Unfortunately, it is a lot harder for the linker to optimize code already in assembly.
So the LLVM provides link time optimization, which passes the linker intermediate representation for all dependencies.
While it is intended to help the optimizer, in this case it results in a complete bitcode file being produced.

To enable link time optimization add the following configuration to your `Cargo.toml`:
``` toml
[profile.dev]
lto = true

[profile.release]
lto = true
```

Before recompiling delete the previous bitcode by running:
```sh
$ rm target/release/deps/graalhello-*.bc
```
Then recompile:
```sh
$ cargo rustc --release -- --emit=llvm-bc
```
And run the code again:
``` sh
$ lli --lib $(rustc --print sysroot)/lib/libstd-* target/release/deps/graalhello-*.bc
ERROR: com.oracle.truffle.api.dsl.UnsupportedSpecializationException: Unexpected values provided for <signals.c:36:30>:36 LLVMSignalNodeGen#1: [13, 1], [Integer,Long]
org.graalvm.polyglot.PolyglotException: com.oracle.truffle.api.dsl.UnsupportedSpecializationException: Unexpected values provided for <signals.c:36:30>:36 LLVMSignalNodeGen#1: [13, 1], [Integer,Long]
	at com.oracle.truffle.llvm.runtime.nodes.intrinsics.c.LLVMSignalNodeGen.executeAndSpecialize(LLVMSignalNodeGen.java:76)
	at com.oracle.truffle.llvm.runtime.nodes.intrinsics.c.LLVMSignalNodeGen.executeGeneric(LLVMSignalNodeGen.java:52)
	at com.oracle.truffle.llvm.runtime.nodes.api.LLVMFrameNullerExpression.executeGeneric(LLVMFrameNullerExpression.java:75)
	at com.oracle.truffle.llvm.runtime.nodes.vars.LLVMWriteNodeFactory$LLVMWritePointerNodeGen.execute(LLVMWriteNodeFactory.java:714)
	at com.oracle.truffle.llvm.runtime.nodes.base.LLVMBasicBlockNode$InitializedBlock.execute(LLVMBasicBlockNode.java:154)
	at com.oracle.truffle.llvm.runtime.nodes.control.LLVMDispatchBasicBlockNode.executeGeneric(LLVMDispatchBasicBlockNode.java:81)
	at com.oracle.truffle.llvm.runtime.nodes.control.LLVMFunctionRootNode.executeGeneric(LLVMFunctionRootNode.java:75)
	at com.oracle.truffle.llvm.runtime.nodes.func.LLVMFunctionStartNode.execute(LLVMFunctionStartNode.java:87)
	at <llvm> main(src/libstd/sys/unix/mod.rs:89:0)
	at org.graalvm.polyglot.Value.execute(Value.java:367)
	at com.oracle.truffle.llvm.launcher.LLVMLauncher.execute(LLVMLauncher.java:219)
	at com.oracle.truffle.llvm.launcher.LLVMLauncher.launch(LLVMLauncher.java:63)
	at org.graalvm.launcher.AbstractLanguageLauncher.launch(AbstractLanguageLauncher.java:121)
	at org.graalvm.launcher.AbstractLanguageLauncher.launch(AbstractLanguageLauncher.java:70)
	at com.oracle.truffle.llvm.launcher.LLVMLauncher.main(LLVMLauncher.java:53)
...
```
Unfortunately, I did not have enough time to work out the cause of this issue.
As far as I can tell, this is caused by something cargo inserts into the program entry point.
So I did something rather hacky, and changed the entry point of the program by calling it from c.

### Changing the entry point
In order change the entry point of the program we will need to convert the program to a library.
This can be done by adding the following lines to your `Cargo.toml`:
``` toml
[lib]
crate-type = ["cdylib"]
```
The use of cdylib here is not a mistake.
Compiling to a cdylib ensures that all the dependencies are linked into the compiled library.

After changing the program to a `cdylib`, it will be necessary to move `src/main.rs` to `src/lib.rs`.
Inside of `src/lib.rs` `fn main` was replaced like so:
``` rust
#[no_mangle]
pub extern fn start() {
    // Based heavily on the scraper crate readme
    let html = r#"
    ...
}
```

Then add a file called `src/stub.c` that will hold the entry point to the program:
``` c
// Import the entry point to the Rust application
void start();

int main(int argc, char** argv) {
    start();
}
```

Now build the two part program:
1. Compile the Rust code:
```sh
$ cargo rustc --release -- --emit=llvm-bc
```
2. Compile the c code to IR:
``` sh
$ clang src/stub.c -S -emit-llvm
```
3. Assemble the stub IR to bitcode:
``` sh
$ llvm-as stub.ll
```
4. Link the two bitcode files:
``` sh
$ llvm-link stub.bc target/release/deps/graalhello.bc -o graalhello.bc
```
The warning does not matter in this case.
5. Run it:
``` sh
$ lli --lib $(rustc --print sysroot)/lib/libstd-* graalhello.bc
ERROR: java.lang.IllegalStateException: Missing LLVM builtin: llvm.fshl.i64
org.graalvm.polyglot.PolyglotException: java.lang.IllegalStateException: Missing LLVM builtin: llvm.fshl.i64
	at com.oracle.truffle.llvm.runtime.nodes.intrinsics.llvm.x86.LLVMX86_MissingBuiltin.executeGeneric(LLVMX86_MissingBuiltin.java:48)
	at com.oracle.truffle.llvm.runtime.nodes.api.LLVMFrameNullerExpression.executeGeneric(LLVMFrameNullerExpression.java:75)
	at com.oracle.truffle.llvm.runtime.nodes.op.LLVMArithmeticNodeFactory$PointerToI64NodeGen.executeGeneric_generic1(LLVMArithmeticNodeFactory.java:506)
	at com.oracle.truffle.llvm.runtime.nodes.op.LLVMArithmeticNodeFactory$PointerToI64NodeGen.executeGeneric(LLVMArithmeticNodeFactory.java:479)
	at com.oracle.truffle.llvm.runtime.nodes.op.LLVMArithmeticNodeFactory$LLVMI64ArithmeticNodeGen.executeGeneric_generic3(LLVMArithmeticNodeFactory.java:22
```

### The caveats of GraalVM's LLVM support
As we can see in the above error, Graal [does not support the entirety of LLVM IR](https://www.graalvm.org/docs/reference-manual/languages/llvm/#limitations-and-differences-to-native-execution-on-top-of-graalvm-ce).
In particular, it does not support `llvm.fshl`, the funnel shift left operation. 
Funnel shift left is produced somewhat frequently by rustc.
So I decided to remove it.

### Removing `fshl`
#### Or not knowing when to give up
Fortunately, `fshl` can be removed if you know to reimpliment it in LLVM IR.
1. Take the `graalhello.bc` file and disassemble it to IR:
``` sh
$ llvm-dis graalhello.bc
```
2. Open up `graalhello.ll` in your preferred editor.
Hopefully it does not crash.
3. find a line starting with:
``` llvm
declare i64 @llvm.fshl.i64(
```
It should look something like this:
``` llvm
declare i64 @llvm.fshl.i64(i64, i64, i64) #24
```
5. Replace that line with the following reimplimentation:
``` llvm
define i64 @fshli64(i64 %a, i64 %b, i64 %s) #24 {
  ; Concatenates a and b and then shifts them left by s
  ; The upper 64 bits of this value are then returned
  ; Sign extend all vars to 128 bits
  %a_extended = zext i64 %a to i128
  %b_extended = zext i64 %b to i128
  %s_extended = zext i64 %s to i128
  ; Shift a left 64 bits to allow a to be concatenated
  ; into the upper 64 bits
  %a_shift = shl i128 %a_extended, 64
  ; Concatenate both ints
  %concat = and i128 %a_shift, %b_extended
  ; Do the actual shift
  %shift_cat = shl i128 %concat, %s_extended
  ; Shift right 64 bits to allow the upper 64 bits to be read
  %shift_back = lshr i128 %shift_cat, 64
  ; Truncate back to 64 bits leaving the upper 64
  %re = trunc i128 %shift_back to i64
  ret i64 %re
}
```
If the # sign in the original line was not followed by 24, use that number in the code above instead.

6. Replace all instances of `@llvm.fshl.i64(` with `@fshli64(`
7. Save
8. Reassemble the IR to bitcode:
``` sh
$ llvm-as graalhello.ll
```
9. Execute the application again:
``` sh
$ lli --lib $(rustc --print sysroot)/lib/libstd-* graalhello.bc
Document title: Hello, world!
Items of class foo in the document:
	Bar
	Baz
```
It Runs!

### Conclusions
While, with the exception of patching out `fshl`, the modifications above may seem relatively simple, creating this example proved distinctly more complicated.
Before writing this post, I attempted running 3 different web libraries: Rocket, Gotham, and Iron.
Gotham generated invalid IR.
Rocket (after removing ring) crashed Graal with a long stack trace.
And Iron caused Graal to segfault.
As a side effect of this, I suspect most libraries that perform IO will not run under GraalVM's LLVM executor.

~~Alternatively, the execution problems could be coming from the fact that my copy of Rust is using LLVM 9 while Graal is using version 7.  At some point I will have to try downgrading to Rust 1.29.~~
(There are similar issues under 1.29 nightly)