---
layout: post
title: "Computer Engineering Bachelor Thesis: 3D Modelling in Virtual Reality"
date: 2024-05-28 10:00:00 +0300
author: "Cem Akarsubaşı"
---

In the past two years, I spent a significant amount of time learning about 3D visual arts.
At first, it was all overwhelming, because there is a lot to learn. In most physical arts,
you have to work with the medium to get the best result. It could be oil, ballpoint pencil,
clay and so on. The medium itself becomes an abstraction but also the biggest limitation. 
In digital 2D painting, you can specify the exact color value of any pixel, but this ends up
being too difficult, so artists use many of the same abstractions that physical tools provide
but they have the advantage of being able to combine all these tools as well as having abilities
that these existing tools do not have.

With 3D arts, things are even more complicated. Now we have a 

Making a 3D application involves an incredibly degree of complexity that 2D GUI applications
simply do not have. Doing things from scratch would be involve too much work, so most people
use a game engine. There are several popular third party ones to choose from such as Unity,
Unreal, and Godot. For this project, we used Unity. Unity is a very popular choice for 
smaller teams due to its reasonable licensing, rich ecosystem of addons and its friendly
and powerful scripting API. Prior experienced also played a strong role in picking Unity.

The project as far as I am concerned is in two parts. One part involved 

# The Vertex, the Edge, the Face, And the Mesh

Before understanding the challenge of making a 3D modelling application, we need to understand
how computers can even represent 3D objects. Let's use Unity as an example. Somewhat simplified,
3D models in Unity consist of:

- Stream descriptors that contain information about the vertex stream and the index stream
- The vertex stream(s) containing vertices which their position, their normal vector, their tangent vector, and a texture coordinate[^note-tex-coord]
- The index stream which has triangles that contain three vertex indices each.

However, this is both too little and too much. For starters, the normal vector and tangent
vector we do not strictly need. These are used for lighting, and since they do not change,
Unity wants them to be precalculated for performance reasons. On the flip side, since each
vertex can only contain one normal, one tangent, and a texture coordinate, you need to have
duplicate vertices at the same position if you have an object with texture seams or flat
shading.[^flat-shading-note] 

In most use cases where a developer would want to directly edit mesh data in Unity, these
limitations do not matter, however when we want to make modelling software, we need to hold
the data and make sure it has some semblance of sanity.

Writing algorithms that operate on meshes is a lot like 

# Virtual Reality


Although it is possible that someone will eventually make a new revolutionary 3D graphics
application in Virtual Reality (if you're going to bet your money on anyone, I would recommend
Adobe), it is much more likely that the first "winner" is going to be a suitably intuitive 
extension to a mature existing application like Blender or Maya.

# Notes

[^note-tex-coord]: Technically you can have up to 8 texture coordinates and texture 
    coordinates can be used to store arbitrary data, but we need to write special 
    shaders in order to take advantage of this and the utility of it can be rather limited.

[^flat-shading-note]: Here is an example:

    ![](/assets/images/smooth_vs_flat_shading.png)

    The GPU interpolates the normal angles for each fragment between each vertex, which enables finite geometry
    to look perfectly smooth. There are some ways of dodging this by modifying the fragment program or using
    a geometry program.