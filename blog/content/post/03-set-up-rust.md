+++
title = "Set Up Rust"
date = "2015-09-02"
updated = "2016-05-29"
aliases = [
    "/2015/09/02/setup-rust/",
    "/setup-rust.html",
    "/rust-os/setup-rust.html",
]
+++

In the previous posts we created a [minimal Multiboot kernel][multiboot post] and [switched to Long Mode][long mode post]. Now we can finally switch to [Rust] code. Rust is a high-level language without runtime. It allows us to not link the standard library and write bare metal code. Unfortunately the setup is not quite hassle-free yet.

[multiboot post]: {{% relref "01-multiboot-kernel.md" %}}
[long mode post]: {{% relref "02-entering-longmode.md" %}}
[Rust]: https://www.rust-lang.org/

<!--more--><aside id="toc"></aside>

This blog post tries to set up Rust step-by-step and point out the different problems. If you have any questions, problems, or suggestions please [file an issue] or create a comment at the bottom. The code from this post is in a [Github repository], too.

[file an issue]: https://github.com/phil-opp/blog_os/issues
[Github repository]: https://github.com/phil-opp/blog_os/tree/set_up_rust

## Installing Rust
We need a nightly compiler, as we will use many unstable features. To manage Rust installations I highly recommend [rustup]. It allows you to install nightly, beta, and stable compilers side-by-side and makes it easy to update them. To use a nightly compiler for the current directory, you can run `rustup override add nightly`.

[rustup]: https://www.rustup.rs/

The code from this post (and all following) is [automatically tested](https://travis-ci.org/phil-opp/blog_os) every day and should always work for the newest nightly. If it doesn't, please [file an issue](https://github.com/phil-opp/blog_os/issues).

## Creating a Cargo project
[Cargo] is Rust excellent package manager. Normally you would call `cargo new` when you want to create a new project folder. We can't use it because our folder already exists, so we need to do it manually. Fortunately we only need to add a cargo configuration file named `Cargo.toml`:

[Cargo]: http://doc.crates.io/guide.html

```toml
[package]
name = "blog_os"
version = "0.1.0"
authors = ["Philipp Oppermann <dev@phil-opp.com>"]

[lib]
crate-type = ["staticlib"]
```
The `package` section contains required project metadata such as the [semantic crate version]. The `lib` section specifies that we want to build a static library, i.e. a library that contains all of its dependencies. This is required to link the Rust project with our kernel.

[semantic crate version]: http://doc.crates.io/manifest.html#the-package-section

Now we place our root source file in `src/lib.rs`:

```rust
#![feature(lang_items)]
#![no_std]

#[no_mangle]
pub extern fn rust_main() {}

#[lang = "eh_personality"] extern fn eh_personality() {}
#[lang = "panic_fmt"] #[no_mangle] pub extern fn panic_fmt() -> ! {loop{}}
```
Let's break it down:

- `#!` defines an [attribute] of the current module. Since we are at the root module, they apply to the crate itself.
- The `feature` attribute is used to allow the specified _feature-gated_ attributes in this crate. You can't do that in a stable/beta compiler, so this is one reason we need a Rust nighly.
- The `no_std` attribute prevents the automatic linking of the standard library. We can't use `std` because it relies on operating system features like files, system calls, and various device drivers. Remember that currently the only “feature” of our OS is printing `OKAY` :).
- A `#` without a `!` afterwards defines an attribute for the _following_ item (a function in our case).
- The `no_mangle` attribute disables the automatic [name mangling] that Rust uses to get unique function names. We want to do a `call rust_main` from our assembly code, so this function name must stay as it is.
- We mark our main function as `extern` to make it compatible to the standard C [calling convention].
- The `lang` attribute defines a Rust [language item].
- The `eh_personality` function is used for Rust's [unwinding] on `panic!`. We can leave it empty since we don't have any unwinding support in our OS yet.
- The `panic_fmt` function is the entry point on panic. Right now we can't do anything useful, so we just make sure that it doesn't return (required by the `!` return type).

[attribute]: https://doc.rust-lang.org/book/attributes.html
[name mangling]: https://en.wikipedia.org/wiki/Name_mangling
[calling convention]: https://en.wikipedia.org/wiki/Calling_convention
[language item]: https://doc.rust-lang.org/book/lang-items.html
[unwinding]: https://doc.rust-lang.org/nomicon/unwinding.html

## Building Rust
We can now build it using `cargo build`. This command creates a static library at `target/debug/libblog_os.a`, which can be linked with our assembly kernel. However, the resulting library is specific to our _host_ operating system. This is undesirable, because our target system might be very different.

Let's define some properties of our target system:

- **x86_64**: Our target CPU is a recent `x86_64` CPU.
- **No operating system**: Our target does not run any operating system (we're currently writing it), so the compiler should not assume any OS-specific functionality.
- **Handles hardware interrupts**: We're writing a kernel, so we'll need to handle asynchronous hardware interrupts at some point. This means that we have to disable a certain stack pointer optimization (the so-called [red zone]), because it would cause stack corruptions otherwise.
- **No SSE**: Our target might not have [SSE] support. Even if it does, we probably don't want to use SSE instructions in our kernel, because it makes interrupt handling much slower. We will explain this in detail in the [“Handling Exceptions”] post.
- **No hardware floats**: The `x86_64` architecture uses SSE instructions for floating point operations, which we don't want to use (see the previous point). So we also need to avoid hardware floating point operations in our kernel. Instead, we will use _soft floats_, which are basically software functions that emulate floating point operations using normal integers.

### Target Specifications
Rust allows us to define [custom targets] through a JSON configuration file. A minimal target specification equal to `x86_64-unknown-linux-gnu` (the default 64-bit Linux target) looks like this:

```json
{
  "llvm-target": "x86_64-unknown-linux-gnu",
  "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
  "target-endian": "little",
  "target-pointer-width": "64",
  "arch": "x86_64",
  "os": "none" TODO
}
```

The `llvm-target` field specifies the target triple that is passed to LLVM. [Target triples] are a naming convention that define the CPU architecture (e.g., `x86_64` or `arm`), the vendor (e.g., `apple` or `unknown`), the operating system (e.g., `windows` or `linux`), and the [ABI] \(e.g., `gnu` or `msvc`). For example, the target triple for 64-bit Linux is `x86_64-unknown-linux-gnu` and for 32-bit Windows the target triple is `i686-pc-windows-msvc`.

The `data-layout` field is also passed to LLVM and specifies how data should be laid out in memory. It consists of various specifications seperated by a `-` character. For example, the `e` means little endian and `S128` specifies that the stack should be 128 bits (= 16 byte) aligned. The format is described in detail in the [LLVM documentation][data layout] but there shouldn't be a reason to change this string.

The other fields are used for conditional compilation. This allows crate authors to use `cfg` variables to write special code for depending on the OS or the architecture. There isn't any up-to-date documentation about these fields but the [corresponding source code][target specification] is quite readable.

[data layout]: http://llvm.org/docs/LangRef.html#data-layout
[target specification]: https://github.com/rust-lang/rust/blob/c772948b687488a087356cb91432425662e034b9/src/librustc_back/target/mod.rs#L194-L214

### A Kernel Target Specification
For our target system, we define the following JSON configuration in a file named `x86_64-blog_os.json`:

```json
{
  "llvm-target": "x86_64-unknown-none",
  "data-layout": "e-m:e-i64:64-f80:128-n8:16:32:64-S128",
  "target-endian": "little",
  "target-pointer-width": "64",
  "arch": "x86_64",
  "os": "none",
  "disable-redzone": true,
  "features": "-mmx,-sse,+soft-float"
}
```

As `llvm-target` we use `x86_64-unknown-none`, which defines the `x86_64` architecture, an `unknown` vendor, and no operating system (`none`). The ABI doesn't matter for us, so we just leave it off. The `data-layout` field is just copied from the `x86_64-unknown-linux-gnu` target. We also use the same values for the `target-endian`, `target-pointer-width`, and `arch` fields. For the `os` field we choose `none`, since our kernel runs on bare metal.

#### The Red Zone
The [red zone] is an optimization of the [System V ABI] that allows functions to temporary use the 128 bytes below its stack frame without adjusting the stack pointer:

[red zone]: http://eli.thegreenplace.net/2011/09/06/stack-frame-layout-on-x86-64#the-red-zone

![stack frame with red zone](images/red-zone.svg)

The image shows the stack frame of a function with `n` local variables. On function entry, the stack pointer is adjusted to make room on the stack for the local variables.

The red zone is defined as the 128 bytes below the adjusted stack pointer. The function can use this area for temporary data that's not needed across function calls. Thus, the two instructions for adjusting the stack pointer can be avoided in some cases (e.g. in small leaf functions).

However, this optimization leads to huge problems with exceptions or hardware interrupts. Let's assume that an exception occurs while a function uses the red zone:

![red zone overwritten by exception handler](images/red-zone-overwrite.svg)

The CPU and the exception handler overwrite the data in red zone. But this data is still needed by the interrupted function. So the function won't work correctly anymore when we return from the exception handler. This might lead to strange bugs that [take weeks to debug].

[take weeks to debug]: http://forum.osdev.org/viewtopic.php?t=21720

To avoid such bugs when we implement exception handling in the future, we disable the red zone right from the beginning. This is achieved by adding the `"disable-redzone": true` line to our target configuration file.

#### SIMD Extensions
The `features` field enables/disables target features. We disable the `mmx` and `sse` features by prefixing them with a minus and enable the `soft-float` feature by prefixing it with a plus.  The `mmx` and `sse` features determine support for [Single Instruction Multiple Data (SIMD)] instructions, which simultaneously perform an operation (e.g. addition) on multiple data words. The `x86` architecture supports the following standards:

[Single Instruction Multiple Data (SIMD)]: https://en.wikipedia.org/wiki/SIMD

- [MMX]: The _Multi Media Extension_ instruction set was introduced in 1997 and defines eight 64 bit registers called `mm0` through `mm7`. These registers are just aliases for the registers of the [x87 floating point unit].
- [SSE]: The _Streaming SIMD Extensions_ instruction set was introduced in 1999. Instead of re-using the floating point registers, it adds a completely new register set. The sixteen new registers are called `xmm0` through `xmm15` and are 128 bits each.
- [AVX]: The _Advanced Vector Extensions_ are extensions that further increase the size of the multimedia registers. The new registers are called `ymm0` through `ymm15` and are 256 bits each. They extend the `xmm` registers, so e.g. `xmm0` is the lower (or upper?) half of `ymm0`.

[MMX]: https://en.wikipedia.org/wiki/MMX_(instruction_set)
[x87 floating point unit]: https://en.wikipedia.org/wiki/X87
[SSE]: https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions
[AVX]: https://en.wikipedia.org/wiki/Advanced_Vector_Extensions

By using such SIMD standards, programs can often speed up significantly. Good compilers are able to transform normal loops into such SIMD code automatically through a process called [auto-vectorization].

[auto-vectorization]: https://en.wikipedia.org/wiki/Automatic_vectorization

However, the large SIMD registers lead to problems in OS kernels. The reason is that the kernel has to backup all registers that it uses on each hardware interrupt (we will look into this in the [“Handling Exceptions”] post). So if the kernel uses SIMD registers, it has to backup a lot more data, which noticably decreases performance. To avoid this performance loss, we disable the `sse` and `mmx` features (the `avx` feature is disabled by default).

As noted above, floating point operations on `x86_64` use SSE registers, so floats are no longer usable without SSE. Unfortunately, the Rust core library already uses floats (e.g., it implements traits for `f32` and `f64`), so we need an alternative way to implement float operations. The `soft-float` feature solves this problem by emulating all floating point operations through software functions based on normal integers.

### Compiling
To build our kernel for our new target, we pass the configuration file's name as `target` argument:

```bash
cargo build --target=x86_64-blog_os
```

However, the following error occurs:

```
error[E0463]: can't find crate for `core`
  |
  = note: the `x86_64-blog_os` target may not be installed
```

The error tells us that the Rust compiler no longer finds the core library. The [core library] is implicitly linked to all `no_std` crates and contains things such as `Result`, `Option`, and iterators.

[core library]: https://doc.rust-lang.org/nightly/core/index.html

The problem is that the core library is distributed together with the Rust compiler as a _precompiled_ library. So it is only valid for the host triple (e.g., `x86_64-unknown-linux-gnu`) but not for our custom target. If we want to compile code for other targets, we need to recompile `core` for these targets first.

#### Xargo
That's where [xargo] comes in. It is a wrapper for cargo that eases cross compilation. We can install it by executing:

[xargo]: https://github.com/japaric/xargo

```
cargo install xargo
```

Xargo depends on the rust source code, which we can install with `rustup component add rust-src`.

Xargo is “a drop-in replacement for cargo”, so every cargo command also works with `xargo`. You can do e.g. `xargo --help`, `xargo clean`, or `xargo doc`. However, the `build` command gains additional functionality: `xargo build` will automatically cross compile the `core` library when compiling for custom targets.

Let's try it:

```bash
> xargo build --target=x86_64-blog_os
   Compiling core v0.0.0 (file:///…/rust/src/libcore)
TODO
```

It worked! We just successfully cross-compiled our kernel for our new custom target. We can now find a static library at `target/x86_64-blog_os/debug/libblog_os.a`, which can be linked with our assembly kernel.

## Linking Rust

### Adjusting the Makefile
To build and link the rust library on `make`, we extend our `Makefile`([full file][github makefile]):

```make
# ...
target ?= $(arch)-blog_os
rust_os := target/$(target)/debug/libblog_os.a
# ...
$(kernel): kernel $(rust_os) $(assembly_object_files) $(linker_script)
	@ld -n -T $(linker_script) -o $(kernel) \
		$(assembly_object_files) $(rust_os)

kernel:
       @xargo build --target $(target)
```
We added a new `kernel` target that just executes `xargo build` and modified the `$(kernel)` target to link the created static lib .

But now `xargo build` is executed on every `make`, even if no source file was changed. And the ISO is recreated on every `make iso`/`make run`, too. We could try to avoid this by adding dependencies on all rust source and cargo configuration files to the `kernel` target, but the ISO creation takes only half a second on my machine and most of the time we will have changed a Rust file when we run `make`. So we keep it simple for now and let cargo do the bookkeeping of changed files (it does it anyway).

[github makefile]: https://github.com/phil-opp/blog_os/blob/set_up_rust/Makefile

### Calling Rust
Now we can call the main method in `long_mode_start`:

```nasm
bits 64
long_mode_start:
    ; call the rust main
    extern rust_main     ; new
    call rust_main       ; new

    ; print `OKAY` to screen
    mov rax, 0x2f592f412f4b2f4f
    mov qword [0xb8000], rax
    hlt
```
By defining `rust_main` as `extern` we tell nasm that the function is defined in another file. As the linker takes care of linking them together, we'll get a linker error if we have a typo in the name or forget to mark the rust function as `pub extern`.

If we've done everything right, we should still see the green `OKAY` when executing `make run`. That means that we successfully called the Rust function and returned back to assembly.

### Fixing Linker Errors
Now we can try some Rust code:

```rust
pub extern fn rust_main() {
    let x = ["Hello", "World", "!"];
    let y = x;
}
```
When we test it using `make run`, it fails with `undefined reference to 'memcpy'`. The `memcpy` function is one of the basic functions of the C library (`libc`). Usually the `libc` crate is linked to every Rust program together with the standard library, but we opted out through `#![no_std]`. We could try to fix this by adding the [libc crate] as `extern crate`. But `libc` is just a wrapper for the system `libc`, for example `glibc` on Linux, so this won't work for us. Instead we need to recreate the basic `libc` functions such as `memcpy`, `memmove`, `memset`, and `memcmp` in Rust.

[libc crate]: https://doc.rust-lang.org/nightly/libc/index.html

#### rlibc
Fortunately there already is a crate for that: [rlibc]. When we look at its [source code][rlibc source] we see that it contains no magic, just some [raw pointer] operations in a while loop. To add `rlibc` as a dependency we just need to add two lines to the `Cargo.toml`:

```toml
...
[dependencies]
rlibc = "0.1.4"
```
and an `extern crate` definition in our `src/lib.rs`:

```rust
...
extern crate rlibc;

#[no_mangle]
pub extern fn rust_main() {
...
```
Now `make run` doesn't complain about `memcpy` anymore. Instead it will show a pile of new errors:

```
target/debug/libblog_os.a(core-35017696.0.o):
    In function `ops::f32.Rem::rem::hfcbbcbe5711a6e6emxm':
    core.0.rs:(.text._ZN3ops7f32.Rem3rem20hfcbbcbe5711a6e6emxmE+0x1):
    undefined reference to `fmodf'
target/debug/libblog_os.a(core-35017696.0.o):
    In function `ops::f64.Rem::rem::hbf225030671c7a35Txm':
    core.0.rs:(.text._ZN3ops7f64.Rem3rem20hbf225030671c7a35TxmE+0x1):
    undefined reference to `fmod'
...
```

[rlibc]: https://crates.io/crates/rlibc
[rlibc source]: https://github.com/rust-lang/rlibc/blob/master/src/lib.rs
[raw pointer]: https://doc.rust-lang.org/book/raw-pointers.html
[crates.io]: https://crates.io

#### --gc-sections
The new errors are linker errors about missing `fmod` and `fmodf` functions. These functions are used for the modulo operation (`%`) on floating point numbers in [libcore]. The core library is added implicitly when using `#![no_std]` and provides basic standard library features like `Option` or `Iterator`. According to the documentation it is “dependency-free”. But it actually has some dependencies, for example on `fmod` and `fmodf`.

[libcore]: https://doc.rust-lang.org/core/

So how do we fix this problem? We don't use any floating point operations, so we could just provide our own implementations of `fmod` and `fmodf` that just do a `loop{}`. But there's a better way that doesn't fail silently when we use float modulo some day: We tell the linker to remove unused sections. That's generally a good idea as it reduces kernel size. And we don't have any references to `fmod` and `fmodf` anymore until we use floating point modulo. The magic linker flag is `--gc-sections`, which stands for “garbage collect sections”. Let's add it to the `$(kernel)` target in our `Makefile`:

```make
$(kernel): cargo $(rust_os) $(assembly_object_files) $(linker_script)
	@ld -n --gc-sections -T $(linker_script) -o $(kernel) \
		$(assembly_object_files) $(rust_os)
```
Now we can do a `make run` again and… it doesn't boot anymore:

```
GRUB error: no multiboot header found.
```
What happened? Well, the linker removed unused sections. And since we don't use the Multiboot section anywhere, `ld` removes it, too. So we need to tell the linker explicitely that it should keep this section. The `KEEP` command does exactly that, so we add it to the linker script (`linker.ld`):

```
.boot :
{
    /* ensure that the multiboot header is at the beginning */
    KEEP(*(.multiboot_header))
}
```
Now everything should work again (the green `OKAY`). But there is another linking issue, which is triggered by some other example code.

#### panic = "abort"

The following snippet still fails:

```rust
    ...
    let test = (0..3).flat_map(|x| 0..x).zip(0..);
```
The error is a linker error again (hence the ugly error message):

```
target/debug/libblog_os.a(blog_os.0.o):
    In function `blog_os::iter::Iterator::zip<core::iter::FlatMap<
        core::ops::Range<i32>, core::ops::Range<i32>, closure>,
        core::ops::RangeFrom<i32>>':
    /home/.../src/libcore/iter.rs:654:
    undefined reference to `_Unwind_Resume'
```
So the linker can't find a function named `_Unwind_Resume` that is referenced in `iter.rs:654` in libcore. This reference is not really there at [line 654 of libcore's `iter.rs`][iter.rs:654]. Instead, it is a compiler inserted _landing pad_, which is used for panic handling.

[iter.rs:654]: https://github.com/rust-lang/rust/blob/b0ca03923359afc8df92a802b7cc1476a72fb2d0/src/libcore/iter.rs#L654

By default, the destructors of all stack variables are run when a `panic` occurs. This is called _unwinding_ and allows parent threads to [recover from panics]. However, it requires a platform specific gcc library, which isn't available in our kernel.

[recover from panics]: https://doc.rust-lang.org/book/concurrency.html#panics

Fortunately, Rust allows us to disable unwinding. We just need to add some entries in our `Cargo.toml`:

```toml
# The development profile, used for `cargo build`.
[profile.dev]
panic = "abort"

# The release profile, used for `cargo build --release`.
[profile.release]
panic = "abort"
```

These [profile sections] specify options for `cargo build` and `cargo release`. By setting the `panic` option to `abort`, we disable all unwinding in our kernel.

[profile sections]: http://doc.crates.io/manifest.html#the-profile-sections

However, there are still references to `_Unwind_Resume` in the precompiled standard libraries. This might lead to linker errors when we use specific parts of `libcore`. To avoid this, we create a dummy `_Unwind_Resume` function that loops indefinitely[^fn-libcore-unwind]:

[^fn-libcore-unwind]: A better solution is to recompile `libcore` with `panic="abort"`. We will do this in a future post.

```rust
// in src/lib.rs

#[allow(non_snake_case)]
#[no_mangle]
pub extern "C" fn _Unwind_Resume() -> ! {
    loop {}
}
```

Now we fixed all linking issues and our kernel builds again. But instead of displaying `Hello World`, it constantly reboots itself when we start it. TODO

## Hello World!
Finally, it's time for a `Hello World!` from Rust:

```rust
pub extern fn rust_main() {
    // ATTENTION: we have a very small stack and no guard page

    let hello = b"Hello World!";
    let color_byte = 0x1f; // white foreground, blue background

    let mut hello_colored = [color_byte; 24];
    for (i, char_byte) in hello.into_iter().enumerate() {
        hello_colored[i*2] = *char_byte;
    }

    // write `Hello World!` to the center of the VGA text buffer
    let buffer_ptr = (0xb8000 + 1988) as *mut _;
    unsafe { *buffer_ptr = hello_colored };

    loop{}
}
```
Some notes:

- The `b` prefix creates a [byte string], which is just an array of `u8`
- [enumerate] is an `Iterator` method that adds the current index `i` to elements
- `buffer_ptr` is a [raw pointer] that points to the center of the VGA text buffer
- Rust doesn't know the VGA buffer and thus can't guarantee that writing to the `buffer_ptr` is safe (it could point to important data). So we need to tell Rust that we know what we are doing by using an [unsafe block].

[byte string]: https://doc.rust-lang.org/reference.html#characters-and-strings
[enumerate]: https://doc.rust-lang.org/nightly/core/iter/trait.Iterator.html#method.enumerate
[unsafe block]: https://doc.rust-lang.org/book/unsafe.html

### Stack Overflows
Since we still use the small 64 byte [stack from the last post], we must be careful not to [overflow] it. Normally, Rust tries to avoid stack overflows through _guard pages_: The page below the stack isn't mapped and such a stack overflow triggers a page fault (instead of silently overwriting random memory). But we can't unmap the page below our stack right now since we currently use only a single big page. Fortunately the stack is located just above the page tables. So some important page table entry would probably get overwritten on stack overflow and then a page fault occurs, too.

[stack from the last post]: {{% relref "02-entering-longmode.md#creating-a-stack" %}}
[overflow]: https://en.wikipedia.org/wiki/Stack_overflow

## What's next?
Until now we write magic bits to some memory location when we want to print something to screen. In the [next post] we create a abstraction for the VGA text buffer that allows us to print strings in different colors and provides a simple interface.

[next post]: {{% relref "04-printing-to-screen.md" %}}
