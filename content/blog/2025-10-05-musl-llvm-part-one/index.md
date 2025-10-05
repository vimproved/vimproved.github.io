+++
title = "The musl+llvm Diaries, Part One: Introduction"
date = "2025-10-05"
+++

---

If you've spent enough time interacting with the Gentoo community, you've probably at least heard of **musl** or
**LLVM**. I often see relative newcomers to Gentoo asking things like "Should I use musl?" or "Should I use LLVM?" or
(god forbid) even "Should I use musl+llvm?". There's a lot of misinformation about this topic, so I've decided to start
a new series of blog posts to better inform people on the topic of using musl, LLVM, or the combination of the two with
Gentoo. It should also serve as a sort of devlog for people interested in the technical elements of making fixes for
these alternative platforms.

![krita-shopped poster for "The Apothecary Diaries", but the title says "The musl+llvm Diaries". Jinshi's face is
replaced with the musl logo while Maomao's is replaced with the LLVM logo.](the-musl-llvm-diaries.png)

---

## Should I use musl+llvm with Gentoo?

<span style="font-size: 96px;">**NO.**</span>

Seriously, no. It is not worth it for 99% of people, and that almost definitely includes you. Stick to a normal system
configuration. If you've seen this and you're still not deterred, here is a list of things that I personally expect you
to be able to do if you are running a Gentoo musl+llvm system:

- Inspect build failures by means including but not limited to:
  - Looking around in headers
  - Checking manpages for functions
  - Using a search engine to find similar errors that have been reported
- Write high-quality, portable patches to fix build failures
- Excepting that, file high-quality bug reports with build logs and the output of `emerge --info`
- Send fixes to both the main Gentoo repository and to upstreams
  - Explain to upstreams what causes the build failure, and how you fixed it
  - Explain to upstreams why musl and LLVM support is important if needed
  - Keep a respectful and cool-headed tone, no matter how rude upstream acts
- Accept that some things will never work on musl+llvm without a massive effort by specialists in that specific software
  (e.g. dotnet)

If you follow all of these instructions, you will have a functioning system and the satisfaction of knowing you have
made a contribution to the open source ecosystem. That's it. You will not get anything else out of this. If you are fine
with doing all of these things, then it's up to you whether you think a musl+llvm system is worth it.

---

## First of all: what?

Before I jump into anything more complicated, I'd like to take some time explaining what musl is, what LLVM is, and what
it means to use musl+llvm on Gentoo. This may sound trivial, but there's seemingly still confusion over the terminology
and what it means for your system.

### musl

![the musl logo.](musl.png)

At the heart of more or less every operating system we have today lies a crucial part of its functionality: the C
standard library. Here in Linux world, the vast majority of distributions use
[glibc](https://www.gnu.org/software/libc/), the GNU C Standard Library. While every implementation of the C standard
library is bound by the C language standards as defined by ISO's WG14, there is no limitation on any extra features or
functions that the C standard library may contain. These extra features and interfaces are commonly called
**extensions**, and glibc has a lot of them.

glibc tends to take the "kitchen sink" approach to extensions, packing a lot of functionality into their standard
library. For an example of this, look no further than `malloc_info`. `malloc_info` is a function that exports an XML
string describing the current state of memory allocation in the system, a piece of functionality that would normally not
be expected of the C standard library. Now, I would like to emphasize that _this is not inherently a bad thing_. There
are situations where glibc-specific extensions can be extreme helpful to certain software.

On the other hand, musl takes an extremely conservative route with what they implement. They aim to stick close to the
standard, and only implement extensions that have become ubiquitous beyond glibc. This leads to it being much smaller,
which is useful for a couple purposes: the most obvious and widespread use case is static linking, as statically linking
binaries to glibc can be a massive pain. Another significant boon given by musl's diminuative size is security: since
musl is smaller and saddled with less technical debt, it is much more auditable and is easier for a single person to
understand.

However, musl has its downsides. The first and most significant of them for most users is software compatibility. Almost
all proprietary software for Linux is built and _dynamically linked_ against glibc. This means that it will not run on
musl, as it is not [**ABI compatible**](https://en.wikipedia.org/wiki/Application_binary_interface) with glibc. However,
even if the software you're looking to run is open source, you may still run into issues. Chances are that the software
you're looking to run only provides builds for glibc, and they possibly have never even tried building it on musl. This
can lead many developers to use glibc extensions without implementing proper build system checks for them, meaning that
their software will not compile on musl. Additionally, while musl touts being fast, there are many areas where it can
unfortunately perform _worse_ than glibc. The most significant area is in its `malloc` implementation, which just plain
kinda sucks. As the developers of Chimera Linux put it, "the stock allocator is the primary reason for nearly all
performance issues people generally have with musl" ([source](https://chimera-linux.org/docs/configuration/musl)).

## LLVM

![the LLVM logo.](llvm.png)

Now, on to the more complicated topic of the two: LLVM. LLVM's logo is a dragon, however it perhaps might be more apt to
represent it as a hydra. LLVM is a massive software toolchain project containing many different individual interlocking
pieces, which can be used together or standalone. In the case of Gentoo's LLVM profile, it means using the full LLVM
toolchain in lieu of their GNU alternatives.

### LLVM (The Library)

At the core of LLVM is a library named... LLVM. Not confusing at all, I know. The core LLVM library focuses on
interacting with the LLVM IR (Intermediate Representation) format, which is essentially an abstraction of
platform-specific machine code (but less abstracted than any high-level programming language). LLVM's job is to optimize
LLVM IR and then convert it into platform-specific binaries. Pretty much every toolchain takes this approach, but LLVM
is special in that it exposes itself as a shared library, meaning that compilers for other languages (e.g. Rust) can use
it and forgo the effort of having to write all that stuff themselves.

### clang (In Place of GCC)

clang is the most well-known element of the LLVM toolchain, being LLVM's C and C++ compiler implementation. For a while,
it was behind GCC in terms of performance and features, however in the past 5 or so years it has more or less caught up.
Many Gentoo users build their systems with clang, even while still using glibc and the rest of the GNU toolchain. It
used to be that building all of the software for a distribution with clang was a tall order, however nowadays it's quite
rare to encounter an issue stemming from clang (aside from periods directly after the release of a new version).

### LLD (In Place of BFD)

When you compile a C file, it does a 1 to 1 compilation of the source files to output files containing machine code. But
if that's the case, how do projects use multiple files of source code to produce one final binary? That's where the
linker comes in. It takes multiple object files containing machine code and links them together into a final binary,
reconciling their symbols and roping in any shared libraries that are requested to be linked against.

The GNU toolchain's primary linker is called ld.bfd. It works. Slowly. The LLVM toolchain's linker is called ld.lld (or
just LLD for short). It is significantly faster than ld.bfd, but it does have some differences in behaviour in specific
situations, meaning it can cause build failures on occasion.

### compiler-rt & libunwind (In Place of libgcc)

There are certain features (such as architecture-specific instruction set extensions) that are more appropriate to be
implemented by the compiler as opposed to the C standard library. In GCC, these are implemented in a library called
libgcc. LLVM's version of this is compiler-rt and libunwind. Without diving too much into the weeds of what they do
(it's ultimately not particularly relevant), compiler-rt generally maintains compatibility with libgcc, however note
that I say "generally" and not "always." You will sometimes find issues caused by compiler-rt, but they're relatively
rare.

### libc++ (In Place of libstdc++)

Just like C, C++ needs a standard library. In C++, they call it the **STL** (Standard Template Libraries). GCC's
implementation of the C++ STL is called libstdc++, and it's more or less used on every Linux system. LLVM's
implementation of the C++ STL is called libc++, and it's almost never used on Linux. In fact, Gentoo is one of two
distros I know that even remotely supports it. You may be asking "okay, so where is libc++ used then?". The most
prominent operating system that uses libc++ is MacOS, however the version it ships is typically a year or two behind the
latest. In addition to MacOS, both FreeBSD and OpenBSD use libc++.

libstdc++ and libc++ have two primary causes of incompatibilities. Firstly, there are some situations where libstdc++ is
perhaps a little bit more lenient than it should be with the standard, while libc++ is more strict. For example, libc++
deprecated and later removed the generic template definition for `std::char_traits<T>` because it was not strictly in
the standard and had unpredictable behaviour
([link](https://discourse.llvm.org/t/deprecating-std-string-t-for-non-character-t/66779)). This caused a lot of
breakage. The second cause of incompatibilities is due to featureset implementation. In general, libstdc++ is typically
faster to implement new C++ features than libc++, meaning that software which springs to use the shiny new feature is
liable to not compile on libc++.

Just like the C standard libraries, libc++ is not ABI compatible with libstdc++, meaning that a piece of software built
and linked against libstdc++ will not run on a system using libc++. In practice, libc++ ends up causing the most issues
out of any of the elements of the LLVM toolchain when used on a system level, and it does not provide any benefits.

The LLVM profiles on Gentoo use all of these components in place of their GNU counterparts, and when I refer to an LLVM
system or a musl+llvm system, I am referring to one using the components listed above.

---

## So like, why though?

I've just spent the entire first part of this article telling you not to use a musl+llvm system and telling you how many
issues they have. But despite that, I still daily-drive a musl+llvm Gentoo installation. Two, actually. I use them to do
schoolwork, play games, watch anime, and more. You may be wondering why?

Well, the answer is simple: ADHD and autism!

![An animated GIF of a flag with Asuka Langley Soryu from Neon Genesis Evangelion on it waving. It is captioned "Mental
Illness".](mental-illness.gif)

Okay, okay. I'll give you a more serious answer. In reality, the reason I daily drive crazy system configurations is
because I love the process of debugging and problem-solving. The satisfaction I get when a fix I made is merged into an
open source project is something nothing else will ever be able to recreate. I enjoy knowing that I can contribute to
the free software ecosystem by simply doing something I find fun and engaging. To that end, I daily-drive musl+llvm to
find these bugs. Additionally, it gives me a sense of accomplishment when I finally fix something and I can use it on my
primary workstation machine. That's it. There are exactly zero technical reasons for using musl+llvm on Gentoo.

I often see relatively new (2-6 months) Gentoo users preaching to absolute newbies about how musl or LLVM is superior
and can enhance your gaming performance or whatever. To you all, I want to say: **STOP**. You are pushing new users down
a difficult path attempting to chase ghosts. I won't pretend I'm some ultimate authority on Gentoo-related topics, but
we're coming up on the two year anniversary of my first pull request to Gentoo, and I've learned a lot since then.

Lastly, if you are really are interested in the potential upsides of using the LLVM toolchain in combination with musl,
I suggest you look into [Chimera Linux](https://chimera-linux.org/). It is a distribution based on musl and LLVM with
some incredibly talented developers behind it. Since they focus specifically on targeting this one system configuration,
they can work quicker than Gentoo when it comes to musl or LLVM fixes.

---

## Next Time on Dragon Ball Z

This post focused on the concept of using the musl libc in tandem with the LLVM toolchain on a higher level, but there's
a lot I've had to do to get everything I use working. In my next posts, I plan to start recounting in depth the issues
that I experienced and how I fixed them, working roughly chronologically (kinda) (maybe).

Until then, see you later!

-- violet
