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
{% highlight Javascript %}
theMaterialInstance.SetVectorParameterValue('color', FLinearColor(0.0,0.0,0.0,1.0));
{% endhighlight %}
  * Unity3D
{% highlight C# %}
Properties {
    _Color ("color", Vector) = (0,0,0,0)
}
{% endhighlight %}
{% highlight C %}
renderer.material.SetVector("_Color", Vector4(0,0,0,1));
{% endhighlight %}
  * Irrlight
{% highlight C++ %}
if (UseHighLevelShaders)
    services->setVertexShaderConstant("color", reinterpret_cast<f32*>(&col), 4);
else //note magic constants!
    services->setVertexShaderConstant(reinterpret_cast<f32*>(&col), 9, 1);
{% endhighlight %}
  * Ogre3D
{% highlight C++ %}
fragment_program_ref GroundPS
{
    param_named color float4 0
}
{% endhighlight %}
{% highlight C++ %}
pParams = pPass->getFragmentProgramParameters();
pParams->setNamedConstant("color", Ogre::Vector4(0.0,0.0,0.0,1.0));
{% endhighlight %}
  * Horde3D
{% highlight C++ %}
Horde3D::setMaterialUniform(m_Material, "color", 0.0, 0.0, 0.0, 1.0);
{% endhighlight %}
  * Three.js
{% highlight JavaScript %}
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
{% endhighlight %}
  * gfx-rs (for comparison)
{% highlight Rust %}
#[shader_param(MyBatch)]
struct Params {
    color: [f32, ..4],
}
let data = Params {
    color: [0.0, 0.0, 0.0, 1.0],
};
{% endhighlight %}

### SYF 101
SYF: Shoot Yourself in the Foot = "to do or say something that causes problems for you".

Notice how almost every implementation requires you to specify the parameter name as a string. This forces the engine to go through all known parameters and compare them with your string. Obviously, this work is wasted for any subsequent calls. It is also a violation of the [DRY](http://en.wikipedia.org/wiki/Don%27t_repeat_yourself) principle and a potential hazard: every time you ask to match the parameter by name, there is a chance of error (parameter not found because you copy-pasted the name wrong?).

In some engines, you can get a handle to the parameter like this:
{% highlight Rust %}
let var_color = program.find_parameter("color");
program.set_param_vec4(var_color, [0.0, 0.0, 0.0, 1.0]);
{% endhighlight %}
This is a bit more verbose, and partially solves the problem, but clearly "color" is still repeated twice here (as a string and a part of the variable name). Besides, another hazard remains - what if a given parameter has an incompatible type with what shader expects?

Three.js comes the closest to being safe - your variable name is used to query the shader, and the type can be verified inside `MeshShaderMaterial` call. Note, however, that in _JavaScript_ you can change the variable type at run-time, which raises the SYF factor significantly.

### Our custom solution in gfx-rs

We are using a procedural macro in Rust to generate the following code at compile time:

1. An associated `Link` structure. It has the same fields as the target one, but the types are replaced by the corresponding variable indices.
2. Implementation of `create_link()` - a function that constructs the `Link` structure by querying a compiled shader for needed variables.
3. Implementation of `fill_params()` - a function that fills up the parameter value, which can be uploaded to GPU.
This is all done behind the `shader_param` attribute:
4. Creates a type alias to the `RefBatch<L ,T>`, named `MyBatch` (see the macro parameter).
{% highlight Rust %}
#[shader_param(MyBatch)]
struct MyParam {
    color: [f32, ..4],
}
{% endhighlight %}
Generated code:
{% highlight Rust %}
struct MyParam {
    color: [f32, ..4],
}
struct _MyParamLink {
    color: ::gfx::shade::VarUniform,
}
type MyBatch = ::gfx::batch::RefBatch<_MyParamLink, MyParam>;
#[automatically_derived]
impl ::gfx::shade::ShaderParam<_MyParamLink> for MyParam {
    fn create_link(__arg_0: Option<MyParam>, __arg_1: &::gfx::ProgramInfo)
     -> Result<_MyParamLink, ::gfx::shade::ParameterError> {
        ::std::result::Ok(_MyParamLink{color:
                                           match __arg_1.uniforms.iter().position(|u|
                                                                                      u.name.as_slice()
                                                                                          ==
                                                                                          "color")
                                               {
                                               Some(p) =>
                                               p as
                                                   gfx::shade::VarUniform,
                                               None =>
                                               return Err(gfx::shade::ErrorUniform("color".to_string())),
                                           },})
    }
    fn fill_params(&self, __arg_0: &_MyParamLink,
                   __arg_1: ::gfx::shade::ParamValues) {
        match *self {
            MyParam { color: ref __self_0_0 } => {
                use gfx::shade::ToUniform;
                __arg_1.uniforms[__arg_0.color as uint] =
                    Some((*__self_0_0).to_uniform());
            }
        }
    }
}
{% endhighlight %}

The end product of this macro is a `MyBatch` type that we can use to create batches with `MyParam` parameters:
{% highlight Rust %}
let batch: MyBatch = context.batch(...).unwrap();
{% endhighlight %}
The `unwrap()` here ignores these possible errors (listing only those related to shader parameters):
  * the structure doesn't provide a parameter that shader needs
  * a shader parameter is not covered by the structure
  * some parameter type is not compatible between the shader and the structure

Later, you provide the `MyParam` instance by reference for every draw call with this batch:
{% highlight Rust %}
let data = MyParam {
    color: [0.0, 0.0, 1.0, 0.0],
};
renderer.draw((&batch, &context, &data), &frame);
{% endhighlight %}
Notice that the data is decoupled from the batch right until the draw call, yet the batch has all the type guarantees about data safety of the shader parameters, thus `draw()` can not fail.

### Serializable solution

Obviously, forcing the type to be dependent on shader parameters prevents the user from loading it at run-time. We are still working towards a more conventional solution for this use-case, and will describe it later on in a separate post.

### Conclusion

In gfx-rs all the parameter queries and type verifications are done at initialization time. Once you got the batch, working with it is 100% safe. You are forced *by the compiler* to set the initial values (when `MyParam` is created), and to preserve their types through the execution. There is zero run-time overhead (all the parameters are uploaded to GPU using their indices), zero memory overhead (we are not allocating a `Matrix4` to cover any parameter needs), and all the boilerplate is hidden from the user. Thus, we are able to bring SYF factor to the minimum, meanwhile improving ergonomics and efficiency.
