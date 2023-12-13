---
layout: post
title: ???
---

## Slimmed gfx-rs

We've done a few big steps towards making gfx-hal easier to build and maintain. First of all, we removed the "magical" dependencies, such as `failure` and `derivative` in favor of straight and simple code. That sped up the gfx-hal build by a factor of 8.5X.

Secondly, the "typed" layer of gfx-hal got removed. Previously, we recognized that the view of the API for the backends (when implementing it) needs to be different from the users (to use it). Backends just need to implement "raw" interfaces, while users need more type safety, such as limiting the set of operations available on a command buffer when recording a render pass. This duality of APIs in gfx-hal worked to an extent. Providing safety was impossible without imposing some restrictions or overhead. At the end of the day, we decided that the "typed" (user-facing) layer is still useful, but it doesn't have to be a part of gfx-hal. So we removed it, recommending `rendy-command` as a replacement. This slimmed up gfx-hal API surface and allowed us to straighten up the terminology (no more "RawXXX" or "XxxTyped").

## Feature-less wgpu-rs

From the very beginning of gfx-rs project, it was structured in a way that backend implementations of the API lived in separate crates. The user (or library client) was responsible to select the proper backend for their needs. It was very difficult for a client to support multiple backends within the same binary, and to our knowledge - nobody did.

With the set of supported backends growing, this became more and more of an issue. Asking the user to enable the proper feature doesn't solve the problem - it just puts it on the users shoulders. Moreover, using features in such a way is typically non-idiomatic, because in Rust features are supposed to be additive, but most of the user code was written to work with a single backend, even if it's generic.

Today, we are happy to announce that the problem is solved in wgpu-rs! Features are gone. The library knows which backends are supported on the target platform, it checks for their availability and enumerates their phusical adapters. When the user requests an adapter, they can provide a mask of backends, and the implementation will pick the most suitable adapter on one of the enabled backends.

The immediate effect on the users is that the library "just works". All the complicated machinery of the backends is now hidden within the implementation. This is the case currently for `master` and will make it to the next release.
