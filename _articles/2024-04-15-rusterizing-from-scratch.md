---
layout: post
title: Rusterizing from Scratch
preview_video: ./assets/videos/OkCube.mp4
---

The Rusterizer is a masterclass that happens every year here at BUAS, where we use Rust to implement a simple software rasterizer. This is the result of my work during the masterclass, as well as some additional features I have added in the following weeks.

//Video

## Introduction

Rasterization is the most common form of 3D rendering that exists. It essentially revolves around using triangles to render all geometry and surfaces in a 3D environment. Triangles are chosen, because they are the simplest polygon to work with and have some useful mathematical properties that make all the math that goes into rendering slightly easier.

Most rasterization pipelines are implemented in hardware level on modern GPUs (and it was the primary use of GPUs before compute and neural networks became more prevalent). Either way, the pipeline itself follows conventional logic and can be implemented in CPU using basic code, although warn you, performance will not be the same. Reasons to implement a software rasterizer can range from having to support machines that don't natively have a GPU, to a simple learning exercise in how graphics actually work under the hood.

## The Rasterization pipeline

//Rasterization Pipeline

The rasterization pipeline involves many steps, even more depending on which API you are using. However, the basic steps that I will be implementing in my Rusterizer are the following:

- Vertex Shader
- Triangle Clipping
- Screen Mapping
- Depth Testing
- Fragment / Pixel Shading
