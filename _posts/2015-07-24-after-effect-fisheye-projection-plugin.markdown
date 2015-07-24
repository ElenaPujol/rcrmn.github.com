---
layout: post
title:  "Fisheye to Equirectangular Projection Plugin for After Effects"
date:   2015-07-24 13:00:00
categories: jekyll update
---

There are multiple ways to record 3D videos. In this post we will be talking of videos recorded for 3D played in an
immersive environment using a frontal dome.

This kind of videos are recorded using a two-camera rig with fisheye with a field of view of 180º. These videos look
something like this:

![](/images/2015-07-24/fisheye-circle-1.jpg){: style="max-width:300px"}

This image can be presented with an equirectangular projection, which would look like this:

![](/images/2015-07-24/fisheye-transformed-1.jpg){: style="max-width:300px"}

<!--more-->

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

![Equirectangular coordinates](/images/2015-07-24/semi-sphere-equirect-coords.png){: style="max-width:300px"}

Where the angle θ will be used as x coordinates of the image and the angle φ will be treated as y coordinates.

![Equirectangular projection](/images/2015-07-24/equirectangular-projection.png){: style="max-width:300px"}

To apply the equidistant fisheye projected image over the hemisphere we use the following set of coordinates.

![Equidistant fisheye coordinates](/images/2015-07-24/semi-sphere-equidist-coords.png){: style="max-width:300px"}

Where the angle θ' will be used as the angle in the circular image, and the radius r will be used as the radius
in the circular image, as follows:

![Equirectangular projection](/images/2015-07-24/equidistant-projection.png){: style="max-width:300px"}

So, to get the x',y' position for the circular image we will have to first pass the coordinates x,y from the rectangular
output image to spherical coordinates using the first coordinate system, then those to the second shown spherical
coordinate system, then those to the polar projection and then pass the polar system to cardinal x',y'.

The formulas for each step will be:

![](/images/2015-07-24/math-1.png)

![](/images/2015-07-24/math-2.png)

![](/images/2015-07-24/math-3.png)

![](/images/2015-07-24/math-4.png)

![](/images/2015-07-24/math-5.png)

Once we have the unprojected coordinates, we apply a subpixel sampling filter to the original image at that point,
and the color we obtain is the color used for the transformed image.

I used [Adobe Pixel Bender](http://www.adobe.com/devnet/archive/pixelbender.html) to test these transformations and
tweak any sign changes. The code for the function is as follows.

{% highlight cpp %}
void evaluatePixel()
{
	float2 o = outCoord() / srcSize;
	float theta = (1.0-o.x) * M_PI;
	float phi = o.y * M_PI;
	float3 sphericCoords = float3(cos(theta)*sin(phi), sin(theta)*sin(phi), cos(phi));
	
	float phi2 = acos(sphericCoords.y)/(M_PI);
	float theta2 = atan(-sphericCoords.z, sphericCoords.x);
	
	float2 inCentered = float2(phi2*cos(theta2), phi2*sin(theta2));
	
	float2 i = inCentered + float2(0.5, 0.5);
	
	dst = sample(src,i*srcSize);
}
{% endhighlight %}


## The second enemy: After Effects SDK

Once I had the math working properly it was time to tackle at the After Effects SDK. It is quite a complex system but
it comes with some example code and with that and a bit of trial and error I managed to put together a working
implementation of the filter.

The first implementation of the plugin was the straightforward implementation which for every frame and for every
pixel calculated the pixel position of the input. That, of course, is very slow, and in a later implementation a
cache for the input positions was created to speed things up.

An AE plugin written in C/C++ is a typical library that exposes a function to After Effects, which receives a command
and the various parameters needed:

{% highlight cpp %}
DllExport
PF_Err EntryPointFunc ( PF_Cmd cmd,
                        PF_InData *in_data,
                        PF_OutData *out_data,
                        PF_ParamDef *params[],
                        PF_LayerDef *output,
                        void *extra );
{% endhighlight %}

The entry point function is expected to always respond to some commands, and some other may respond to or not, optionally.
The required commands are `PF_Cmd_ABOUT`, `PF_Cmd_GLOBAL_SETUP` and `PF_Cmd_PARAMS_SETUP`. These are expected to
give AE the basic information to make the plugin work or to set any required data for the whole plugin lifespan.
Their implementation was taken from the code of one of the provided examples and modified so that the plugin
would have no parameters, since we aren't using any.

The initial implementation only responded to one more command, which was `PF_Cmd_RENDER`, which is called for every
frame to be rendered. In it, an iterator function provided by the SDK is used to traverse every pixel to be rendered
and the transformation is applied to get its final color.

The info provided by the iterator function to the iterated function consists only of the position of the pixel to be
rendered and the color of that pixel in the source image, and an optional pointer to a user-provided structure, which
we have to fill with the original image so we have access to the pixel at the transformed position.

On the second version of the plugin a matrix was created prior to render a sequence of frames for those frame's
dimensions to hold the transformed positions of each pixel of a frame, so that it was computed only once per
per sequence instead of for each frame.

This was done by responding to `PF_Cmd_SEQUENCE_SETUP`, `PF_Cmd_SEQUENCE_RESETUP`, `PF_Cmd_SEQUENCE_SETDOWN`,
and `PF_Cmd_FRAME_SETUP`. The later was implemented but it might be it is not necessary for what we are doing,
but since I'm not fully aware of the actual workflow of AE, I'm not going to risk it.

In `PF_Cmd_SEQUENCE_SETUP` an array of positions is created of size equal to the number of pixels a frame in the
sequence will have and fill it with the transformed positions each position corresponds to. We don't have
access to an actual frame, so according to the documentation, the width and height of the sequence might
not correspond to the width and height of an actual frame. That's why we also respond to the command
`PF_Cmd_FRAME_SETUP`, there we check if width and height coincide, and if they don't, we recreate the array.

With that, for the command `PF_Cmd_RENDER` we only have to take the precomputed values and use them as positions
in the input image.

In `PF_Cmd_SEQUENCE_SETDOWN` the array of positions is disposed of. In `PF_Cmd_SEQUENCE_RESETUP` we simply call
the implementations of SequenceSetdown and then of SequenceSetup.


## Wrapping up

Once the plugin is created, we can use it on a video. The process to apply it is rather tedious, since the plugin
has to receive a square image with the equidistant projection centered in it and covering it fully. To add
to this problem, there is also the fact that the videos for each of two eyes has to be synchronized between
them and that we have to make sure the images of the video for each eye are well aligned.

After some testing, it was seen that, since the fisheye lens actually has a FOV of 185º, the results were
a bit better if the images were stretched a bit outside the square, as can be seen in the next images:

![](/images/2015-07-24/fisheye-circle-2.jpg){: style="max-width:300px"}

![](/images/2015-07-24/fisheye-transformed-2.jpg){: style="max-width:300px"}

I'm sorry that this post is not more tutorialish, I can't publicly share the work done here, but it might help
someone anyway, at least in the maths. Thanks for reading to anyone who stayed this long.

C'ya!
