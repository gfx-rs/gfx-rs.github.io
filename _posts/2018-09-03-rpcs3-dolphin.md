---
layout: post
title: RPCS3 and Dolphin on macOS using gfx-portability
---

[gfx-rs](https://github.com/gfx-rs/gfx) is a Rust project aiming to make graphics programming more accessible and portable, focusing on exposing a universal Vulkan-like API. It's a single Rust API with multiple backends that implement it: Direct3D 12/11, Metal, Vulkan, and even OpenGL. We are also building a Vulkan Portability implementation ([gfx-portability](https://github.com/gfx-rs/portability)) based on it, which allows non-Rust applications using Vulkan to run everywhere. This post is focused on the Metal backend only.

## Expanding Portability Testing

After improving functionality and performance of _gfx-portability_'s Metal backend through [benchmarking Dota2](http://gfx-rs.github.io/2016/01/22/pso.html), and verifying certain functionality through the Vulkan [Conformance Test Suite](https://github.com/KhronosGroup/VK-GL-CTS/blob/master/external/vulkancts/README.md) (CTS), we decided to expand our testing to other projects. We were especially interested in finding projects matching the following criteria:

* Open-source: allows us to easily debug issues and add or modify existing macOS window and surface support
* Already using Vulkan for rendering: allows us to simply load gfx-portability on macOS in place of a Vulkan driver on other platforms, because gfx-portability implements `vulkan.h`
* Lacking strong macOS/Metal support: potentially enables macOS users to be able to run these projects natively for the first time

Fortunately, we quickly found two projects which matched our criteria: [RPCS3](https://github.com/RPCS3/rpcs3) and [Dolphin](https://github.com/dolphin-emu/dolphin). Similar to our experience with running Dota2 for the first time, we immediately hit some missing functionality in gfx-portability, and discovered various issues with the projects.

## RPCS3

[RPCS3](https://github.com/RPCS3/rpcs3) is an open-source Sony PlayStation 3 emulator and debugger written in C++ for Windows and Linux. RPCS3 has a Vulkan backend, and some attempts were made to support macOS previously.

The gfx-rs team [started macOS integration](https://github.com/RPCS3/rpcs3/pull/4996) by adding surface and swapchain support, and in the process identified a number of blockers in both gfx-rs and RPCS3. The RPCS3 developers were extremely helpful, and both teams were able to collaborate to quickly address the blockers. With the blockers addressed, we were able to render gameplay within RPCS3.

![RPCS3 Cube](/img/rpcs3-cube.png)
![RPCS3 Scogger](/img/rpcs3-scogger.png)

## Dolphin

[Dolphin](https://github.com/dolphin-emu/dolphin) is an emulator for two recent Nintendo video game consoles: the GameCube and the Wii. We noticed Dolphin was [actively working on adding support for macOS](https://github.com/dolphin-emu/dolphin/pull/7039), so we took the opportunity to test it with gfx-portability.

While testing Dolphin, we noticed some further minor bugs in gfx. Fortunately we were able to quickly address these issues, and soon begin to render real gameplay. The Dolphin developers have been actively working on additional improvements to support macOS and we're excited to help this support to progress even more.

![Dophin Crash Bandicoot](/img/dolphin-crash-bandicoot.png)
![Dophin Paper Mario](/img/dolphin-paper-mario.png)
![Dophin Smash Bros](/img/dolphin-smash-bros.png)
![Dophin Mario Kart](/img/dolphin-mario-kart.png)
![Dolphin Metroid](/img/dolphin-metroid.png)

## Adding Continuous Releases

For anyone wanting to try _gfx-portability_ for themselves, we have started automatically releasing binaries under the portability repository under the GitHub [`latest`](https://github.com/gfx-rs/portability/releases) release. We create these automatically by [running continuous integration against the `latest` tag](https://github.com/gfx-rs/portability/pull/142), which will build and upload binaries to the corresponding release. Currently we produce MacOS (Metal) and Linux (Vulkan) binaries, and plan to add Windows (Direct3D 12/11 and Vulkan) binaries soon as well.

We added these releases so users don't have to buid _gfx-portability_ themselves in order to test it with an existing project. The binaries are compatible with both the Vulkan loader on macOS (as a typical Installable Client Driver expected by the Vulkan loader) and by linking the binaries directly from an application (the application calls functions from `vulkan.h` as usual).

## Conclusions

The initial focus on specific areas of CTS coverage and running Dota2 has proven to be a success. We were able to run RPCS3 and Dolphin on top of _gfx-portability_'s Metal backend and only had to address some minor issues in the process. As we continue to test additional real world use cases and expand CTS coverage, we expect stability and performance to continue to improve.

We gained great feedback while assisting RPCS3 and Dolphin to run well on macOS, and we look forward to helping other projects in the future as well. There are many projects only target Vulkan or OpenGL for rendering, yet would still like to be portable to other platforms, and the Vulkan Portability initiative aims to make this possible.
