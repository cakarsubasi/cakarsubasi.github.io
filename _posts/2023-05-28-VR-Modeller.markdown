---
layout: post
title: "3D Modelling in Virtual Reality Project Website"
date: 2023-05-28 10:00:00 +0300
author: "Cem Akarsubaşı"
---

Welcome to our project website! Instead of some fancy overdesigned css,
we thought that we would share some project history.

Our project is a CAD or a 3D graphics application. CAD stands for
Computer Assisted Design. CAD programs are used in a variety of
industries to meet specific goals. Our application is in 3D graphics
and mainly covers the area of modelling. 

I came up with the idea with this project nearly two years ago, when I first
learned 3D modelling in Blender, one of the most impressive FOSS projects
in history. I was both interested in the low level process of how
Blender manipulated 3D meshes, but I was also interested in what would it
take to recreate such a thing.

I also learned 3D modelling specifically for asset creation in VR applications.
This provided the project motivation and gave it a nice little gimmick. So,
the project is in two parts approximately:

- Create a library that can represent 3D meshes, manipulate them, import and export them.
- Build a VR application on top of that library to actually achieve these goals.

I have mostly worked on the former goal while my project teammates have worked
on the latter.

There were several additional goals besides these:

- Accessibility: accessibility is a frequent problem in VR applications
    and I wanted our application to be "accessibility aware" at least
- Cross platform: build for both PC and Meta Quest 2. Simple enough.

Our application is built on Unity Engine. Building our application without
an existing foundation within 4 months simply would not be possible
without such a foundation.

# Meet "UMesh"

I would like to open this part by acknowledging [Jasper Flick](https://catlikecoding.com/unity/tutorials/)
and his excellent tutorial series on Unity, it would have been considerably
more difficult to get this project started without his great documentation
on how Unity handles and draws 3D objects.

In a nutshell, UMesh is a framework for managing arbitrary non-manifold
geometry strongly bound to Unity. The name is a reference to Blender's
[BMesh](https://github.com/blender/blender/blob/main/source/blender/bmesh/bmesh_class.h)
although I originally just called it "EditableMesh". 

Initially, it looked something like this (ignore the cube in the background):

![](/assets/images/vr_modelling/first-quad.png)

It is black as the normal vectors for the vertices are 0, and Unity uses
the normals for correct lighting.

![](/assets/images/vr_modelling/first-cube.png)

Unity has its own somewhat configurable format for meshes, it goes something
like this in UMesh:

{% highlight cs %}
var vertexAttributes = new NativeArray<VertexAttributeDescriptor>(
    4, Allocator.Temp, NativeArrayOptions.UninitializedMemory);

// positions
vertexAttributes[0] = new VertexAttributeDescriptor(
    attribute: VertexAttribute.Position);
// normals
vertexAttributes[1] = new VertexAttributeDescriptor(
    attribute: VertexAttribute.Normal);
// tangents
vertexAttributes[2] = new VertexAttributeDescriptor(
    attribute: VertexAttribute.Tangent,
    dimension: 4);
// uv coordinates
vertexAttributes[3] = new VertexAttributeDescriptor(
    attribute: VertexAttribute.TexCoord0,
    dimension: 2);

meshData.SetVertexBufferParams(vertexCount, vertexAttributes);
vertexAttributes.Dispose();

meshData.SetIndexBufferParams(triangleCount * 3, IndexFormat.UInt32);

meshData.subMeshCount = 1;
meshData.SetSubMesh(0, new SubMeshDescriptor(0, triangleCount * 3),
    MeshUpdateFlags.DontRecalculateBounds | MeshUpdateFlags.DontValidateIndices);
{% endhighlight %}

Now, this looks like complete gibberish, but let me explain. Unity meshes
have two buffers: the vertex buffer and the index buffer. These buffers 
are allocated with a fixed size and need to be resized if we want to add new
vertices or faces. In addition, the index buffer describes triangles.
The goal of UMesh is to work around these problems.

Here is another early picture:

![](/assets/images/vr_modelling/degenerate-errors.png)

One duty of UMesh is to control the rendering of the Mesh, which is done by
Unity, if the data given to Unity is invalid or erronous, that obviously
causes issues. In addition, it is very important that this is done efficiently
as copying data to the GPU is expensive.


One feature that took among the longest time to implement and is the primary
reason why the project is not a complete joke is extrusion.

Extrusion is this in case you did not know:

![](/assets/images/vr_modelling/extrusion.png)

It is kind of like when you take some geometry, and then yank it out, creating
new geometry underneath it. Here is an early implementation in UMesh:

![](/assets/images/vr_modelling/flipped-faces.png)

That is not right at all! Some of the faces are flipped and appear invisible
as a result. Now, the solution of course is to check neighbouring faces
when creating new faces, but this turns out to be a bit trickier since during
an extrusion operation, a mesh is in a temporarily invalid state.

C# has a garbage collector, this is not a problem, as long as you do not
frequently do small allocations. I am a functional programmer at heart,
so I initially did not notice this was what I was doing with all the LINQ
expressions.

![](/assets/images/vr_modelling/optimization-bad.png)

You can barely see 12.5 ms CPU time on the top right. The editor adds some
overhead, but that is actually really bad and it is actually even worse
than the image shows, because Unity has a "hiccup" every second or so
when the garbage collector kicks in and causes stuttering.

![](/assets/images/vr_modelling/optimization-good.png)

After eliminating the vast majority of allocations, we are down to 4.2 ms per
frame which is much more reasonable. 

Here is a fun fact, every algorithm except for selection in UMesh runs at
`O(n)` time. The reason for this is, every operation that actually modifies
the graph can be done locally. With selection, it depends on the data
structures used since searches can add polynomial factors.

Stats:

    UMesh LOC: 4277
    UMesh Tests LOC: 1800 (I should have written more)

# Frontend

The earliest picture of the frontend looks something like this:

![](/assets/images/vr_modelling/first-ui.png)

It is hard to make UIs in Unity.

One of the biggest problems with the UI design was that a 
significant part of the UMesh backend had to be written before
meaningful interactions could be programmed in.

Perhaps somewhat ironically, the making of our "pretend" modelling
app included modelling in a "real" modelling app. Here is one such
model:

![](/assets/images/vr_modelling/real-modelling.png)

Give me a few more months, and I can probably do this in our application!

# Cut Features

There were many things we wanted to do but could not due to various
project constraints. One such issue was visualizing selections. 
Here is a prototype:

![](/assets/images/vr_modelling/vertex-selection.png)

This was not that straightforward to implement, took two weeks of learning
how GPU programs ("shaders") worked, and likely would require geometry
shaders to work as intended whose usage are heavily discouraged.

Other things we planned during the project proposal was to support
skinned meshes and UV editing. Skinned meshes are meshes that support
"skinning" (very descriptive I know), which allows them to deform relative
to positions of bones, 

A UV is typical name for a vertex's texture coordinates. These are required
for texture mapping, or really if you want your model to have more than a
single color. In our proposal, we wanted to add UV editing to the system,
and in fact, this is one of the earlier issues I have tackled with UMesh.
Here are some meshes with incorrect UV data for instance:

![](/assets/images/vr_modelling/incorrect-uvs.png)

The problem here is that on parts where the UV breaks up (seams), there needs
to be duplicate vertices given to Unity with correct UV coordinates.
UMesh handles this correctly now, but there is no way to interact with it in 
the application.
