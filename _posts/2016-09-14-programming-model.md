---
layout: post
title: Programming Model
---


## Overview

GFX is a graphics abstraction layer in Rust. It aims to conceal the quirks and capabilities of different APIs, enforce safety at the type system level, and approach graphics programming in a Rusty way, with minimal performance overhead.

Graphics APIs, much like programming languages, come with their own model of thinking. Understanding this model allows programmers to use the API efficiently, but also to see the rationale behind the core design decisions that would otherwise be non-obvious. GFX has historically been confusing for newcomers, especially those coming from OpenGL background. We've come a long way too - first GFX was very similar to GL, but our thinking went through several iterations since then. We are not done yet, but given that the number of backends we support has quadrupled in 2016 ([DX11](https://github.com/gfx-rs/gfx/tree/master/src/backend/dx11), [Metal](https://github.com/gfx-rs/gfx/tree/master/src/backend/metal), [Vulkan](https://github.com/gfx-rs/gfx/tree/master/src/backend/vulkan), in addition to [OpenGL](https://github.com/gfx-rs/gfx/tree/master/src/backend/gl)), it would be considerably more difficult to make radical changes now.

In this post, I'll try to explain the main concepts behind our programming model, in chronological order:
  
  - Bind-less draw calls
  - Command buffers
  - Resource views
  - Pipeline states


## Bind-less Draw Calls

Low-level graphics operates with states, very much like low-level assembly mutates registers. In GL, you'd set a bunch of states and then issue a draw call like this:
{% highlight Rust %}
gl.UseProgram(program);
gl.EnableVertexAttrib(0);
gl.VertexAttribPointer(...);
gl.Enable(gl::BLEND);
gl.DrawArrays(gl::LINES, 0, 10);
// restore states
{% endhighlight %}

This approach is painful due to the lack of locality. Seeing the draw call code would not allow you to predict the outcome easily, since you'd have to inspect all the states that are being used and where they are set up prior to the draw call. It's a classic case of imperative programming with all its faults.

With GFX, you pass everything into the draw call:
{% highlight Rust %}
encoder.draw(
	&slice, // contains the range of vertices and (optionally) an index buffer pointer
	&pso,	// the pipeline state object, having all the graphics state encapsulated
	&data	// the pipeline data, such as: vertex buffers, constant buffers, textures, etc
);
{% endhighlight %}

Note that you don't pass *all* the state in each draw call, but only what is needed. We encode as much dependency information as possible into the type system, thanks to Rust's powerful trait, enums, and references. At the same time, we allow one to trade the run-time correctness for the compile-time in case where more flexibility is needed.


## Command Buffers

Last-gen APIs (such as OpenGL and DX11) suffered from the limited multi-threading capabilities, because you could only render in one thread that owned the graphics context. The CPU rendering overhead thus would often become the bottleneck of applications, reducing the number of draw calls sent through the pipeline.

This issue has been properly addressed by all of the current-gen APIs (DX12, Vulkan, etc) - they introduced the concept of a command buffer that can be filled up independently of the rendering context. Conceptually, a command buffer is an opaque object that is optimized for direct execution on your graphics hardware (it's not portable, much like the machine code). The idea is - you can populate multiple command buffers on multiple threads simultaneously, and then schedule them for execution on the hardware to the single graphics queue. Obviously, the concept complicates programming a bit. In exchange, it allows multi-threading your renderer and thus being able to draw  many more objects on screen.

GFX functions the same way. We have the abstract command buffer interface implemented by the backends, and we wrap them into `Encoder` objects, which fill up (encode) the command buffers by exposing methods for drawing as well as updating the buffers/textures. Once you are ready to execute the encoded commands, you need to pass it to `Device` for submission. The `Device` represents the hardware context and the execution queue. If the commands are encoded on a different thread, you can use channels or sharing to coordinate the submission.

For an example of some multi-threading, you can check out [yasteroids](https://github.com/kvark/yasteroids). In short, the context-owning thread does this:
{% highlight Rust %}
let mut encoder = dev_recv.recv().unwrap(); // Receive the next command buffer through a channel. Note the error processing is omitted for clarity.
encoder.flush(&mut device);                 // submit for execution, also clears the buffer
dev_send.send(encoder).unwrap();            // send it back for reuse
window.swap_buffers().unwrap();             // flip the front and back buffers, showing the frame
device.cleanup();                           // delete the resources no longer used
{% endhighlight %}

Yasteroids doesn't have a dedicated render thread. Instead, it's using [specs](https://github.com/slide-rs/specs) ECS, and the renderer is just one of the systems that can travel freely between threads in a pool. It behaves like this:
{% highlight Rust %}
let mut encoder = self.channel.receiver.recv().unwrap(); // get the next command buffer. Error processing is cut off for clarity here
encoder.clear(&self.out_color, [0.0, 0.0, 0.0, 1.0]);
encoder.update_constant_buffer(...);                     // called multiple times
encoder.draw(...);                                       // called multiple times
self.channel.sender.send(encoder);                       // send the command buffer for submission through a channel
{% endhighlight %}

Note: the OpenGL implementation of our command buffers is particularily interesting. Since OpenGL (core profile) doesn't support command buffers natively, we emulate them. On one hand, it may introduce a bit of an overhead comparing to raw GL. On the other hand, this allows us to off-load some work on the non-context threads: verifying and caching the draw state.


## Resource Views

In OpenGL a texture actually represents multiple things:
  - the allocated video memory storage for the texel data
  - the way it's being accessed (format, binding)
  - the sampler state to be used with it

DX11 introduced a conceptually new look at the graphics objects. It separated all these three concepts into different things. First, you create a texture object, which you can only have limited things to do with. If you want to use it as a render target, you create a render target view (RTV) for it. For sampling, you create a shader resource view (SRV), and optionally an extra sampler object. There are also depth-stencil views (DSV) and unordered/ordered access views (UAV, OAV). You can have multiple different views into the same resource, as long as they are compatible (yout can't view a single-channel 16 bit format as RGBA8, for example).

In GFX, when you create a texture you only provide the surface format, which dictates how many bits each channel has. Then, for the view creation, you specify how these bits are interpret (normalized integers, floats, raw ints, etc).

Note: the resources are created by a `Factory` object, which may or may not be sendable/clonable, depending on the backend features. Conceptually, you are supposed to create resources on dedicated threads, but this choice is left up to the user.

Example:
{% highlight Rust %}
let texture = try!(factory.create_texture(kind, levels, SHADER_RESOURCE | RENDER_TARGET, Usage::GpuOnly, None));         // providing the storage configuration and requesting the capabilities
let resource = try!(factory.view_texture_as_shader_resource::<Srgba8>(&texture, (0, levels-1), format::Swizzle::new())); // creating an SRV for the whole mipmap range
let target = try!(factory.view_texture_as_render_target(&texture, 0, None));                                             // creating an RTV for level 0
{% endhighlight %}

There are helper methods to create the texture with the views for your convenience (`create_render_target`, `create_depth_stencil`, etc). Later, the texture object itself does not participate in any rendering. It can only be used for updating the contents and reading them back. Typically, you'd ignore this object and only work with its views, given that they hold the parent object automatically.

Note: resources in GFX are atomically reference-counted (`Arc`). The device holds the last references, and it really deletes the resources (in the `cleanup()` call) if no user/internal references are alive.

Note: applied to OpenGL, the GFX texture and RTV/DSV are mapping to the same thing (GL texture). The GL render buffer is used instead if `SHADER_RESOURCE` binding is not requested.


## Pipeline states

The general trend in graphics hardware is to do more and more with programmable shaders. This has been the case in consumer cards for a long time, since the first assembly shaders came into play. Thus, the states that used to map directly to switches in fixed-function hardware (like vertex attribute fetches) can now be compiled (internally by the driver) into a small chunk of shader code and executed somewhere in-between your shader code. This compile step and the hookup can introduce lags and require some anticipation by the driver, making those drivers rather bloated and full of quirks (part of the reason AMD/NV supply game profiles in the drivers). Current-gen APIs largely address this by exposing the pipeline state concept.

A pipeline state objects (PSO) encapsulates all the dependent states of your shader program (with the program itself). You declare in advance what kind of vertex attributes will come as input, what render targets to draw to, with what blending modes, what textures/samplers to use, etc. This allows the driver to compile the most efficient code for the set of states. Batching becomes trivial - just send multiple draw calls with the same PSO to get the maximum performance. The downside here is the inconvenience and restrictive nature of these forward state declarations.

Our PSOs are defined with macros like this:
{% highlight Rust %}
gfx_defines!{
    vertex Vertex { // Defininig a vertex buffer format. You can have multiple vertex buffers having different attributes in them.
        pos: [f32; 2] = "a_Pos",
        color: [f32; 3] = "a_Color",
    }

    pipeline pipe {
        vbuf: gfx::VertexBuffer<Vertex> = (), // our vertex buffer
        out: gfx::RenderTarget<ColorFormat> = "Target0", // the output target, no blending involved
    }
}
{% endhighlight %}

When making a draw call, you provide the generated `Data` struct instance pointing to the used resources and views:
{% highlight Rust %}
encoder.draw(&slice, &pso, &pipe::Data {
        vbuf: vertex_buffer,
        out: main_color,
});
{% endhighlight %}

We match all the specified compile/run-time information (names, formats, offsets, field etc) with the shader reflection upon creating a PSO. This allows catching the possible mismatches at the initialization time, leaving the draw time error-less. We also provide a range of PSO components to be used in that definition, including some trade-offs between compile time and run-time safety. For example, you can have your RTV format known in advance, or you can pass it in as data during the PSO creation. Please refer to the dedicated [blog post](http://gfx-rs.github.io/2016/01/22/pso.html) for more information about how PSOs are implemented in GFX.


## Conclusion

We borrowed a lot of concepts from the new graphics APIs. This allows us to stay as close to the underlying hardware as possible, while still being portable. DX12 has the closest programming model to GFX, despite the fact that we don't have the corresponding backend in the works. Plus, GFX adds a bit of Rusty flavor to the API, in particular with the macro-based PSO definition and the bind-less approach.

Hopefully, this post sheds some light on the complexity that GFX brings along, and allow more people to get familiar with it. We strive to be the best graphics abstraction library out there, and we appreciate any help or feedback. Check out our [Readme](https://github.com/gfx-rs/gfx/blob/master/README.md) and visit us on [Gitter](https://gitter.im/gfx-rs/gfx). A big thanks to all contributors who made GFX possible!
