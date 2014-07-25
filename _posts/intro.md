---
layout: post
title: Introduction to gfx-rs
---

The goal of the `gfx-rs` project is to make a high-performance, easy to use,
robust graphics API for the Rust programming language. Later posts will detail
show how we achieve that, but this one serves as a high-level introduction and
tutorial to `gfx-rs`. A basic familiarity with 3D graphics in general is
assumed (know what a vertex is). We use the classic triangle example.

# Basic Setup

{% highlight rust %}
#![phase(phase)]

#[phase(plugin)]
extern crate gfx_macro;
extern crate gfx;
extern crate glfw;

#[vertex_format]
struct Vertex {
    pos: [f32, ..2],
    color: [f32, ..3]
}
{% endhighlight %}

Just some boilerplate to link to the libraries we need, as well as defining our vertex type.
`gfx-rs` will generate the code necessary to pack `Vertex` into (in OpenGL) a VAO, in a completely
typesafe way.

# Creating a renderer

The first thing any `gfx-rs` program needs to do is get a window to render to.
Right now, only the `glfw` library is supported.

{% highlight rust %}
let glfw = glfw::init(glfw::FAIL_ON_ERRORS).unwrap();

let (mut window, events) = gfw::glfw::WindowBuilder::new(&glfw)
    .title("Welcome to gfx-rs!")
    .try_modern_context_hints()
    .create()
    .expect("Could not make window :(");

window.set_key_polling(true);
{% endhighlight %}

This will create an 800x600 window using the latest OpenGL version the
graphics driver supports, and listen to any keypresses that happen. The next
thing we need to do is create a renderer and device.

{% highlight rust %}
let (renderer, mut device) = {
    let (context, provider) = gfx::glfw::Platform::new(window.render_context(), &glfw);
    gfx::start(context, provider, 1);
};
{% endhighlight %}

The context provides buffer swapping, and the provider provides GL extension querying and function
loaidng. The magic `1` is how many frames the renderer will buffer before it blocks on the device.
The `device` abstracts over a specific graphics API and isn't that interesting, but the `renderer`
provides a high-level, easy to use interface.

The interaction between `device` and `renderer` is simple: the `renderer` sends commands to the
`device`, and the `device` processes these commands. These can happen in separate threads in a
completely safe way. So, let's create the thread that will drive the `renderer`:

{% highlight rust %}
spawn(proc() {
    let mut renderer = renderer;
    let frame = gfx::Frame::new();
    let state = gfx::DrawState::new();
    let vertex_data = vec![Vertex { pos: [ -0.5, -0.5 ] },
                           Vertex { pos: [ 0.5, 0.5 ] },
                           Vertex { pos: [ 0.0, 0.5 ] }
                          ];
    let mesh = renderer.crate_mesh(vertex_data);
    let program = renderer.create_program(...);

    let clear = gfx::ClearData {
        color: Some(gfw::Color([0.3, 0.3, 0.3, 1.0])),
        depth: None,
        stencil: None
    };

    while !renderer.should_finish() {
        renderer.clear(clear, frame);
        renderer.draw(&mesh, gfx::mesh::VertexSlice(0, 3), frame, &bundle, state).unwrap();
        renderer.end_frame();
        for err in renderer.errors() {
            println!("Render error: {}", err);
        }
    }
})
{% endhighlight %}

There's a lot going on here, but it's mostly simple. We create a new frame to draw into
(`Frame::new()` always returns a frame that corresponds to the window). We create drawing state,
which we don't customize here, but specifies things like winding order and depth testing. We then
hard-code the vertices of the triangle. We create a mesh from that vertex data. We create a program
out of individual shaders. Next we create the data that we will clear the `Frame` with.  Next is the
mainloop of the renderer. We clear the frame, draw our triangle, and end the frame. We then print
out any errors that happened while doing all this.

That's all in a separate thread. The benefit of this is that the driver can still do work
while the renderer is doing complex work. We still need to do something with the device, though:

{% highlight rust %}
while !window.should_close() {
    glfw.poll_events();
    // any event handling
    device.update();
}
device.close();
{% endhighlight %}

And that's it! The `update` method will process any commands it has received from the renderer.
There is some synchronization going on here. We created the renderer/device with a queue of 1 frame,
which means that when `renderer.end_frame()` is called, it will wait until the `device.update()`
replies that it has finished a frame. For the complete, runnable example, [see the git
repo](https://github.com/gfx-rs/gfx-rs/blob/master/src/examples/triangle/main.rs).
