---
title: Getting Forem running on Apple Silicon
date: 2021-01-05
tags: [apple, forem, ruby]
image: https://joshpuetz.com/assets/images/covers/2021-01-05.jpg
excerpt: Gotta go fast
---


## Intro

I recently bought an Apple Silicon based Mac Mini for my home office setup, and was excited to see if I could do development for with Forem on it. After a little bit of experimentation I have [Forem](https://github.com/forem/forem) running locally, and wanted to document the steps I took in case anybody else is curious or looking to make the switch.

The executive summary: it's very possible to develop and run Forem on Apple Silicon without many changes, even at this early stage.

## Some Background

First a bit of background: my day to day development environment is a 2019 15" MacBook Pro with a 2.3GHz 8-core i9 processor and 16 GB of memory. I develop Forem locally using the installation instructions for MacOS at https://docs.forem.com/installation/mac/

While my MacBook Pro (and all modern Macs before it) have Intel processors, the new Apple Silicon based Macs run on an ARM based M1 processor. Intel processors run executables compiled for x86 architecture, and the M1 runs code compiled for ARM64 architectures. These two processors basically speak difference languages.

It would be an huge problem if the Apple Silicon Macs couldn't run x86 code as that would mean the last 20 some years of Mac software would be incompatible with them. As a stopgap measure Apple has a software translation layer called Rosetta 2 which can "translate" x86 executables on the fly to ARM64 code (important note: I am not a low level software engineer and this is a huge over simplification of Rosetta 2. Apple's [documentation about Rosetta 2](https://developer.apple.com/documentation/apple_silicon/about_the_rosetta_translation_environment) leaves a lot unexplained).

As an absolute low bar we could just run all of Forem (and associated development tools) under Rosetta 2 and call it a day.  There's a performance penalty, but the M1 chip is so fast that even under Rosetta 2 I'm finding most processes run just as fast as they did on my Intel based MacBook Pro.

## The Simple Way

You can run any process in Terminal via Rosetta 2 by prepending the command `arch -x86_64`. So the easiest way to setup Forem on M1 is to just install the x86 version of everything.

Install Homebrew with Rosetta 2:

```
$ arch -x86_64 /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install.sh)"
```

Then you can run Homebrew in in x86 mode by using: 

```
$ arch -x86_64 brew
```

Typing that every times gets old really fast, so I created an alias in my `~/.zshrc` file:

```
alias ibrew='arch -x86_64 /usr/local/bin/brew'
```
(the i is for Intel!)

Proceed with the rest of the [installation instructions](https://docs.forem.com/installation/mac/), and you're done! Remember to use the `ibrew` alias so Homebrew installs the x86 version of packages.


## The Complicated way

Sure, you can just run everything in x86 mode, but where is that fun in that: I bought an Apple Silicon Mac for it's incredible performance, so let's try to install as much as possible in the Arm64 architecture!

There's [early initial support for Apple Silicon in Homebrew](https://github.com/Homebrew/brew/pull/9117), but they recommend installing both the x86 and Arm versions side by side as many Homebrew recipes haven't been updated for Apple Silicon yet. I used Sam Soffe's [excellent write up](https://soffes.blog/homebrew-on-apple-silicon) to install both the x86 and Arm versions of Homebrew side by side. 

After installing the x86 version of Homebrew, we'll install Homebrew **without** the x86 prefix (which installs it as a native Arm64 executable), but we'll do so in a different place than the default `/usr/local/bin` so we don't overwrite our x64 Homebrew:

```
$ sudo mkdir -p /opt/homebrew
$ sudo chown -R $(whoami):staff /opt/homebrew
$ cd /opt
$ curl -L https://github.com/Homebrew/brew/tarball/master | tar xz --strip 1 -C homebrew
```

Finally, we'll put this Arm64 Homebrew on our path before the x86 version. Back in our `~/.zshrc` file:

```
export PATH="/opt/homebrew/bin:/usr/local/bin:$PATH"
```

Now if we want to install an x86 version of a package we'll use `ibrew`, and if we want the Arm64 version of a package we'll use `brew`.

Working through the Forem development environment setup instructions, as of January 5, 2021 I was able to use native Arm64 versions of everything **except** ElasticSearch and Ruby.

ElasticSearch is dependent on OpenJDK, which doesn't have an Apple Silicon version ready yet.

Ruby works on Apple Silicon, as do most gems. The problem is any gem that has "native extensions" needs that extension to be built as an Arm64 version if you're going to run it on Arm64 Ruby. Most do, but I ran into a few older gems that just wouldn't build:
- [CLD](https://github.com/jtoy/cld) is a very old gem used to language detection that just has no idea how to build on Apple Silicon. @fdoxyz played around with swapping it out for a new version in his great writeup on [getting Forem to run on a Raspberry PI](https://forem.dev/fdoxyz/how-to-get-forem-to-run-on-arm-5hea).
- Several gems (notably the [Twitter](https://github.com/sferik/twitter) gem) use an older version of the [http-parser](https://github.com/cotag/http-parser) gem that doesn't build under Apple Silicon. [Http-parser](https://github.com/cotag/http-parser) was just updated yesterday, so this might not be an issue any longer.
- I was able to install the mini_racer gem successfully under Arm64, but ran into the error `dyld: lazy symbol binding failed: Symbol not found` when trying to actually run Rails.

Given these gem issues I'm using the x86 version of Ruby. While Ruby and friends do work just fine under Rosetta 2, there are a few idiosyncrasies because we have two versions of Homebrew install. First: I had both the Arm64 and x86 versions of the `readline` and `openssl` libraries installed. Since I'm building an x86 Ruby, I want it to use the x86 versions. To force `ruby-build` to use the proper libraries, I added the following to my `~/.zshrc`:

```
export RUBY_CONFIGURE_OPTS="--with-openssl-dir=$(ibrew --prefix openssl@1.1) --with-readline-dir=$(ibrew --prefix readline)"
export LDFLAGS="-L/usr/local/opt/openssl@1.1/lib"
export CPPFLAGS="-I/usr/local/opt/openssl@1.1/include"
```

If you installed the Arm64 version PostgreSQL you'll be missing the x86 version of `libpq`. I solved this by installing the x86 version of `libpq`, then adding it to the beginning of my `PATH`:

```
brew install libpq
```

Then in `~/.zshrc`:
```
export PATH="/usr/local/opt/libpq/bin:/opt/homebrew/bin:/usr/local/bin:$PATH"
```
 
Hopefully in the next few weeks these gems will be updated, or the Forem community will work on replacing/forking them.

And Docker fans...no, it's not ready yet, but it's in [preview](https://www.docker.com/blog/download-and-try-the-tech-preview-of-docker-desktop-for-m1/). Stay tuned.


## Looking forward

Expect ** a lot** of changes in the coming weeks. I expect it won't take too long for Arm versions of most Homebrew packages and native gems to be available. Two great sites for checking progress are [Is Apple Silicon Ready?](https://isapplesiliconready.com) and [Does It Arm?](https://doesitarm.com).

If you run into any problems developing or running Forem on Apple Silicon, please open an issue on our [issue board](https://github.com/forem/forem/issues). 
