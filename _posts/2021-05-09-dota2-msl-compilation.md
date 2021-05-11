---
layout: post
title: Shader translation benchmark on Dota2/Metal
---

gfx-rs community's goal is to make graphics programming in Rust easy, fast, and reliable. See [The Big Picture](https://gfx-rs.github.io/2020/11/16/big-picture.html) for the overview, and [release-0.8](https://gfx-rs.github.io/2021/04/30/release-0.8.html) for the latest progress. In this post, we are going to share the first performance metrics of our new pure-Rust shader translation library [Naga](https://github.com/gfx-rs/naga), which is integrated into [gfx-rs](https://github.com/gfx-rs/gfx). Check the [Javelin announcement](https://gfx-rs.github.io/2019/07/13/javelin.html), which was the original name of this project, for the background.

[gfx-portability](https://github.com/gfx-rs/portability) is a [Vulkan Portability](https://www.khronos.org/blog/fighting-fragmentation-vulkan-portability-extension-released-implementations-shipping) implementation in Rust, based on gfx-rs. Previous [Dota2 benchmarks](https://gfx-rs.github.io/2018/08/10/dota2-macos-performance.html) showed good potential in our implementation. However, it couldn't be truly called [an alternative](https://gfx-rs.github.io/2018/04/09/vulkan-portability.html) to [MoltenVK](https://github.com/KhronosGroup/MoltenVK) if it relies on [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross). Today, we are able to run Dota2 with a purely rust Vulkan Portability implementation, thanks to Naga.

## Test

Testing was done on MacBook Pro (13-inch, 2016), which has a humble dual-core Intel CPU running at 3.3GHz. We created an alias to `libMoltenVK.dylib` and pointed `DYLD_LIBRARY_PATH` to it for Dota2 to pick up on boot, thus running on gfx-portability. It was build from [naga-bench-dota](https://github.com/gfx-rs/portability/tree/naga-bench-dota) tag in release. The SPIRV-Cross path was enabled by uncommenting `features = ["cross"]` line in `libportability-gfx/Cargo.toml`.

In-game steps:
 1. launch `make dota-release`
 1. skip the intro videos
 1. proceed to "Heroes" menu
 1. select "Tide Hunter"
 1. and click on "Demo Hero"
 1. walk the center lane, enable the 2nd and 3rd abilities
 1. use the 3rd ability, then quit

![Hero selection screen with Naga (low settings)](/img/dota-naga-hero.jpg)

The point of this short run is to get a bulk of shaders loaded (about 600 graphics pipelines). We are only interested in the CPU cost for loading shaders and creating pipelines. This isn't a test for the GPU time executing the shaders. The only fact about GPU that matters here is that the picture looks identical. We don't expect any architectural changes for potential visual issues to be discovered.

Times were collected using [profiling](https://github.com/aclysma/profiling) instrumentation, which is integrated into gfx-backend-metal. We added this as a temporary dependency to gfx-portability with "profile-with-tracy" feature enabled in order to capture the times in [Tracy](https://github.com/wolfpld/tracy).

In tracy profiles, we'd find the relevant chunks and click on the "Statistics" for them. We are interested in the mean (μ) time and the standard deviation (σ).

## Results

| Function                         | Cross μ | Cross σ | Naga μ | Naga σ |
| -------------------------------- | ------- | ------- | ------ | ------ |
| SPIR-V parsing                   | 0.34ms  | 0.15ms  | 0.45ms | 0.50ms |
| MSL generation                   | 3.94ms  | 3.5ms   | 0.56ms | 0.38ms |
| Total per stage                  | **4.27ms** |      | **1.01ms** |    |
| | | | |
| `create_shader_module`           | 0.005ms | 0.01ms  | 0.53ms | 0.57ms |
| `create_shader_library`          | 5.19ms  | 6.19ms  | 0.89ms | 1.23ms |
| `create_graphics_pipeline`       | 10.94ms | 12.05ms | 2.24ms | 5.13ms |


The results are split in 2 groups: one for the time spent purely in the shader translation code of SPIRV-Cross (or just "Cross") and Naga. And the other group shows combined times of the translation + Metal runtime doing its part. The latter very much depends on the driver caches of the shaders, which we don't have any control of. We made sure to run the same test multiple times, and only take the last result, giving the opportunity for caches to warm up. Interestingly, the number of outliers (shaders that ended up missing the cache) was still higher in the "Cross" path. This may be just noise, or improperly warmed up caches, but there is a chance it's also indicative of the fact "Cross" generates more of different shaders, and/or being non-deterministic.

The total time spent in shader module or pipeline creation is **7s** with Cross path and just **1.29s** with Naga. So we basically shaved 6 seconds off the user (single-core) time just to get into the game.

In neither case there was any pipeline caching involved. One could argue that pipeline caches, when loaded from disk, would essentially solve this problem, regardless of the translation times. We have the support for caching implemented for Naga path, and we don't want to make it unfair to Cross, so we excluded the caches from the benchmark. We will definitely include them in any full games runs of gfx-portability versus MoltenVK in the future.

## Conclusions

This benchmark shows Naga being roughly **4x** faster than SPIRV-Cross in shader translation from SPIR-V to MSL. It's still early days for Naga, and we want to optimize the SPIR-V control-flow graph processing, which can be seen in the numbers taking time. We assume SPIRV-Cross also has a lot of low-hanging fruits to optimize, and are looking forward to see its situation improving.

Previously, we heard [multiple](https://www.reddit.com/r/rust/comments/n1uidm/gfxrs_ecosystem_releases_v08/gwg1uym/) [requests](https://github.com/gfx-rs/gfx/issues/3030) to allow MSL generation to happen off-line. We are hoping that the lightning fast translation times (1ms per stage) coupled with pipeline caching would resolve this need.

The quality and read-ability of generated MSL code in Naga is improving, but it's still not at the level of SPIRV-Cross results. It also doesn't have the same feature coverage. We are constantly adding new things in Naga, such as interpolation qualifiers, atomics, etc.

Finally, Naga is architectured for shader module re-use. It does a lot of work up-front, and can produce target-specific shaders quickly, so it works best when there are many pipelines created using fewer shader modules. Dota2's ratio appears to be 2 pipelines per 1 shader module. We expect that applications using multiple entry points in SPIR-V modules, or creating more variations of pipeline states, would see even bigger gains.
