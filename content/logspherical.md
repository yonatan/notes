Title: Log-spherical Mapping in SDF Raymarching
Author: Pierre Cusa <pierre@osar.fr>
Created: February 2019
Source: https://github.com/pac-dev/notes

![recursive lotus](recursive_lotus_1_white.jpg)
{: .wideImg }

In this post, I'll describe techniques for manipulating [signed distance fields][] (SDFs) allowing the creation of self-similar geometries like the flower above. Although these types of geometries have been known and renderable for a long time, I believe the techniques described here offer much unexplored creative potential, since they're based on SDF raymarching, making it possible to explore previously impossible realtime renderings of self-similar geometries with an infinite level of visual recursion on the average current-gen graphics card. This is done by crafting distance functions based on the log-spherical mapping, which I'll explain starting with basic tilings and building up to 3D self-similar geometries.


## Log-polar Mapping in 2D

Conversions between log-polar coordinates $(\rho, \theta)$ and Cartesian coordinates $(x, y)$ are defined as:

$$
\tag{1}
\begin{cases}
  \rho = \log\sqrt{x^2 + y^2}, \cr
  \theta = \arctan\frac{y}{x}.
 \end{cases}
$$
$$
\tag{2}
\begin{cases}
  x = e^\rho\cos\theta,\hphantom{\sqrt{x^2+}} \cr
  y = e^\rho\sin\theta.
 \end{cases}
$$

The conversion $(2)$ can be used as an active transformation, the inverse log-polar map, in which we can consider $(\rho, \theta)$ as the Cartesian coordinates before the transformation, and $(x, y)$ after the transformation. Applying this to any geometry that regularly repeats along the $\rho$ axis results in a self-similar geometry. Let's apply it to a grid of polka dots to get an idea of how this happens.

figure: shaderFig
caption: Log-polar tiling in 2D. Controls perform translation before mapping. Red axis: $\rho$, green axis: $\theta$
code: logpolar_polka.glsl

Note how regular tiling yields self-similarity, $\rho$-translation yields scaling, and $\theta$-translation yields rotation. Also visible, the mapping function's domain does not cover the whole space, but is limited to $\theta\in\left]-\pi, \pi\right]$ as represented by the dark boundaries.


## Estimating the Distance

The above mapping function works well when applied to an implicit surface, but in order to prepare for 3D raymarching, we should apply it to a distance field, and obtain an equally correct distance field. Unfortunately, the field we obtain is heavily distorted as visualized below on the left side: the distance values are no longer correct since they are shrunk along with the surface. In order to correct this, consider that the mapping effectively scales geometry by the exponent of $\rho$. Furthermore, as Inigo Quilez [notes][iqnotes], when scaling a distance field, we should multiply the distance value by the scaling factor. Thus the simplest correction is to multiply the distance: $d\mapsto d e^\rho$. In most cases, this correction gives a sufficiently accurate field for raymarching.

figure: shaderFig
caption: Distance field correction. Left: distorted field, right: corrected field. Colors represent the distance field's contour lines, red: negative, blue: positive.
url: https://github.com/pac-dev/notes
code: distortion_polka.glsl


## Log-polar Mapping in 3D

The above 2D map can be simply applied in 3D space by picking two dimensions to transform. However, this means the geometry will get scaled in those two dimensions but not in the remaining dimension, causing quite a bit of stretching and field distortion (in fact, geometry will be infinitely thin at the origin, and infinitely fat at the horizon). This can be corrected by scaling the remaining dimension proportionally to the others, using the same $e^\rho$ factor as before. The resulting distance function works well with raymarching.

figure: shaderFig
runnable: true
caption: Log-polar mapping applied to two dimensions of a 3D geometry.
code: logpolar3d_rods.glsl


## Log-spherical Mapping

The above technique lets us create, without loops, self-similar surfaces with an arbitrary level of detail. But the recursion still only happens in two dimensions - what if we want fully 3D self-similar geometries? Log-polar coordinate transformations can be generalized to more dimensions. Conversions between log-spherical coordinates $(\rho, \theta, \phi)$ and Cartesian coordinates $(x, y, z)$ are defined as:

$$
\tag{3}
\begin{cases}
  \rho = \log\sqrt{x^2 + y^2 + z^2}, \cr
  \theta = \arccos\frac{z}{\sqrt{x^2 + y^2 + z^2}} \cr
  \phi = \arctan\frac{y}{x}.
 \end{cases}
$$
$$
\tag{4}
\begin{cases}
  x = e^\rho\sin\theta\cos\phi, \hphantom{\log x^2} \cr
  y = e^\rho\sin\theta\sin\phi, \cr
  z = e^\rho\cos\theta.
 \end{cases}
$$

Once again, conversion $(4)$ can be used as an active transformation. This is the inverse log-spherical map, which results in a self-similar geometry when applied to a regularly repeating geometry.

figure: shaderFig
runnable: true
caption: Log-spherical mapped geometry.
url: https://github.com/pac-dev/notes
code: logspherical_cage.glsl


## Similarity

With the above technique, any geometry we define will be warped into a spherical shape, with visible pinching at two poles. What if we want to create a self-similar version of any arbitrary geometry, without this deformation? For this, the mapping needs to be *similar*, that is, preserving angles and ratios between distances. To achieve this, we apply the forward log-spherical transform before applying its inverse, and perform regular tiling on the intermediate geometry. This means we can define any arbitrary geometry contained in a [spherical shell][], and have it infinitely repeated inside and outside of itself - without relying on loops. Details can be found in the source to this example:

figure: shaderFig
runnable: true
caption: Similar log-spherical mapped geometry.
url: https://github.com/pac-dev/notes
code: similarity_boxes.glsl


## Implementation Challenges

It's important to note that the above techniques do *not* produce exact distance fields. When raymarching, this can cause rays to overshoot the desired surface, resulting in visible holes under some angles. To better understand the problem, we can take a cross-section of a raymarched scene and color it according to its distance value.

figure: shaderFig
caption: Log-spherical scene with distance gradient shown in a cross-section.
url: https://github.com/pac-dev/notes
code: implementation_cubes.glsl

Under the log-spherical mapping, repeated tiles become spherical shells contained inside each other. Here, it becomes visible that at the edges between shells, there is a discontinuity in the distance gradient, because each tile only provides distance to the geometry inside of it. In this case, modifying the "twist" parameter worsens the discontinuities because the tiled shapes are not aligned with each other. This is a recurring problem when working with this technique, and I've settled on 3 main ways to counter it, all of which are tradeoffs against rendering performance:

- *Add hidden geometry to bring rays into the correct tile.* In some cases, we know that the final geometry will approximately conform to a certain plane or other simple shape. We can estimate this shape's distance, cut off its field below a certain threshold so it's never actually hit, and combine it into the scene.
- *Combine the SDF for adjacent tiles into each tile,* so that the distance value takes into account more surrounding geometry. This is especially useful when the contents of each tile are close to, or touching, each other.
- *Shorten the raymarching steps.* This can be done globally, or only locally in problematic parts of the scene.


## Examples

figure: shaderFig
runnable: true
caption: "Recursive Lotus"
url: https://github.com/pac-dev/notes
code: recursive_lotus.glsl

figure: shaderFig
runnable: true
caption: "Cube Singularity"
url: https://github.com/pac-dev/notes
code: cube_singularity.glsl



## Prior Art, References and More

While I haven't found any previous utilization of the log-spherical transform in distance fields, there are many related and similar techniques that have been creatively used:

- The 2D log-polar transform, and its related logarithmic spirals, have been used since the early days of fragment shader programming. They have even developed into much more interesting and complex structures and related transformations. A nice example: Flexi's [Bipolar Complex](https://www.shadertoy.com/view/4ss3DB).
- Applying the 2D log-polar transform to 3D SDFs has been done previously, but apparently only as a cursory exploration, see knighty's [Spiral tiling](https://www.shadertoy.com/view/ls2GRz) shader.
- Self-similar structures and zoomers have also been implemented in shaders using loops, as opposed to the loopless techniques described here. Looping strongly limits the number of visible levels of recursion for realtime, however, it allows more freedom in geometry placement while giving an exact distance. Some examples of this: [Gimbal Harmonics](https://www.shadertoy.com/view/llS3zd); [Infinite Christmas Tree](https://www.shadertoy.com/view/4ltBzf). This might also be how Quite's [zeo-x-s](https://youtu.be/eKbTaxDEtXY?t=333) was made.
- As mentioned in the introduction, visually recursive structures have long been known and renderable with technologies other than SDF raymarching. There are many examples out there, a popular one being the works of [John Edmark](http://www.johnedmark.com).




[signed distance fields]: http://jamie-wong.com/2016/07/15/ray-marching-signed-distance-functions/
[implicit surface]: https://en.wikipedia.org/wiki/Implicit_surface
[Michael Walczyk]: http://www.michaelwalczyk.com/blog/2017/5/25/ray-marching
[iqnotes]: https://www.iquilezles.org/www/articles/distfunctions/distfunctions.htm
[spherical shell]: https://en.wikipedia.org/wiki/Spherical_shell