**Final Project Report**

Student name: Sébastien Gachoud

Sciper number: 250083


Final render
============

<div class="twentytwenty-container">
    <img src="final_scene/final_scene.png" alt="Final Render">
</div>


Motivation
==========

I used to have a cat in my neighborhood that had this habit of bringing us little dead animals. Therefore, it has been the first thing that came to my mind when I discovered the theme of the project. The unwanted present is represented by the dead bird on the carpet.


Feature list
============

<table>
    <tr>
        <th>Feature</th>
        <th>Standard point count</th>
        <th>Adjusted point count</th>
    </tr>
    <tr>
        <td>Textures</td>
        <td>10</td>
        <td>10</td>
    </tr>
    <tr>
        <td>Cluster Rendering</td>
        <td>10</td>
        <td>10</td>
    </tr>
    <tr>
        <td>Hair Rendering</td>
        <td>60</td>
        <td>60</td>
    </tr>
    <tr>
        <td><strong>Total</strong></td>
        <td>80</td>
        <td>80</td>
    </tr>
</table>



Textures
=========

Textures adds a lot of realism to a scene and the basic implementation to support texture is simple and one of the most useful use of it consists of having a BSDF with different albedo in different area. This can be achieved by mapping the pixels of an image on a surface. 

In this project I only implemented textures support for the diffuse BSDF but it could be also implemented for other BSDF as in microfacet, for example, to replace a constant kd. I followed the implementation proposed by the PBRT-book. The abstraction of a texture allows to choose the return type of its evaluation. For this project the only type currently used is Color3f. To provide a default and basic kind of texture I implemented the constant texture which allows to represent a constant albedo and therefore can be used by default. Since textures might require much more information about the intersection point on the surface than provided by a BSDFQueryRecord it was easier to add a reference to the intersection to the BSDFQueryRecord that could be used to evaluate the texture. The interesting part of the textures here comes with the implementation of the texture from an image. To generalize the mapping between the shape and the image I chose to follow the implementation of the PBRT-book even if this abstraction was not useful for this particular scene. Therefore, I introduced the class TextureMapping2D which represents a mapping between the 2D image and the UV coordinates in general. This class is specialized in the UVMapping2D class to map an image uniformly on the shape according to the UV map. This class also introduce the ability to scale or offset the image on both directions. To keep it simple and precise the scale needs to be set on the size of the image to perfectly match the u and the v to both dimensions. The class ImageTexture allows to use an image as texture and comes with 4 different wrap mode to handle an "out of dimension" mapping: the wrap mode REPEAT which repeat the image, the wrap mode CLAMP which us the border of the image, the wrap mode BLACK which use black when out of range and finally the DEFAULT wrap mode which use a default color when out of range.


In this render, the textures of the pictures are wrapped with a default color of $(1.0, 1.0, 1.0)$.
<div class="twentytwenty-container">
    <img src="cbox_tableaux.png" alt="textures">
</div>


Cluster
=======

To be able to render on multiple machine at the same time I added the possibility to give new arguments to nori. First the possibility to give an ID to the image with the following command: ./nori -id "id" "path to scene xml". The id must be an integer and, to be able to merge the resulting images, the ids should start from $0$ and continue up to the number of images - $1$. The intended behavior expects that all images are run with the exact same xml, meshes and textures. The final image can be obtain by running nori with the following command: ./nori -merge "amount of images" "path to scene xml". This command trigger nori to open all corresponding exr images that are then weighted by $\frac{1}{\text{nb of images}}$, added to a bitmap and then saved as a png and exr files. To expand the randomization to multiple images, I simply added and offset based on the id of the images to the method prepare() of the sampler. This offset modifies the seed for each block such that it is not the same as for the same block in the previous image.

The three images below show a piece of the final scene rendered with three different ids (0, 1, 2). As you can see the noise is different in each image. Those images are rendered with 2048 samples per pixel.
<div class="twentytwenty-container">
    <img src="final_scene0_small.png" alt="Final Scene id = 0">
    <img src="final_scene1_small.png" alt="Final Scene id = 1">
    <img src="final_scene2_small.png" alt="Final Scene id = 2">
</div>
 
The final scene is a merge of 8 1080p images at 2048 samples per pixel. As you can see below, the noise is drastically reduced. Each image has been rendered in approximately 45 minutes on a single machine of the cluster where I used up to 2 nodes at a time. The total render time was around 3 hours while it would have been around 6 hours on a single node. I chose to split further than 2*8192 such that in case of a crash or interruption, I would only loose up to $\frac{1}{8}$ of the total workload instead of a half.
<div class="twentytwenty-container">
    <img src="final_scene0_small2.png" alt="Final Scene id = 0">
    <img src="final_scene_small.png" alt="Final Scene">
</div>

Hair Rendering
==============

Fiber Shape
-----------

To render hair and fur I needed a fiber shape to intersect. Since I followed the implementation described in this (https://www.pbrt.org/hair.pdf) chapter of the pbrt-book, I first chose to use bezier curves as they do but since I had no tool to generate hair and fur with bezier curves, I chose to implement a support for fiber made of segments. Therefore, I integrated the segments primitive to the meshes and the acceleration structure. I did not try to mix triangles and segments in a single mesh, so I am not sure if it would behave fine but it would not for sure with a hair BSDF.

From there I had to implement intersection testing. I began by implementing the intersection testing with a cylinder around the segment capped by two spheres at each end points. The shadow frame and geo frame were calculated from the normal to the surface of the cylinder or the sphere respectively. I noticed later when I was implementing the hair BSDF that in the reference they were using flat ribbon and that it was probably best to keep my implementation as close as possible form their one to reduce the probability to introduce errors. Therefore, I changed the shadow frame to be built on the normal of the ribbon corresponding to the segment. More formally the shadow frame was built on a coordinate system that has x aligned with the segment direction, y perpendicular to the incoming ray and x and finally z perpendicular to x and y. To get even closer to their implementation I dropped the idea of the cylinder and the spheres to only intersect the ribbon.


Hair BSDF
---------

The hair BSDF is the part of this implementation which is heavily based on the code described in the pbrt-book. The hair BSDF is characterized by an index of refraction ($\eta$), an absorption coefficient ($\sigma_a$ that cover each RGB channel), a longitudinal roughness ($\beta_m$), an azimuthal roughness ($\beta_n$) and an angle for the scale deviation compared to the hair direction ($\alpha$). The few differences between this implementation and the implementation of pbrt are due to the difference between their bezier curves and the segmented curve I use or the fact that the interface of nori is different than the interface of pbrt. The most important difference is the output of the sampling method which in nori needs to be weighted by the PDF and the cos of wo. To improve the confidence I have that this code is correct I implemented two tests proposed in the pbrt-book. The first test is the whiteFurnaceTest and it ensures that for an incident radiance of one, a hair that have no absorption (i.e. $\sigma_a = 0$) reflects the full radiance. The second test is the samplingWeights and it ensures that for hairs that have no absorption the sample method always returns one. Both these tests converge and pass for this implementation. I did not use any library for those two tests and I implemented them in a new application called unittests.

In the pbrt book they try to render hairs with different $\beta_m$ while keeping the rest of the property constant. Here are three renders, all of them are rendered with $\eta = 1.55$, $\sigma_a = (0.42, 0.7, 1.37)$, $\beta_n = 0.6$ and $\alpha = 2.0$ but with a $\beta_m$ at, respectively, $0.1$, $0.25$, $0.6$.
<div class="twentytwenty-container">
    <img src="hair/cbox_hair0_1.8_0.1.png" alt="0.1">
    <img src="hair/cbox_hair0_1.8_0.25.png" alt="0.25">
    <img src="hair/cbox_hair0_1.8_0.6(bn0.6)).png" alt="0.6">
</div>
The book mention that for $\beta_m = 0.1$ the hair are too shiny, for $\beta_m = 0.25$ the hair looks human and for $\beta_m = 0.6$ the hair are too diffuse, and it seems accurate with the result above.

They also played with $\sigma_a$, $\beta_n$ and $\alpha$ and so did I. Here are three renders produced with $\eta = 1.55$, $\beta_m = 0.125$, $\beta_n = 0.3$, $\alpha = 2.0$ and $\sigma_a$ with a value respectively at $(3.35, 5.58, 10.96)$, $(0.84, 1.39, 2.74)$ and $(0.06, 0.10, 0.20)$.
<div class="twentytwenty-container">
    <img src="hair/cbox_hair0_1.10_black.png" alt="(3.35, 5.58, 10.96)">
    <img src="hair/cbox_hair0_1.10_brown.png" alt="(0.84, 1.39, 2.74)">
    <img src="hair/cbox_hair8_1.10_blond.png" alt="(0.06, 0.10, 0.20)">
</div>
As expected the absorption changes the color of the hair. A high absorption yield darker hair while a sufficiently low absorption yield blond hair.

Here are three renders produced with $'\eta = 1.55'$, $'\sigma_a = (0.42, 0.7, 1.37)'$, $'\beta_m = 0.3'$, $\alpha = 2.0$ and $\beta_n$ with a value respectively at $0.3$, $0.6$ and $0.9$.
<div class="twentytwenty-container">
    <img src="hair/cbox_hair0_1.18_0.3.png" alt="0.3">
    <img src="hair/cbox_hair0_1.18_0.6.png" alt="0.6">
    <img src="hair/cbox_hair0_1.18_0.9.png" alt="0.9">
</div>
As $\beta_n$ grows, the hair are more and more enlightened because of the simulated scattering that "transmit" more and more light through the volume.

Finally, here are two renders produced with $\eta = 1.55$, $\sigma_a = (0.42, 0.7, 1.37)$, $\beta_m = 0.3$, $\beta_n = 0.6$ and $\alpha$ with a value respectively at $0.0$ and $2.0$.
<div class="twentytwenty-container">
    <img src="hair/cbox_hair0_1.19_0.png" alt="0.0">
    <img src="hair/cbox_hair0_1.18_0.6.png" alt="2.0">
</div>
There are some subtle differences, but I do not see the secondary highlight they describe in the book. It could be caused by the fact that my mesh of hair is not fit to clearly distinguish this subtle change and it is likely that the lighting of the scene does not help either.

Credit
======
The final scene comes from blendswap. The cat, the bird and the room are from distinct scenes and authors.

The cat has been downloaded from http://www.blendswap.com/blends/view/57118. The original name of the scene is "Lowpoly Siamese Cat" by Gwinna. The modification added for this project where a change in the posture of the cat and the addition of fur.

The bird has been downloaded from http://www.blendswap.com/blends/view/67044. The original name of the scene is "Woodpecker" by hypnosis. The modification added for this project where a change in the posture of the bird and some textures have been modified.

The bedroom has been downloaded from https://www.blendswap.com/blends/view/85400 . The original name of the scene is "BEDROOM" by Rendars. The modification added for this project where the cat and the bird. Also, some object have been removed from the original scene.

The hair used to try the hair BSDF have been extracted from the scene "Katie - Version 2.01" by RikSavage and has been downloaded from https://www.blendswap.com/blends/view/69006.
Feedback
========

This project is massively time-consuming, but I guess you already know that. This is still interesting and motivating to work on a renderer. The access to the cluster until the project submission deadline would have been very useful.


<!--- Markdeep & image comparison library - probably no need to change anything below -->
<style class="fallback">body{visibility:hidden;white-space:pre;font-family:monospace}</style><script src="../resources/markdeep.min.js"></script><script>window.alreadyProcessedMarkdeep||(document.body.style.visibility="visible")</script>
<script src="https://ajax.googleapis.com/ajax/libs/jquery/1.11.0/jquery.min.js"></script>
<script src="../resources/jquery.event.move.js"></script>
<script src="../resources/jquery.twentytwenty.js"></script>
<link href="../resources/offcanvas.css" rel="stylesheet">
<link href="../resources/twentytwenty.css" rel="stylesheet" type="text/css" />
<script>
$(window).load(function(){$(".twentytwenty-container").twentytwenty({default_offset_pct: 0.5});});
</script>
