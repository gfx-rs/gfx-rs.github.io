---
layout: post
title: wgpu alliance with Deno
---

gfx-rs community's goal is to make graphics programming in Rust easy, fast, and reliable. Our main projects are:

  - [wgpu](https://github.com/gfx-rs/wgpu) is built on top of wgpu-hal and naga. It provides safety, accessibility, and portability for graphics applications.
  - [naga](https://github.com/gfx-rs/naga) translates shader programs between languages, including WGSL. It also provides shader validation and transformation, ensuring user code running on the GPU is safe and efficient.

`wgpu` works over native APIs, such as Vulkan, D3D12, Metal, and others. This involves a layer of translation to these APIs, which is generally straightforward. It promises safety and portability, so it's critical for this library to be well tested. To this date, our testing was a mix of unit tests, examples, and a small number of integration tests. Is this going to be enough? Definitely no! 

Fortunately, [WebGPU](https://gpuweb.github.io/gpuweb/) is developed with a proper [Conformance Test Suite](https://github.com/gpuweb/cts) (CTS), largely contributed by Google to date. It's a modern test suite covering all of the API parts: API correctness, validation messages, shader functionality, feature support, etc. The only complication is that it's written in TypeScript against the web-facing WebGPU API, while `wgpu` exposes a Rust API.

## Deno

We want to be sure that the parts working today will keep working tomorrow, and ideally enforce this in continuous integration, so that offending pull requests are instantly detected. Thus, we were looking for the simplest way to bridge `wgpu` with TS-based CTS, and we found it.

Back in March [Deno 1.8 shipped](https://www.infoq.com/news/2021/03/deno-1-8-webgpu/) with initial WebGPU support, using `wgpu` for implementing it. [Deno](https://github.com/denoland/deno) is a secure JS/TS runtime written in Rust. Using Rust from Rust is :heart:! Deno team walked the extra mile to hook up the CTS to Deno WebGPU and run it, and they reported first CTS results/issues ever on `wgpu`.

Thanks to Deno's modular architecture, the WebGPU implementation is one of the pluggable components. We figured that it can live right inside `wgpu` repository, together with the CTS harness. This way, our team has full control of the plugin, and can update the JS bindings together with the API changes we bring from the spec.

Today, WebGPU CTS is fully hooked up to `wgpu` CI. We are able to [run the white-listed tests](https://github.com/gfx-rs/wgpu/runs/3606626618?check_suite_focus=true) by the virtue of adding "needs testing" tag to any PR. We are looking to expand the list of passing tests and eventually cover the full CTS. The GPU tests actually run on github CI, using D3D12's WARP software adapter. In the future, we'll enable Linux testing with lavapipe for Vulkan and llvmpipe for GLES as well. We are also dreaming of a way to run daemons on our working (and idle) machines that would pull revisions and run the test suite on real GPUs. Please reach out if you are interested in helping with any of this :wink:.

Note that Gecko is also going to be running WebGPU CTS on its testing infrastructure, independently. The expectation is that Gecko's runs will not show any failures on tests enabled on our CI based on Deno, unless the failures are related to Gecko-specific code, thus making the process of updating `wgpu` in Gecko painless.

We love the work Deno is doing, and greatly appreciate the contribution to `wgpu` infrastructure and ecosystem!
Special thanks to [Luca Casonato](https://lcas.dev/) and [Leo K](https://github.com/crowlKats) for leading the effort :medal_military:.
