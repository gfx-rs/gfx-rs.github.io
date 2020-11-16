---
layout: post
title: The Big Picture
---

gfx-rs community's goal is to make graphics programming in Rust easy, fast, and reliable.
In this post, we are going to provide an overview of the projects we work on, how they are connected, and what the future holds for us:

![big picture](/img/wgpu-big-picture.svg)

Full [diagram link](https://viewer.diagrams.net/?highlight=0000ff&edit=_blank&layers=1&nav=1&title=wgpu-big-picture#R7V1tk5s2EP41nmk%2F%2BAYk3vzxXtI0aZrenKe9Jl8yspExPQwuYJ%2Bvv77CIBskgTEGgdNeZjJICDB69tldrVZiBO9Xu%2FchWi9%2FDWzsjYBi70bwYQTARDPI%2F0nFW1qhQz2tcELXTqvUY8XU%2FQdnlUpWu3FtHBUaxkHgxe66WDkPfB%2FP40IdCsPgtdhsEXjFp66Rg7mK6Rx5fO2za8fLtNbSlWP9z9h1lvTJqpKdmaH5ixMGGz97nh%2F4OD2zQvQ2WdNoiezgNfc8%2BG4E78MgiNOj1e4ee0mv0h5Lr%2Fup5OzhJ4fYj%2Btc4H18dh6%2For9%2FHtvbr78%2Bxar2dTU2QXqbLfI2WV98nI59FLtb8h7KM569f%2Fw96eQ1QSJ9jfiN9trr0o3xdI3mSfmVCMYI3i3jlUdKKjm0UbTEdlZYuJ53H3hBuL8ULhYLMJ%2BT%2Bi0OY5cAceu5jk%2FOxUFyF5SV5uTdcJhcHvjxNHtycrsoDoMXnLuhbcwM3SBn%2BG7Jeip5Et6x%2FU8kGgcrHIdvpMmOSm8GWSbNwMjKr0fZUGndMicXtA5l4ugcbn3EhRxk0JwBk2FxMH3uHCOs2jo2W8RoYpgQtYWRUcRItQQY6QKMzBYwQr9vP8cPzz%2B9C55U7%2Bnbt1%2F86YexanCdj22iZbJiEMbLwAl85L071t4daz8FSa%2FuO%2B4vHMdvmZJEmzgoQoZ3bvwnOVay4y%2FJ8Y2elR52uVMPb1mhAHP6M5PfVqpJsqoo2IRzXCWXhhikEHupeBYUtaDHs0sfA5c8%2BgCuxmCr68U7xCh0cJxdxMB2%2BBUXsM0oZ9sfG%2B8F%2Bf81th3sHUUECNhmdsQ2MUYKhxFRhAO0VC30PmR0HYR87wNL0PtWV72vT7jeHwHDi5O%2BdLfk0EkOnzZRfEAkPTsL6UlaQ56fu6RV5Kw5rkCuA6DApAiUwCZBEU6dsQRYw7VJpJvDtz%2FpDZJC7qqkeLxsX6LXpYqOuumgCrmT9ouONfb2pKqdJjSaFxi%2Bi2CFpmRYb4CeQ1ath2sOyi8FJOvgWsnIk7jCuri2Dev%2B0tswRG%2B5BuvEUYkq3B1Gv3MjObY90Kvak4P0F7TqFEFZqqRELYBTeqGJDmpTlxhmr0LHSYnOGCOgMV50%2BkadedFwIllglBuzoHDUugJT1G6KNIkBtc3PINSUrond0FI1petV7btRU7oqSeqaCE9jUZXg4YBBeTg6H5cUDDCe308%2FlY0juMbRZr0mQDceiSQ98wnNsFcUBG4AyI42Vq5t76UsxJH7D5rt75eAmvGN3Fy%2FG%2BkPQpgrZZwbmBwC5NlTRplnUDpgUW5UMMl826bmhjYJFosId2JJ1FJZSMZwBZSMvzcBPTGO9uS9JQ1Ip%2B6OJynEr856kwM%2FvVkJ%2BtESrZNDAg7yPOwFTohWCYg4dMn7JcgXzz0eT9ydGMQu3B2mkyKCcaytY8vWRCEeC8yg0VKIBxqMU6kJQjxAZtBb12Sr8u4HORc4CzUVORyEr6BZbAj3hK8wqWzfja9Qy8bc4e1bTQvRo444xLo4HTGzdE2vFL36OkIDDEpqTR0BW9AR4kkXXa6OUEf1B5dHHXHmSLar2Zq6UZF0JNtbcItjZWKnxyEf1f%2FubbLGTEQLJzm74pvYJkvmW0ObbEoyytpVGWWDncUzqo2yzqp7Q4JR1jj639zcDIn6NUxtcgWlRCbbLWgDNv4i1fqKtQG8Cm0AJGkD%2Faq0ATurfEobsLMUUrSBzmmDz8j3g82VKYQW2A9Z5d07%2B4H0WchGkVYgKdJq1GT%2FMIL57BziqWA%2BnPQQzOfzsKY43AaDIj9ND%2Bo0I5UNzh3KvZHf5KF5%2FPD0x%2Fg%2BDKKz86%2F4vjMnMyWho%2BOhKMqoeUigLnGxohccz5dlMGFjD1MLYADGLYa9j8ro5E0ODR85aFg8yQAox7oNaNgwotk3NEB6ENsctJVU6w6ZBzYhCXlPVBAsdrCPQxQnicLJyhDs2%2FWnJ4l2w2HzRMnhTE9Skb98enJM5BIqVoHSdPQ03OlKICvxReK8lSAk3Sn7zWGRn3dE%2Fyd%2FtfC3Qn6V%2FBXt%2BdC5r8lKP2qYPF1K3dP22GibkuKBJoBa0btWGNesJG3x3AHwwTVknlM2AObaZ1OP3QbD%2BXV%2F9o56RoPkv1bmszfITYKTCbgy%2BssOiJ%2Fv6relMFr09Fs39o0UAqCrKmgCFJRBcD6Osn0ZMr3p7HwL9FYUq9jj4yHxW5xuIla9ktZTmd0sqBKyrz3OCztS64Ph6oQ1%2BZNqk8%2B219vVCGIR4%2FNfiMkXu5VSdEJtqhdFpoI%2FFTpBNY3B6wA%2BQ2EVe33qBXCVekHcUL9QMVwWR%2BehdRa7oaWeIWwthJF0Y27h2aKlSDozySFayyyMpGstRNKFvJMdVStQrJphbTLlUgKUTN8yaJpyd9RQ%2Beg5uSXy%2FpPUYtK4hLsLSU2ivo6czgZJXO0wUpDY0YlPexGGA97ooXUsutaONLQnTTvy0ws2tFXwv3bMhX%2F7W2KicCB8l9S6dNFHPccDSqYWH19D0XJIrFroyT8hq%2FZ%2Fo3Qxa64%2B%2FevGFxHt6yU1X0aT7Yyc74oMMIJuCRWB9Ag6mxinVU%2BpAeXC9hMZEXreODviaM9AIvR0qNXGBLyiXl2IXnZSek%2FW2pJhrQ9J37KsNT%2Fd7XjB65DMdT9OcO8ZxodXOCLzHs9f%2BFT8k73bQ%2Fa8rjLdWTfi0kYsU7ywQbSlCau2fPs22dqclPa7igsy5uk2VHv3MZ8dTwniuf5LVR%2BedBhyPSTaTJnWXZrzU8RHU5l%2BL0n5OXWfA6ay9g0GAlS5XL%2FfprXTAu2Q9OkxLfBk%2B%2F0%2B4FeWMGiU0blBRoFpQVh0V8CA3BXx60sPvhrN9vLUVPOcAOy%2BlDO7%2FAqeVvderD1San0j12YjJUU87i5VbGZl%2B45W3%2FGzoaKs58VunOyrhmau58alW%2BVw1%2F1AfpRy%2B%2Fjhx8ZJz9TILfEOETIw%2Fl9WexBBcL7rx%2B%2Ft3YcvqDIrNQ9fNejNGTT4oNp%2Bixb62ZDvEmspG77oJrMJW%2B%2BLvE3Z%2B2meMZ4ui8iZp0JyFXbosvUtVE3LXwR%2BGcj8xxQesB%2BUszXP7Y%2FTqye3pGGozhhxmaN6YaxswF%2BsEfLjJCm7yzU9GSrr7hs15S9VNMGuj9ZuKbmucWpLMCbkBKB2DK3ufqYN2EaKx%2B%2B4pQgfP5MH3%2F0L).

Legend:
  - _parallelogram_: Rust node
  - _rectangle_: C/C++ node
  - _hexagon_: node in Rust that exposes an external API

Colors correspond roughly to the areas: gfx is blue, wgpu is green, JS stuff is yellow, etc.

## Nodes

[ash](https://github.com/MaikKlein/ash/) - Vulkan bindings we use in _gfx-rs_, external to us.

[metal-rs](https://github.com/gfx-rs/metal-rs) - Metal bindings crate we use in _gfx-rs_, also used outside.

[d3d12-rs](https://github.com/gfx-rs/d3d12-rs) - simple D3D12 convenience wrapper we use in _gfx-rs_.

[winapi](https://github.com/retep998/winapi-rs) - WinAPI bindings we use in _gfx-rs_ for both DX11 and DX12 (where _d3d12-rs_ has gaps), external to us.

[glow](https://github.com/grovesNL/glow) - OpenGL (including ES and WebGL) bindings we use in _gfx-rs_, also used outside.

[gfx-rs](https://github.com/gfx-rs/gfx) - our portable graphics API in Rust, tightly following Vulkan.

[gfx-portability](https://github.com/gfx-rs/portability) - a [Vulkan Portability](https://www.khronos.org/blog/fighting-fragmentation-vulkan-portability-extension-released-implementations-shipping) API wrapper around _gfx-rs_. Can be used as a drop-in Vulkan driver on platforms without native Vulkan support.

[SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross) - shader translation library in C++, taking SPIR-V as an input.  It's developed by a few Khronos members. We currently use it for generating platform-specific shaders in _gfx-rs_. We are phasing it away, replacing by _naga_ gradually (the striped fill signifies deprecation in the diagram).

[naga](https://github.com/gfx-rs/naga) - our new shader translation library in Rust, it has a number of front-ends (SPIR-V, GLSL, and WGSL so far), various backends (SPIR-V, MSL, GLSL so far), and a solid intermediate representation (IR) in the middle. It's a young project, and we are slowly rolling it out to replace the C++ blocks around the ecosystem, such as SPIRV-Cross and glsl-to-spirv/shaderc (used by many gfx/wgpu/Vulkan devs).

[wgpu](https://github.com/gfx-rs/wgpu) - our implementation of WebGPU API in Rust. It's safe, portable, and fast (in this order).
  - Uses _gfx-rs_ to reach the hardware.
  - Uses _naga_ to parse WGSL shaders, as well as introspect both SPIR-V and WGSL shaders. This allows us to validate the shaders, also to derive the implicit bind group layouts.
  - able to record and replay [API traces](http://kvark.github.io/wgpu/debug/test/ron/2020/07/18/wgpu-api-tracing.html) across platforms.

[wgpu-rs](https://github.com/gfx-rs/wgpu-rs) - idiomatic Rust bindings to _wgpu_, meant for Rust end-users. Has many dependent [applications and libraries](https://github.com/gfx-rs/wgpu-rs/wiki/Applications-and-Libraries), including Nannou and Bevy.
  - Currently able to target _wgpu_ as well as the WebGPU/WASM (experimental in browsers).
  - Will be able to link to an implementation behind the portable [WebGPU C header](https://github.com/webgpu-native/webgpu-headers), such as _wgpu-native_.

[wgpu-native](https://githubb.com/gfx-rs/wgpu-native) - a wrapper around _wgpu_ that exposes a WebGPU C API conforming to the [shared header](https://github.com/webgpu-native/webgpu-headers). The plan is to have it accessible by non-Rust libraries, as a drop-in replacement for [Dawn](https://dawn.googlesource.com/dawn), also accessible via [NAPI](https://nodejs.org/api/n-api.html).

[Deno](https://github.com/denoland/deno) - JS/TS runtime in Rust. There is a mostly complete [PR #7977](https://github.com/denoland/deno/pull/7977) delivering support for WebGPU via _wgpu_. It should allow native JS/TS applications (such as  _TensorFlow.js_) using WebGPU to run directly on top of _wgpu_ in the future.

[Servo](https://github.com/servo/servo) - an experimental browser engine in Rust. It has WebGPU support via _wgpu_.

[Gecko](https://developer.mozilla.org/en-US/docs/Mozilla/Gecko) - the browser engine powering Firefox, developed by Mozilla. It also implements WebGPU via _wgpu_.
