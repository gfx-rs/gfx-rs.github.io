---
layout: post
title: Shader Parameters in gfx-rs
---

### Introduction

Shader parameters (also called uniforms) are values, provided by the user, that are constant through the draw call's execution. Things like material properties, textures, screen size, or MVP ([Model-View-Projection](http://stackoverflow.com/questions/5550620/the-purpose-of-model-view-projection-matrix)) matrix are all shader parameters.

When a shader program is linked, the graphics API allows querying all the parameter names, their types, and internal locations that are used to assign the values for them. Storing the locations, verifying the compatibility of types with your data, and managing extra/unused error cases may quickly become an error-prone burden for the programmer.

A typical solution for the problem is introducing built-ins: MVP matrix, for example, is used for almost every vertex shader. Built-ins are convenient, but they have issues:
- require the user to obey the conventions in naming and type
- make it difficult to extend the system outside of the predefined scope

We wanted gfx-rs to be low level and flexible, to provide an equally convenient interface for any custom setup. In this article, we'll describe gfx-rs solution, analyze the common pitfalls of existing interfaces, and explain how we tackle them.

### Overview of the existing shader interfaces

_Disclaimer_: I haven't worked closely with most of these engines, so any corrections are welcome!

  * UDK
```JavaScript
    theMaterialInstance.SetVectorParameterValue('color', FLinearColor(0.0,0.0,0.0,1.0));
```
  * Unity3D
```
    Properties {
        _Color ("color", Vector) = (0,0,0,0)
    }
```
```C
    renderer.material.SetVector("_Color", Vector4(0,0,0,1));
```
  * Irrlight
```C++
    if (UseHighLevelShaders)
        services->setVertexShaderConstant("color", reinterpret_cast<f32*>(&col), 4);
    else //note magic constants!
        services->setVertexShaderConstant(reinterpret_cast<f32*>(&col), 9, 1);
```
  * Ogre3D
```
    fragment_program_ref GroundPS
    {
        param_named color float4 0
    }
```
```C++
    pParams = pPass->getFragmentProgramParameters();
    pParams->setNamedConstant("color", Ogre::Vector4(0.0,0.0,0.0,1.0));
```
  * Horde3D
```C++
    Horde3D::setMaterialUniform(m_Material, "color", 0.0, 0.0, 0.0, 1.0);
```
  * Three.js
```JavaScript
    var uniforms = {
      amplitude: {
        type: 'v4',
        value: new THREE.Vector4(0.0,0.0,0.0,1.0)
      }
    };
    var shaderMaterial = new THREE.MeshShaderMaterial({
      uniforms:       uniforms,
      attributes:     attributes,
      vertexShader:   vShader.text(),
      fragmentShader: fShader.text()
    });
```
  * gfx-rs (for comparison)
```rust
    #[shader_param]
    struct Params {
        color: [f32, ..4],
    }
    let data = Params {
        color: [0.0, 0.0, 0.0, 1.0],
    };
```

### SYF 101
SYF: Shoot Yourself in the Foot = "to do or say something that causes problems for you".

Notice how almost every implementation requires you to specify the parameter name as a string. This forces the engine to go through all known parameters and compare them with your string. Obviously, this work is wasted for any subsequent calls. It is also a violation of the [DRY](http://en.wikipedia.org/wiki/Don%27t_repeat_yourself) principle and a potential hazard: every time you ask to match the parameter by name, there is a chance of error (parameter not found because you copy-pasted the name wrong?).

In some engines, you can get a handle to the parameter like this:
```rust
let var_color = program.find_parameter("color");
program.set_param_vec4(var_color, [0.0, 0.0, 0.0, 1.0]);
```
This is a bit more verbose, and partially solves the problem, but clearly "color" is still repeated twice here (as a string and a part of the variable name). Besides, another hazard remains - what if a given parameter has an incompatible type with what shader expects?

Three.js comes the closest to being safe - your variable name is used to query the shader, and the type can be verified inside `MeshShaderMaterial` call. Note, however, that in _JavaScript_ you can change the variable type at run-time, which raises the SYF factor significantly.

### Our solution in gfx-rs

We are using a procedural macro in Rust to generate the following code at compile time:
  1. An associated `Link` structure. It has the same fields as the target one, but the types are replaced by the corresponding variable indices.
  2. Implementation of `create_link()` - a function that constructs the `Link` structure by querying a compiled shader for needed variables.
  3. Implementation of 'upload()' - a function that iterates over all enclosed parameters, used for uploading them onto GPU.
This is all done behind the `shader_param` attribute:
```rust
#[shader_param]
struct Params {
    color: [f32, ..4],
}
```
Generated code:
```rust
struct Params {
    color: [f32, ..4],
}
struct _ParamsLink {
    color: ::gfx::VarUniform,
}
#[automatically_derived]
impl ::gfx::ShaderParam<_ParamsLink> for Params {
    fn create_link<S: ::gfx::ParameterSink>(&self, __arg_0: &mut S) ->
     Result<_ParamsLink, ::gfx::ParameterError<'static>> {
        match *self {
            Params { color: ref __self_0_0 } =>
            ::std::result::Ok(_ParamsLink{color:
                                              match __arg_0.find_uniform("color")
                                                  {
                                                  Some(_p) => _p,
                                                  None =>
                                                  return ::std::result::Err(::gfx::ErrorUniform("color"))
                                              },})
        }
    }
    fn upload<'a>(&self, __arg_0: &_ParamsLink, __arg_1: ::gfx::FnUniform<'a>,
                  __arg_2: ::gfx::FnBlock<'a>,
                  __arg_3: ::gfx::FnTexture<'a>) {
        match *self {
            Params { color: ref __self_0_0 } => {
                use gfx::ToUniform;
                __arg_1(__arg_0.color, (*__self_0_0).to_uniform());
            }
        }
    }
}
```

With `ShaderParam` implemented and a hidden link defined, we can weld the program together with the parameter struct:
```rust
let data = Params {
    color: [0.0, 0.0, 0.0, 1.0],
};
let mut bundle = renderer.bundle_program(program, data).unwrap();
```
This returns a bundle that is later passed into draw calls. The `unwrap()` here ignores these possible errors:
  * providing a param that is not in the shader
  * not covering a shader param
  * some parameter type is not compatible
  * program failed to compile

The bundle exposes the parameter structure publicly and defined as follows:
```rust
#[deriving(Clone)]
pub struct ShaderBundle<L, T> {
    /// Shader program
    program: super::ProgramHandle,
    /// Global data in a user-provided struct
    pub data: T,
    /// Hidden link that provides parameter indices for user data
    link: L,
}
```
So every time you need to change a parameter before a draw call, you just do it directly:
```rust
bundle.data.color[0] = 1.0;
renderer.draw(..., &bundle).unwrap();
```
### Analysis

In gfx-rs all the parameter queries and type verifications are done at init time. Once you got the bundle, working with it is 100% safe. You are forced *by the compiler* to set the initial values (when `Params` is created), and to preserve their types through the execution. There is zero run-time overhead (all the parameters are uploaded to GPU using their indices), zero memory overhead (we are not allocating a `Matrix4` to cover any parameter needs), and all the boilerplate is hidden from the user. Thus, we are able to bring SYF factor to the minimum, meanwhile improving ergonomics and efficiency.
