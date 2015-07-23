---
layout: post
title:  "Fisheye to Equirectangular Projection Plugin for After Effects"
date:   2015-07-22 18:25:00
categories: jekyll update
---


I've decided to make this post since it was a real pain in the ass to find any documentation on how to make
a plugin for After Effects/Premiere Pro in the web that wasn't 10 years old at the least.

## Background
There are multiple ways to record 3D videos. In this post we will be speaking of videos recorded for 3D played in an
immersive environment using a frontal dome.

This kind of videos are recorded using a two-camera rig with fisheye with a field of view of 180º. These videos look
something like this:

![](/images/2015-07-22/fisheye-circle-1.jpg){: style="max-width:300px"}

This image can be presented with an equirectangular projection, which would look like this:

![](/images/2015-07-22/fisheye-transformed-1.jpg){: style="max-width:300px"}

## The problem
We have to make some 3D videos that have been recorde using two GoPro with a 180 fisheye lens each one. The problem
is that the vides need to be in equirectangular projection, since the software it has to be played on doesn't support
playback of fisheye projection.

Until now, the video team in our company has been doing it with a very manual system. Their 3D video-editing software
doesn't take much into consideration the fisheye 180º videos, so they had to warp and position the video in a way that
was very very manual and didn't produce optimal results.

On Monday it was decided that I would try and make it easier for them by making a plug-in for After Effects to
automate the change of projections.

## The quest
After looking for a while, I got to some website where they proposed various ways of transforming the frames from a video:
- One was to use Quartz Composer, an outdated tool from OsX. I decided it would not be a good solution since it would be 
difficult even to find the latest release.
- Create a filter using Pixel Bender by Adobe and use it as a Plug-In in After Effects.
- Create a full C/C++ plug-in for After Effects that did the transformation.

The simplest way to do what we needed seemed to be by using Pixel Bender. It wasn't until the filter was already created
and working that I discovered that since Adobe CS6, Pixel Bender filters were no longer supported as plug-ins for After
Effects.

So, having the filter already created and the maths of it working correctly, I went along and started development of a
full C++ After Effects plug-in. It took me a whole day to have it working right.

## The first enemy: projection maths
The filter we are going to create is going to use the standard way of applying transformations to an image:
For each pixel of the output, it's going to find the untransformed position in the input image and take
the color corresponding to that position. We are going to use subpixel sampling to avoid tessellation.

To get the untransformed position from the input, we have to consider the two coordinate systems we are working on.

To apply the equirectangular projected image over the hemisphere we use the following set of coordinates:

![Equirectangular coordinates](/images/2015-07-22/semi-sphere-equirect-coords.png){: style="max-width:300px"}

Where the angle θ will be used as x coordinates of the image and the angle φ will be treated as y coordinates.

![Equirectangular projection](/images/2015-07-22/equirectangular-projection.png){: style="max-width:300px"}

To apply the equidistant fisheye projected image over the hemisphere we use the following set of coordinates.

![Equidistant fisheye coordinates](/images/2015-07-22/semi-sphere-equidist-coords.png){: style="max-width:300px"}

Where the angle θ' will be used as the angle in the circular image, and the radius r will be used as the radius
in the circular image, as follows:

![Equirectangular projection](/images/2015-07-22/equidistant-projection.png){: style="max-width:300px"}

So, to get the x',y' position for the circular image we will have to first pass the coordinates x,y from the rectangular
output image to spherical coordinates using the first coordinate system, then those to the second shown spherical
coordinate system, then those to the polar projection and then pass the polar system to cardinal x',y'.

The formulas for each step will be:

![](/images/2015-07-22/math-1.png)

![](/images/2015-07-22/math-1.png)

![](/images/2015-07-22/math-3.png)

![](/images/2015-07-22/math-4.png)

![](/images/2015-07-22/math-5.png)

Once we have the unprojected coordinates, we apply a subpixel sampling filter to the original image at that point,
and the color we obtain is the color used for the transformed image.

## The second enemy: After Effects SDK






C'ya!
