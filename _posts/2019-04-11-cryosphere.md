---
layout: post
title: Cryosphere - A visual node-based shader editor
date:   2019-04-11 19:44
description: An integrated shader editor in our engine
toc: true
tags: engine shader cryo
---

Cryosphere is a node-based visual shader editor, that uses ImGUI for the visual layer and exports as saved node graphs and HLSL code.

# Aspirations and goals
My main goal heading into the development and design of Cryosphere was to create a more intuitive and lower friction method for creating and maintaining our shader pipeline. It should provide a faster method for prototyping events and integrate seamlessly with out engine, while also being as portable and lightweight as possible.

In order to achieve this Cryosphere should provide close-to-live updates in a preview window and properly type-check the node graph prior to code generation so that the preview window can always be correctly displayed. Aditionally it should provide a simple and minimal interface to the engine in order to maintain the portability.

# A note on Typesafety
Keeping the node-graph type-safe while also providing a good generic toolset of nodes for mathematic operations is quite tricky! I wanted to keep the node-descriptions as rigid as I could while also providing flexibility for more common operations. Therefore, some nodes have pins with dynamic types. Most math operations take inputs `a` and `b`, to provide output `result`, and most mathematical nodes in Cryosphere deduce the type of `result` from the inputs connected to it, this is an exception in Cryosphere, and most other nodes have more rigid typing.

For convieniently splitting vectors into components and subvectors, Cryosphere has a special "swizzle" node. Upon creating the node the user is prompted to enter a "swizzle expression". A swizzle expression is a comma-separated list of vector subscripts, using the namesets `xyzw` or `rgba`. For example the swizzle expression `xyz, xw, ggb, a` produces a node that looks like this.
![swizzlenode]({{ './assets/images/swizzle_node.png' | relative_url}}){: .center-image }
The type of the input pin is determined from the elements used in the swizzle expression, so the example above would require a 4-component vector as it uses the `w` and `a` subscripts.

In order to aid the user with understanding what types are being used where, the connections between nodes show the number of components in the vector being transferred.
![components]({{ './assets/images/components.png' | relative_url}}){: .center-image }

# Under the hood
At the core of Cryosphere lies of course the node system, which contains a list of node-definitions for all the various nodes that the user can create. The node definitions in turn create the node and attach all the relevant pins to it, and are responsible for generating the code they represent later when the final shader is built.

A core set of nodes is always availible in Cryosphere, these nodes are responsible for basic things such as creating and destructing vectors, performing common math operations, and introducing constant values to the shader. These are all defined using the C++ API and are submitted to the edtior at startup.

The rest of the nodes are defined in what Cryosphere refers to as the "Shader model". The shader model is simply a text file (that can include others) that defines the different types of nodes that can be used, as well as the framework for the actual shader code that will be produced. There are a few components to the sahder model:

## Shader models and nodes
### Function nodes
A function node simply parses a standard HLSL function and creates a node that accepts the correct inputs and produces a single output pin. These are useful for a lot of things and are by far the simplest way to extend the functionality of the editor. A simple HLSL function that looks like this:
>HLSL
{:.filename}
{% highlight java %}
float4 SampleAlbedo(float2 uv)
{
    return AlbedoTexture.Sample(defaulySampler, uv);
}
{% endhighlight %}
Procudes a very simple node that looks like this (note how it correctly deduces the types):
![samplealbedo]({{ './assets/images/sample_albedo.png' | relative_url}}){: .center-image }
The vast majority of the nodes created for the example shader model for our engine exist as function nodes, and I suspect these would be the most common in the more general case.

### Mixin nodes
Mixin nodes are a more advanced way to extend the functionality of the edtior, and are necessary for nodes with multiple output pins. They closely resemble a macro system and hace some special syntax, but are as a result very powerful. Consider this example mixin node below:
>Shader Model
{:.filename}
{% highlight java %}
#mixin "Get Material" %albedo: Vector3, %uv: Vector2 -> $emissive: Vector3, $metalness: Scalar, $roughness: Scalar
float3 material = materialTexture.Sample(linearWrapSampler, %uv).rgb;
$emissive = %albedo * material.b;
$metalness = material.r;
$roughness = material.g;
#endmixin
{% endhighlight %}
Which produces a node that look like this:
![getmaterial]({{ './assets/images/get_material.png' | relative_url}}){: .center-image }
And the generated HLSL looks like this:
>HLSL
{:.filename}
{% highlight java %}
float3 _out_126;
float _out_127;
float _out_128;
{
    float3 material = materialTexture.Sample(linearWrapSampler, _out_125).rgb;
    _out_126 = _out_124 * material.b;
    _out_127 = material.r;
    _out_128 = material.g;
}
{% endhighlight %}
The extra scoping is to avoid any naming conflicts with the variables declared inside the mixin.

### Fixed nodes
There are a few nodes that always need to be present in order for a shader to be generated, notable the input and output nodes. These are generated from the function signature provided in the shader model file; if your shader contains the definition:
>HLSL
{:.filename}
{% highlight java %}
PixelOutput PSMain(PixelInput input) { ... }
{% endhighlight %}
Cryosphere will parse the `PixelInput` and `PixelOutput` structures and generate nodes from them, and place them into the empty node graph. Here's an example of a Cryosphere shader using those nodes.
![inout]({{ './assets/images/intout9.png' | relative_url}}){: .center-image }

## Code generation
