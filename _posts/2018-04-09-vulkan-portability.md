---
layout: post
title: Bringing Vulkan everywhere
---

gfx-rs pursues to provide a graphics API around the current open graphics libraries (Vulkan, D3D11/12, Metal and OpenGL). [Last year](http://gfx-rs.github.io/2017/12/30/this-year.html) we realized that we want to provide a low-level API, and it's reasonable to mimic Vulkan, since it is an existing API with detailed specification and in our analysis regarding mapping from D3D12 and Metal it seemed possible. This also matches the goal of the [Vulkan Portability Initiative](https://www.khronos.org/blog/khronos-announces-the-vulkan-portability-initiative) to define a subset of Vulkan which can be efficiently implemented on top of D3D12 and Metal. We started experimenting with a [C wrapper](https://github.com/gfx-rs/portability) that implements "vulkan.h" on a target system. However, getting from a hacked prototype to a production-ready library is most difficult (spoiler: we aren't there yet).

### MoltenVK

In the meantime, MoltenVK has been [open-sourced](https://www.khronos.org/news/press/vulkan-applications-enabled-on-apple-platforms) with Valve's help. This changed the landscape of portability and the perception of Vulkan by game developers. Some of the alternative Vulkan on Metal implementations [got deprecated](https://github.com/Chabloom/VulkanOnMetal/commit/979dc181b414c607fd1847ef2fd2e0ed56e00005), but gfx-rs portability is still moving forward. We've been working in Vulkan Portability TSG (technical subgroup) from day one, contributing our research and discussing the portability issues. Our Metal backend is not nearly at the same quality level as MoltenVK yet, but we see a few ways where we can excel in terms of performance, as well as benefit from low maintenance cost of pure-Rust codebase in the long run.

### Vulkan CTS

So, supposing we want to ship a Vulkan portability library one day. How do we know it's ready? By passing the official [Conformance Test Suite](https://github.com/KhronosGroup/VK-GL-CTS/blob/master/external/vulkancts/README.md), of course. We got the CTS ported to OSX and have been [tracking the progress](https://github.com/gfx-rs/portability/tree/014ad5476045efe805271d623e5cdf87d992ee9c#vulkan-cts-coverage) on all three major backends (Vulkan, D3D12, Metal). As of today, we aren't reaching the end of CTS, panicking in the middle, but we pass over a thousand of tests on all backends.
Our goal will be to get the whole CTS running trough and gradually improve the pass rate. It's still a long way until we pass the over 250k+ tests!

We have a super talented team of volunteer contributors, but there is still so much work to be done that we feel overwhelmed with it and would be happy to accept any help. Good thing is - there is a clear goal to reach, which we can measure quantitatively, and the work can be split effectively. Let the gfx-rs implementation period kick off!
