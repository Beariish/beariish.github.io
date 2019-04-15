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
My main goal heading into the development and design of Cryosphere was to create a more intuitive and lower friction method for creating and maintaining our shader pipeline. It should provide a faster method for prototyping events and integrate seamlessly with our engine, while also being as portable and lightweight as possible.

In order to achieve this, Cryosphere should provide close-to-live updates in a preview window and properly type-check the node graph prior to code generation so that the preview window can always be correctly displayed. Aditionally, it should provide a simple and minimal interface to the engine in order to maintain the portability.

# A note on Typesafety
Keeping the node-graph type-safe while also providing a good generic toolset of nodes for mathematic operations is quite tricky! I wanted to keep the node-descriptions as rigid as I could while also providing flexibility for more common operations. Therefore, some nodes have pins with dynamic types. Most math operations take inputs `a` and `b`, to provide an output `result`, and most mathematical nodes in Cryosphere deduce the type of `result` from the inputs connected to it; this is an exception in Cryosphere, and most other nodes have more rigid typing.

To convieniently split vectors into components and subvectors, Cryosphere has a special "swizzle" node. Upon creating the node, the user is prompted to enter a "swizzle expression". A swizzle expression is a comma-separated list of vector subscripts, using the namesets `xyzw` or `rgba`. For example, the swizzle expression `xyz, xw, ggb, a` produces a node that looks like this.
![swizzlenode]({{ './assets/images/swizzle_node.png' | relative_url}}){: .center-image }
The type of the input pin is determined from the elements used in the swizzle expression, so the example above would require a 4-component vector as it uses the `w` and `a` subscripts.

In order to aid the user with understanding what types are being used where, the connections between nodes show the number of components in the vector being transferred.
![components]({{ './assets/images/components.png' | relative_url}}){: .center-image }

# Under the hood
At the core of Cryosphere lies, of course, the node system, which contains a list of node-definitions for all the various nodes the user can create. The node definitions in turn create the node and attach all the relevant pins to it, and are responsible for generating the code they represent later when the final shader is built.

A core set of nodes is always availible in Cryosphere, these nodes are responsible for basic things such as creating and destructing vectors, performing common math operations, and introducing constant values to the shader. These are all defined using the C++ API and are submitted to the edtior at startup.

The rest of the nodes are defined in what Cryosphere refers to as the "Shader model". The shader model is simply a text file (which can include others) that defines the different types of nodes that can be used, as well as the framework for the actual shader code that will be produced. There are a few components to the shader model:

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
Produces a very simple node that looks like this (note how it correctly deduces the types):
![samplealbedo]({{ './assets/images/sample_albedo.png' | relative_url}}){: .center-image }
The vast majority of the nodes created for the example shader model for our engine exist as function nodes, and I suspect these would be the most common in the more general case.

### Mixin nodes
Mixin nodes are a more advanced way to extend the functionality of the edtior, and are necessary for nodes with multiple output pins. They closely resemble a macro system and have some special syntax, but are more powerful as a result. Consider this example mixin node below:
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
### Prerequisites
Before Cryosphere can attempt to generate shader code, a few conditions have to be met.

* All nodes need to have every input pin connected and satisfied.
* All connected pins need to have matching types, this should always be guarenteed by the UI, but the code generator checks again just in case.
* There can be no looping node connections.

### The method
The code generator then starts stepping through every node, mapping the output pins to intermediate variables named `_out_[index]` where `index` starts at 0 and increments for every variable generated. Granted this creates a lot of unnessecary variables, and even some that are never used, but that's not enough of an issues to be a concern currently. Maybe as complexity increases there'd be cause for doing a second optimization pass, but that's for a later post.

Function nodes work very simply, the generator just calls the function and assignes the result to a new variable, like so:
>HLSL 
{:.filename}
```java
float4 _out_8 = SampleAlbedo(_out_2);
float4 _out_9 = ApplyAO(_out_8, _out_2);
```

Mixins declare the output `$`-variables first, enter a new scope, and string-replace the input `%`-variables with the corresponding `_out_[index]`. The extra scope exists to make sure that there are no naming conflicts with mixin nodes.
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

The resulting code is then inserted into the function signature defined in the shader model. The definition looks like the following:
>HLSL
{:.filename}
```java
#entrypoint PixelOutput PSMain(PixelInput input)
```

And the code generated is placed at the bottom of the shader file, with all mixins and the `#entrypoint` directive stripped, the example here has been manually prettified a little for readability:
![codegen]({{ './assets/images/codegen.png' | relative_url}}){: .center-image }

>HLSL
{:.filename}
```java
<struct definitions>
<function definitions>
PixelOutput PSMain(PixelInput input)
{
    float4 _out_0 = input.myPosition;
    float2 _out_1 = input.myUV;
    float4 _out_2 = input.myNormal;
    float4 _out_3 = input.myTangent;
    float4 _out_4 = input.myBinormal;
    float2 _out_5 = input.myUV2;
    float4 _out_6 = input.myColor;
    float4 _out_7 = input.myWorldPos;
    float4 _out_8 = input.myViewPos;
    float4 _out_9 = input.myLightPos;
    float4 _out_10 = input.myScreenPos;
    float4 _out_11 = input.myPositionPrev;

    float4 _out_12 = SampleAlbedo(_out_1);
    float4 _out_13 = ApplyAO(_out_12, _out_1);

    flout3 _out_14 = _out_12.rgb;

    float3 _out_15;
    float _out_16;
    float _out_17;
    {
        float3 material = materialTexture.Sample(linearWrapSampler, _out_1).rgb;
        _out_15 = _out_14 * material.b;
        _out_16 = material.r;
        _out_17 = material.g;
    }

    float3 _out_18 = _out_13.rgb;
    float3 _out_19 = _out_18 + _out_15;

    float _out_20 = _out_19.r;
    float _out_21 = _out_19.g;
    float _out_22 = _out_19.b;

    float _out_23 = 1.00000;

    float4 _out_24 = float4(_out_20, _out_21, _out_22, _out_23);

    output.myColor = _out_24;
}
```

## Portability
Earlier in this post I mentioned how I wanted to keep Cryosphere as portable and embeddable as possible, so here's a few notes on how I achieved that.

Cryosphere builds as a standalone shared library, and hooks into the engine with only a few callbacks, one of which expects the engine's ImGui context for drawing.
>C++
{:.filename}
```java
void Cryo_Init(void);
void Cryo_SetImGuiContext(void* context);
void Cryo_Draw(void);
```

Using only these exported symbols you can use Cryosphere with it's built-in functionality and core feature set. Everything relevant is exported though, and choosing to include Cryosphere's headers allows you to define new types of nodes, extend the shader module system, and hook into code generation and saving.

As the shaders generated by Cryosphere don't depend on the library itself, Cryosphere can be compltely omitted in a release or retail build.

# Conclusion
Cryosphere produces working shaders and enables a much smoother workflow and prototyping process for Techincal Artists and Artists to utilize when making shaders for our games. The performance difference between a Cryo-generated shader and an equivalent hand-written one is negligible within the scope that Cryosphere was intended for.

Ideally Cryosphere would offer more tools to be more user friendly, such as seaching the node creation menu, and allowing for comment blocks and re-routing the pin connections visually; but the entire core feature set is there and ready to be built upon.