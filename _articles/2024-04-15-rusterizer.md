---
layout: post
title: Rusterizing from Scratch
preview_video: ./assets/videos/OkCube.mp4
---

In every year at BUAS, we have the Rusterizer masterclass, where we create a software rasterizer from scratch using Rust. In this post I will go over the main features I have implemented in my software rasterizer written in Rust.

<video width="100%" height="100%" controls>
  <source src="../assets/videos/OkCube.mp4" type="video/mp4">
</video>


## Table of Contents
1. [Setup](#libraries-used-and-setup)
2. [Textures and Screen Output](#textures-and-screen-output)
1. [Vertex Transformations and Triangle Clipping](#vertex-transformation-and-triangle-clipping)
2. [Screen mapping and Depth testing](#screen-mapping-and-depth-testing)
1. [Depth Correction and Shading](#depth-correction-and-shading)
2. [Conclusion](#conclusion)


## Libraries Used and Setup

Even though this is from scratch I obviously resorted to using libraries for a couple of things that are out of scope from graphics. 

I utilized the `minifb` crate for windowing and input and `glam` for math, since coding a matrix, vec4, quaternion data types by hand is very tedious. I also used `stb_image` for loading images from disk.

## Textures and Screen Output

In order to output an image to the screen, `minifb` expects you to provide a `u32` texture buffer, as well as width and height. Because of this, I created a ``Texture`` struct that can act as texture input for my renderer, but also as an output:

```rust
#[derive(Default)]
pub struct Texture {
    data: Vec<u32>,
    width: usize,
    height: usize
}

impl Texture {
    pub fn new(width: usize, height: usize) -> Self {
        Self { data: vec![0; width * height], width, height}
    }

    pub fn from_data(data: Vec<u32>, width: usize, height: usize) -> Self {
        debug_assert!(width * height == data.len());
        Self { data, width, height}
    }

    pub fn width(&self) -> usize { self.width }
    pub fn height(&self) -> usize { self.height }

    //...
}
```

I can then render directly an image to the screen. This was pretty useful to test if my image loading was working fine.

```rust
fn main() {
    
    let mut window = create_window().unwrap();

    let texture = load_image_file(std::path::Path::new("assets/icon.png")).unwrap();

    while window.is_open() {

        window.update_with_buffer(
            texture.as_slice(),
            texture.width(),
            texture.height()
        ).unwrap();
    }
}
```

## Vertex Transformation and Triangle Clipping

My rasterizer takes as an input a set of vertex positions, colours and uvs, as well as a slice of triangle indices. The vertex shader takes this input and outputs clipped and transformed triangles in normalized device coordinates:

```rust
#[derive(Default)]
pub struct VertexShader {
    pub view: glam::Mat4,
    pub projection: glam::Mat4,
}

impl VertexShader {

    pub fn dispatch(
        &self,
        vertex_in: &VertexInput,
        indices: &[usize])
    -> (VertexOutput, Vec<usize>) { /*...*/ }
}
```

The hardest thing to implement in this project was the triangle clipping. I had to use the Sutherland-Hogdman line clipping algorithm in 4D (Homogenous clip space).

```rust
let clip_planes = [
    Vec4::new(1.0, 0.0, 0.0, 1.0), //Left
    Vec4::new(-1.0, 0.0, 0.0, 1.0), //Right
    Vec4::new(0.0, 1.0, 0.0, 1.0), //Bottom
    Vec4::new(0.0, -1.0, 0.0, 1.0), //Top
    Vec4::new(0.0, 0.0, 1.0, 0.0), //Near
    Vec4::new(0.0, 0.0, -1.0, 1.0), //Far
];

for plane in clip_planes {

    let input_list = output_list.clone();
    output_list.clear();

    for i in 0..input_list.len() {

        let current_point = input_list[i];
        let next_point = input_list[(i+1) % input_list.len()];

        let current_inside = plane.dot(current_point.0) >= 0.0;

        if current_inside {
            output_list.push(current_point);
        }

        if let Some(t) = homogenous_intersect(current_point.0, next_point.0, plane) {

            let interpolated = lerp(current_point.0, next_point.0, t);
            let barycentric_coords =  lerp(current_point.1, next_point.1, t);
            output_list.push((interpolated, barycentric_coords));
        }
    }
}

output_list
```

Triangle clipping returns a set of vertices, which are the clipped vertices of the triangle, as well as the barycentric coordinates of the vertices for interpolating all the vertex attributes for the new vertices.

The intersection between a 4D plane and a line segment was quite hard to derive, but the clip planes have the interesting property of being defined by just their normal vector:

```rust
pub fn homogenous_clip(a: Vec4, b: Vec4, plane: Vec4) -> Option<f32> {
    let line_vector = b - a;

    let div = plane.dot(line_vector);
    if div.abs() < std::f32::EPSILON { return None; }

    let t = -plane.dot(a) / div;
    if t > 0.0 && t < 1.0 { Some(t) }
    else { None }
}
```

Then I apply the clipping inside the vertex shader and process all the new vertices and triangulate.

```rust
let clip_coordinates = VertexShader::world_to_clip_space(&mvp, &vertices);

//Frustum culling
if math::should_cull_triangle(clip_coordinates[0], clip_coordinates[1], clip_coordinates[2]) { continue; }         

//Homogenous clipping
let clipped_vertices = math::clip_homogenous_triangle(&clip_coordinates);
if clipped_vertices.is_empty() { continue; }

for (vert, bary) in &clipped_vertices {
    
    let inv_depth = 1.0 / vert.w;
    out_vertex.ndc_positions.push((*vert * inv_depth).truncate().extend(inv_depth));

    //Interpolate attributes

    let colours = VertexInput::retrieve(&vertex_in.colours, triangle_indices);
    let uvs = VertexInput::retrieve(&vertex_in.uvs, triangle_indices);

    let result_colour = math::barycentric_lerp(*bary, colours[0], colours[1],colours[2]);
    let result_uv = math::barycentric_lerp(*bary, uvs[0], uvs[1],uvs[2]);

    out_vertex.colours.push(result_colour * inv_depth);
    out_vertex.uvs.push(result_uv * inv_depth);
}

//Triangulate
let triangulation_indices: Vec<(usize, usize, usize)> = (1..clipped_vertices.len() - 1)
    .map(|v| { (0, v, v + 1) }).collect();
```

## Screen mapping and Depth testing

The Fragment Shader struct is then in charge of getting the clipped vertices and attributes from the vertex shader and map/shade them in the screen:

```rust
#[derive(Default)]
pub struct FragmentShader {
    pub mesh_texture: Texture,
    pub mesh_sampler: Sampler
}

impl FragmentShader {
    pub fn dispatch(&self,
        out: &mut Texture,
        depth_buffer: &mut DepthTexture,
        vs_output: &VertexOutput,
        indices: &[usize]
    ) { /*...*/ }
}
```

The Depth texture is essentially a single channel float texture that is used for depth buffering. The first step was to use a screen space matrix to map the normalized coordinates to the screen: 

```rust
let half_screen_width = (out.width() as f32) * 0.5;
let half_screen_height = (out.height() as f32) * 0.5;

let screen_space_matrix = glam::Mat3::from_scale_angle_translation(
    glam::Vec2::new(half_screen_width, -half_screen_height),
    0.0,
    glam::Vec2::new(half_screen_width, half_screen_height)
);

let screen_1 = screen_matrix.mul_vec3(v1.truncate().truncate().extend(1.0));
let screen_2 = screen_matrix.mul_vec3(v2.truncate().truncate().extend(1.0));
let screen_3 = screen_matrix.mul_vec3(v3.truncate().truncate().extend(1.0));
```

Then create a bounding box and intersect it with the screen to get the screen region where the triangle lives:

```rust
let triangle_bounds = math::generate_triangle_bounding_box(screen_1.truncate(), screen_2.truncate(), screen_3.truncate());

let screen_bounds = math::bounding_box::BoundingBox::new(
    glam::UVec2::new(0, 0), 
    glam::UVec2::new(out.width() as u32, out.height() as u32)
);

let triangle_bounds = triangle_bounds.intersect(&screen_bounds)?;
```

Finally, we iterate through all the pixels in the resulting bounding box and test if the point is inside the triangle by calculating the barycentric coordinates and depth testing against the value in the current depth buffer.

```rust
let pixel_point = glam::Vec2::new(i as f32 + 0.5, j as f32 + 0.5);

//means the point is inside the triangle
if let Some(weights) = math::barycentric_weights(pixel_point, screen_1.truncate(), screen_2.truncate(), screen_3.truncate()) {
    let depth = weights.dot(glam::Vec3::new(v1.z, v2.z, v3.z));

    if depth_buffer.depth_test(i, j, depth) {
        //Shading happens here
    }
}
```

## Depth Correction and Shading

Finally, we need to apply shading to each pixel. I have no form of lighting, but we have access to uv coordinates and colour values. I simply multiply them:

```rust
let depth_correction = 1.0 / (weights.x * v1.w + weights.y * v2.w + weights.z * v3.w);

let colour = math::barycentric_lerp(weights, colour1, colour2, colour3) * depth_correction;
let uv = math::barycentric_lerp(weights, uv1, uv2, uv3) * depth_correction;

let out_frag = colour * self.mesh_sampler.sample(&self.mesh_texture, uv).truncate();

out.write(i, j, math::colour::vec4_to_hex(out_frag.extend(1.0)));
```

Apart from interpolating the values between each vertex based on the barycentric coordinates, we also have to apply depth correction because this interpolation should happen in 3D and not 2D.

When I forward the vertex attributes I divide them by the depth, which is more efficient than dividing them here.

After this you can get an amazing cube:

<video width="100%" height="100%" controls>
  <source src="../assets/videos/OkCube.mp4" type="video/mp4">
</video>

## Conclusion

Concluding, I had a pretty fun time creating this small project and 
I really enjoyed working with Rust and all the math I got to understand. 

Hopefully I get some time to add model loading, which was really one of the things I am lacking to actually render impressive models.