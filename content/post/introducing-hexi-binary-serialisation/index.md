---
title: "Introducing Hexi, a lightweight C++23 binary serialisation library"
description: Emulators need to handle networking data. Hexi is how it's done in Ember.
slug: introducing-hexi-binary-serialisation
date: 2025-03-27 16:00:00+0000
author: Chaosvex
image: cover.jpg
categories:
    - Programming
tags:
    - programming
    - performance
    - networking
draft: false
weight: 1
---

Like any respectable MMORPG server emulator, Ember needs to be able to handle network data, in and out. Over the years, a small collection of utilities has been built up to ensure any network data handling can be done quickly, effectively and perhaps most importantly, safely.

That collection of utilities has now been spun out into its own dependency-free (minus the stdlib) library, available for drop-in use by other projects that want to be able to handle with minimal fuss and effort. Here are the key points:

* Modern, C++23!
* Cross-platform, tested with Clang, GCC and MSVC.
* Unit tested.
* Header-only. Available as a single header or a collection of headers for picking and choosing the bits you want to use.
* CMake integration if you'd prefer not to drop the headers into your own project.
* Provides multiple useful buffer structures to help shuffle those bytes around efficiently.
* Provides allocators and Asio integration for when you feel the need for speed.
* Endian utilities for ensuring your bytes are all in order.

If this resonates with you, check out more details over at http://github.com/EmberEmu/Hexi.