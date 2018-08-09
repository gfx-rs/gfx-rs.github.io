---
layout: post
title: Portability benchmark of Dota2 on MacOS
---

### The Race

gfx-rs is a Rust project aiming to make graphics programming more accessible and portable, focusing on exposing a universal Vulkan-like API. It's a single Rust API with multiple backends that implement it: Direct3D 12/11, Metal, Vulkan, and even OpenGL. We are also building a Vulkan Portability [implementation](https://github.com/gfx-rs/portability) based on it, which allows non-Rust applications using Vulkan to run everywhere. This post is focused on the Metal backend only.

As our portability implementation matured in terms of Vulkan [Conformance Test Suite](https://github.com/KhronosGroup/VK-GL-CTS) coverage, we started to ask questions about performance. Fortunately, just around this time frame [Valve announced](https://twitter.com/plagman2/status/1002324195135520768?lang=en) official Metal support for Dota2, via [MoltenVK](https://github.com/KhronosGroup/MoltenVK). We couldn't miss the opportunity to test _gfx-portability_ on Dota2, and substituting MoltenVK with our portability implementation appeared straightforward. The initial results were... slightly discouraging. For a while we weren't able to get the game to start, due to our Metal backend was missing some required features, like compressed texture support.

Once Dota2 became functional under our portability implementation, we entered a long period of implementing a mixture of stability and performance improvements. We had to rethink our state caching architecture, resource lifetimes, lock contention, and much more. After finally figuring out all the issues we were having with Apple, Metal, and Dota2 itself, we were ready for the race.

![simple test with gfx-portability](/img/dota-simple.jpg)

#### Modes

Our portability library has been tested in two different modes of the Metal backend:
  - "Immediate" - We record the command buffers "live" as the corresponding Vulkan command buffers are recorded. Doing this allows the user to parallelize the recording as they need be, allowing them to make sure that everything is ready to go for submission.
  - "Deferred" - We collect the commands internally in our command buffer format, only starting to record them to Metal's command buffers at submission time. This is what MoltenVK is currently doing as well. This approach makes the recording feel much faster, since the work is done at submission time. The real benefit, however, comes from us knowing at submission time the order of command buffers to be submitted, allowing the Metal driver to process them simultaneously as we record them, resulting in lower latency overall.

Please note that technically "Immediate" recording has less CPU overhead. Additionally, "Immediate" mode's more explicit threading model results in no surprises at submit time. That's why we consider it to be the superior mode, and that was the focus of our optimization efforts. However, to capitalize on the benefits provided by "Immediate" mode, the application needs to make sure to not hold onto the recorded command buffers for too long and submit more often than once per frame. Failure to do so increases latency, limiting the number of frames produced, as observed in Dota2, which has been programmed to only allow at most a single frame of latency.

Another interesting observation is that 71% of time spent by gfx/immediate (when recording the commands only) is actually spent inside the driver, according to our instrumented profile. This gives us an upper bound for Vulkan Portability overhead on Metal to be about 40%, although that includes some OS and Metal runtime bits as well.

### Results

| Test/Library                     | gfx/immediate | gfx/deferred | MoltenVK     | OpenGL      |
| -------------------------------- | ------------- | ------------ | ------------ | ----------- |
| CPU % of Main thread             | 35%           | 12%          | 21%          | ?           |
| _platform A_ (Intel, dual-core)  | | | |
| fps/variability on low settings  | 49.3 / 4.1    | 46.3 / 4.7   | 40.5 / 6.3   | 45.0 / 5.2  |
| fps/variability on high settings | 33.4 / 3.3    | 40.2 / 4.1   | 35.9 / 5.3   | 34.9 / 6.6  |
| _platform B_ (AMD, quad core)    | | | |
| fps/variability on low settings  | 58.1 / 11.4   | 74.5 / 11.2  | 71.7 / 12.6  | 77.0 / 12.7 |
| fps/variability on high settings | 51.1 / 9.3    | 59.2 / 7.4   | 61.4 / 10.0  | 49.0 / 5.8  |
| _platform C_ (NV, quad core)     | | | |
| fps/variability on low settings  | 54.3 / 10.0   | 66.0 / 7.7   | 64.0 / 7.8   | 56.7 / 7.0  |
| fps/variability on high settings | 40.6 / 4.4    | 43.1 / 3.8   | 42.1 / 3.9   | 37.6 / 3.2  |

The first metric shows how much time of the main thread is spent inside the portability library, compared to the total execution time (which is the actual time minus all the sleeping). It was measured with Time Profiler on a simple scene (see screenshot). Note that the submission is done on a separate thread by Dota2, so that time isn't taken into account here. Interestingly, the less time we spend on the main thread the faster our frame rate ends up being. Or, in other words, it's a race of who gets to the submission first :)

The rows "fps/variability" metric is from the `Source2Bench.csv` produced by timing the [dota2-pts-1971360796.dem](http://www.phoronix-test-suite.com/benchmark-files/dota2-pts-1971360796.dem.tar.bz2) demo in the range 40000 .. 50000. The "low" settings refers to adding `-autoconfig_level 1` to the command line, while the "high" settings is when we passed `-autoconfig_level 3` instead. We followed the instructions provided by [GamingOnLinux](https://www.gamingonlinux.com/articles/want-to-benchmark-dota-2-on-linux-heres-how-to-do-it.7435). Note that lower fps variability is better, as it translates to a more stable framerate.

We were running the same scene as [Phoronix](https://www.phoronix.com/scan.php?page=news_item&px=Dota-2-Initial-Mac-Vulkan), just with a longer time frame and some slightly adjusted settings. The benchmark scripts can be seen at the [gfx-portability PR](https://github.com/gfx-rs/portability/pull/118):
```bash
make dota-bench-gl # for GL
make dota-bench-orig # for MoltenVK bundled with Dota2
make dota-bench-gfx GFX_METAL_RECORDING=deferred # for gfx-portability with "Deferred" recording
```

#### Platforms

![macbook fleet](/img/macbook-fleet.jpg)

| Platform         | A | B | C |
| ---------------- | - | - | - |
| Model            | MacBook Pro 2016 | MacBook Pro 2016 | MacBook Pro 2013 |
| CPU              | 3.3 GHz Intel Core i7 | 2.9 GHz Intel Core i7 | 2.6 GHz Intel Core i7 |
| number of cores  | 2 | 4 | 4
| GPU              | Intel Iris Graphics 550 | AMD Radeon Pro 460 | NVIDIA GeForce GT 750M |
| OS               | macOS 10.14 beta | macOS 10.14 beta | macOS 10.13.6 |
| resolution       | 1440 x 900 | 1680 x 1050 | 1440 x 900 |

### Conclusions

MoltenVK does a good job translating Vulkan to Metal. Interestingly though, it can be slower than OpenGL on low settings. This doesn't match either Phoronix or Valve/MoltenVK's numbers. We suspect the difference to be explained by Phoronix taking a 10x shorter run with pre-loaded pipeline caches. In this case GL would struggle creating all the new pipelines and will not have time to reach the full speed - not exactly an Apples to Apples comparison :)

Strangly, MoltenVK exhibit some sort of "warmup" behavior, where the second run can be up to 5% faster than the first one (we do include this in the comparison). This is unlike gfx-portability, which always shows the same performance. We are not sure what might be causing the warmup effect, given that we cleared `dota/shadercache/vulkan/shaders.cache` consistently.

Either way, our benchmarks show that OpenGL is still fairly good on MacOS, and it's a high bar to beat for any Vulkan portability implementation, at least in the context of Dota2 performance. The performance shift seen on high settings could be explained by the raised scene complexity (which is where low-level APIs generally help), but there is also compute shaders usage in Dota2, which are not available on MacOS's older OpenGL version. In other words, it's not always the raw performance to gain on MacOS but rather more features that allow certain workloads to use the GPU more efficiently.

We believe that immediate command recording has a great potential that hasn't yet been realized with Dota2 as it is architectured today with it's one big submission per frame, which increases latency on the already latency-limited program. Hopefully, we'll see more applications taking advantage of it in the future.

#### Rust

Rust has proven itself viable in complex high-performance systems. We were able to build solid abstractions and hide the complexity behind tiny interfaces, while still being able to reason about and optimize low-level performance. Iterating on large architectural changes was a breeze - we just change the core piece and then fix all the compile errors.

Optimizing our implementation wasn't different from optimizing a regular C++ one, other than the feel of safety when doing so, and the fact a typical profiler considers Rust programs to be mostly moving bytes around. In the end, our portability implementation was able to compete toe to toe with the official alternative (MoltenVK shipped as Dota2 DLC), surpassing it by 13% on a dual-core system with integrated GPU, and by 3% on quad-core systems with dedicated GPUs.

To be fair, we don't think MoltenVK has put as much effort into optimizing the code to date as we have. So while we could see a lot of potential in improving our Metal backend further, MoltenVK likely has more low-hanging fruits to grab at this point.

The Rust ecosystem deserves a special mention: hooking up extra dependencies took mere minutes, and most of the time things just worked. Special thanks to:
  - [@SSheldon](https://github.com/SSheldon) for macOS API libraries
  - [@Amanieu](https://github.com/Amanieu) for the faster locks
  - [@danginsburg](https://github.com/danginsburg) for help in understanding Dota2 rendering pipeline
  - [@jrmuizel](https://github.com/jrmuizel) for performance investigation assistance
  - gfx-rs hackers - you are the best!
  - Rust team and contributors
