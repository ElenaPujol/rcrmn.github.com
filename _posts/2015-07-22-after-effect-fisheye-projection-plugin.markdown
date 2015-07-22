---
layout: post
title:  "Fisheye to Equirectangular Projection Plugin for After Effects"
date:   2015-07-22 18:25:00
categories: jekyll update
---


I've decided to make this post since it was a real pain in the ass to find any documentation on how to make
a plugin for After Effects/Premiere Pro in the web that wasn't 10 years old at the least.

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
So, we have a video that looks like a circle. We know it unfolds on a semi-sphere and covers it completely.
We have to transform it so that we have a square that also unfolds and covers a semi-sphere.
TODO: Add images


C'ya!
